---
title: "Basic F5 BIG-IP Playbooks"
date: 2026-08-04T21:53:00-06:00
author: "Caleb Delbrugge"
description: "Provisioning F5 BIG-IP VIPs and GTM Wide IPs with Ansible"
tags: ["ansible", "f5", "bigip", "networking", "automation"]
---

Back in the day, I wanted a way to provision new sites on the F5 as fast as possible. That meant combining VIPs and forwarding to pools based on Host header. This can be done with an iRule, but LTM policies are more effecient. I've created basic playbooks to create new VIPs, and then another playbook to add sites to them later, including GTM if you have that as well.

[`bigip-playbooks`](https://github.com/cdelbrug/bigip-playbooks)

## What it does

There are two entry-point playbooks, both driven by the [`f5networks.f5_modules`](https://docs.ansible.com/projects/ansible/latest/collections/f5networks/f5_modules/index.html) collection:

- **`playbooks/new-vip.yml`** — stands up a brand-new 80/443 virtual server pair: HTTP monitor, pool, LTM policies (draft → publish) for both ports, an HTTP-to-HTTPS redirect rule on port 80, a host-header forward rule on port 443, and the two virtual servers themselves.
- **`playbooks/new-site.yml`** — adds one or more new sites to an *existing* VIP's LTM policy, and wires up the matching GTM configuration (monitor, pool, wide IP) so the site is reachable through global DNS load balancing too.

Both playbooks work off the same shape of data: a list of sites, each describing a domain, a health check, pool members, and the naming pieces (VIP prefix, external interface, wide IP) needed to assemble consistent object names on the BIG-IP.

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

The playbooks call into these with `include_role` + `tasks_from`, looping over the site list.

One nice side effect of the shared naming convention (`{{ domain }}-{{ protocol }}-MONITOR`, `{{ domain }}-{{ server_port }}-POOL`, `{{ vip_prefix }}-443-POLICY`, and so on) is that `new-site.yml` and `new-vip.yml` can reference objects created by each other without any extra bookkeeping — the names are derived, not hand-tracked.

## GTM: primary and secondary sites

The GTM pool logic is worth calling out. Instead of a single generic pool task, there are two nearly-identical `bigip_gtm_pool` tasks — one ordering members NY4-first, one CH1-first — gated by `when: item['primary_servers'] == 'ny4'` / `'ch1'`. That lets each site declare which data center should be preferred for `global-availability` load balancing, while keeping both data centers in the pool as a fallback.

## Running it

Credentials never live in the repo. They're passed at runtime:

```bash
ansible-playbook playbooks/new-vip.yml -e username=myuser -e password=mypassword
```

or exported as `F5_USER` / `F5_PASSWORD` / `F5_VALIDATE_CERTS` if you uncomment the `environment:` block at the top of a playbook. Everything else — the domain, pool members, VIP address, SSL profile — is defined inline in each playbook's `vars.site_list` (or `vars.vip` for `new-vip.yml`), so a change is: edit the block, run the playbook, review the config on the BIG-IP.

The full source is up on GitHub [bigip-playbooks](https://github.com/cdelbrug/bigip-playbooks) if you want to adapt the roles for your own BIG-IP estate.