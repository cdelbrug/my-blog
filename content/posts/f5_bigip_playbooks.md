---
title: "Basic F5 BIG-IP Playbooks"
date: 2026-08-04T22:02:00-05:00
author: "Caleb Delbrugge"
description: "Provisioning F5 BIG-IP VIPs, GTM Wide IPs, and Let's Encrypt certs with Ansible"
tags: ["ansible", "f5", "big-ip", "networking", "automation"]
---

Back in the day, I wanted a way to provision new sites on the F5 as fast as possible. That meant combining VIPs and forwarding to pools based on Host header. This can be done with an iRule, but LTM policies are more effecient. I've created basic playbooks to create new VIPs, and then another playbook to add sites to them later, including GTM if you have that as well. Most recently I added a third playbook that requests and installs a Let's Encrypt certificate via Cloudflare DNS-01, so the cert side of standing up a VIP is one command too.

[`bigip-playbooks`](https://github.com/cdelbrug/bigip-playbooks)

## What it does

There are three entry-point playbooks now, driven by the [`f5networks.f5_modules`](https://docs.ansible.com/projects/ansible/latest/collections/f5networks/f5_modules/index.html), [`community.crypto`](https://docs.ansible.com/ansible/latest/collections/community/crypto/index.html), and [`community.general`](https://docs.ansible.com/ansible/latest/collections/community/general/index.html) collections:

- **`playbooks/new-vip.yml`** — stands up a brand-new 80/443 virtual server pair: HTTP monitor, pool, LTM policies (draft → publish) for both ports, an HTTP-to-HTTPS redirect rule on port 80, a host-header forward rule on port 443, and the two virtual servers themselves.
- **`playbooks/new-site.yml`** — adds one or more new sites to an *existing* VIP's LTM policy, and wires up the matching GTM configuration (monitor, pool, wide IP) so the site is reachable through global DNS load balancing too.
- **`playbooks/new-certificate.yml`** — requests a Let's Encrypt certificate using a Cloudflare DNS-01 challenge, then imports the cert/key/chain onto BIG-IP and creates a client-ssl profile

The playbooks work off the same shape of data: a list of sites (or a single cert request), each describing a domain, a health check, pool members, and the naming pieces (VIP prefix, external interface, wide IP, cert label) needed to assemble consistent object names on the BIG-IP.

## The building blocks

All the actual BIG-IP work lives in one reusable role, `roles/f5`, as a set of small, single-purpose task files under `roles/f5/tasks/`:

| Task file | Module | Purpose |
|---|---|---|
| `ltm-monitor-http` / `ltm-monitor-https` | `bigip_monitor_http(s)` | HTTP(S) health check with a Host header and expected response |
| `ltm-pool` | `bigip_pool`, `bigip_pool_member` | Pool creation and member registration |
| `ltm-policy-draft` / `ltm-policy-publish` | `bigip_policy` | Draft-then-publish LTM policy lifecycle |
| `ltm-policy-forward-pool-rule` | `bigip_policy_rule` | Host-header match → forward to pool |
| `ltm-policy-redirect-rule` | `bigip_policy_rule` | Plain HTTP → HTTPS redirect |
| `ltm-vip-80` / `ltm-vip-443` | `bigip_virtual_server` | The virtual servers themselves, with TCP/HTTP profiles, OneConnect, SSL, and persistence attached on 443 |
| `gtm-monitor` / `gtm-pool` / `gtm-wideip` | `bigip_gtm_*` | GTM monitor, A-pool with NY4/CH1 ordered by which site is primary, and the wide IP tying it to the pool |
| `ssl-cert-import` / `ssl-key-import` | `bigip_ssl_certificate`, `bigip_ssl_key` | Import a Let's Encrypt cert, chain, and key onto BIG-IP |
| `ssl-client-ssl-profile` | `bigip_profile_client_ssl` | Build a client-ssl profile from the imported cert/key/chain |

Two new roles handle the certificate side: `roles/letsencrypt` generates the key/CSR and drives the ACME v2 order, and `roles/cloudflare_dns` creates/removes the `_acme-challenge` TXT record used to prove domain ownership.

The playbooks call into all of this with `include_role` + `tasks_from`, looping over the site list.

I made a standardized naming convention (`{{ domain }}-{{ protocol }}-MONITOR`, `{{ domain }}-{{ server_port }}-POOL`, `{{ vip_prefix }}-443-POLICY`, and so on) so the vars fill in all the names for you. The client-ssl profile follows the same convention (`{{ cert_label | upper }}-CLIENT-SSL-PROFILE`), so as long as `new-vip.yml`'s `client_ssl_profile` var matches what you passed as the cert's common name in `new-certificate.yml`, a cert issued by one playbook drops straight into a VIP built by another — there's no shared variable behind it, just the same name typed in both places.

## GTM: primary and secondary sites

The GTM pool logic is worth calling out. Instead of a single generic pool task, there are two nearly-identical `bigip_gtm_pool` tasks — one ordering members NY4-first, one CH1-first — gated by `when: item['primary_servers'] == 'ny4'` / `'ch1'`. That lets each site declare which data center should be preferred for `global-availability` load balancing, while keeping both data centers in the pool as a fallback.

## Certificates: DNS-01 through Cloudflare

`roles/letsencrypt` handles the whole ACME flow: generate a key and CSR for `cert_common_name` (wildcards like `*.routeyour.net` work fine), open an ACME v2 order asking for a `dns-01` challenge, hand the pending challenge to `roles/cloudflare_dns` to create the `_acme-challenge` TXT record, poll with `dig` until it's actually propagated, validate and pull down the cert, then clean the TXT record back up. Nothing sits around in the DNS zone afterward.

Two things worth calling out:

- `*` isn't valid in file paths or BIG-IP object names, so everything on disk and on the BIG-IP uses `cert_label` (the common name with `*` swapped for `wildcard`), while the actual ACME order and CSR still use the real common name.
- The chain cert gets named after its own issuing CA rather than the domain — the role reads the intermediate's subject CN (e.g. `R11`) and stores/imports it under that slug. Let's Encrypt reuses the same intermediate across every cert it issues around the same time, so this way it only gets imported once and every domain's client-ssl profile just points at it.

### Staging VS Production
It defaults to Let's Encrypt's **staging** environment for testing. Delete the local cert directory and F5 certs/keys/profiles and switch `letsencrypt_environment` to `production` once staging works for you.

## Running it

Copy `.env.sh.example` to `.env.sh`, fill in `F5_USER` / `F5_PASSWORD` (and `CLOUDFLARE_API_TOKEN` for the cert playbook), and source it before running anything:

```bash
source .env.sh
ansible-playbook playbooks/new-vip.yml
```

`-e` still works if you'd rather type credentials explicitly instead of exporting them:

```bash
ansible-playbook playbooks/new-vip.yml -e username=myuser -e password=mypassword
```

Everything else — the domain, pool members, VIP address, cert common name, Cloudflare zone — is defined inline in each playbook's `vars` block, so a change is: edit the block, run the playbook, review the config on the BIG-IP (and, for certs, the DNS zone and the `certs/` directory).

The full source is up on GitHub [bigip-playbooks](https://github.com/cdelbrug/bigip-playbooks) if you want to adapt the roles for your own needs.