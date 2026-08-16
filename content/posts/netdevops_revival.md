---
title: "NetDevOps Lab - Reviving a 2021 lab in 2026"
date: 2026-08-16T16:00:00-05:00
author: "Caleb Delbrugge"
description: "Reviving a IaC demo from 2021"
tags: ["ansible", "networking", "automation", "batfish"]
---

# Reviving a 2021 NetDevOps Lab in 2026

I recently found an old bookmark to a good intro to infra as code for netdevops. It was `Vista-Technology/netops-quickstart`, an older but ambitious lab built around Arista cEOS, GitLab, Ansible, Batfish, Prometheus, Grafana, Loki, and Docker.

But to get it running presently, it became a tour through what happens when a good infrastructure demo ages for a few years.

## The Goal

My goal was to run the lab locally in VMware Workstation. The host was more than capable with enough CPU and memory to comfortably run a serious nested lab. I built an Ubuntu Server 20.04 VM - the version from the year 20-21, installed Docker, cloned the repository, downloaded the Arista cEOS image, and started working through the instructions.

The original lab promised a full NetOps flow:

1. Build a small Arista spine/leaf topology.
2. Start GitLab and a local runner.
3. Validate intended configurations with Batfish.
4. Deploy configurations with Ansible.
5. Test host-to-host reachability.
6. Watch the results in Prometheus, Grafana, Loki, and Consul.

That is a great learning architecture. It tells the full story instead of only showing one tool.

## The First Problem: `latest` Is Not a Version

The first major theme was Docker image drift.

The original Compose file used several unpinned or outdated images:

```yaml
gitlab/gitlab-ce:latest
gitlab/gitlab-runner:latest
grafana/grafana:latest
grafana/loki:latest
grafana/promtail:latest
batfish/allinone
docker.io/bitnami/consul:1-debian-10
```

That worked when the repo was new. Years later, it meant the lab was pulling modern images into an old dependency stack.

The GitLab Runner image was the first to fail. The runner was now based on a modern Ubuntu/Python environment, and the Dockerfile tried to install old Python packages system-wide with `pip`. Python rejected that with a PEP 668 externally managed environment error.

I worked around that at first, but the next failure made the real issue obvious: old packages like `PyYAML==5.4.1` did not build cleanly on Python 3.12. Although the Ubuntu VM used Python 3.8, the unpinned `gitlab/gitlab-runner:latest` image pulled a much newer Python environment. That newer container environment could not cleanly install the lab's old Python dependency stack, including `PyYAML==5.4.1`.

The right fix was not to keep patching `pip`. The right fix was to stop using `latest`.

I pinned GitLab Runner to a 2021-era image:

```yaml
gitlab/gitlab-runner:ubuntu-v13.12.0
```

I also pinned GitLab CE:

```yaml
gitlab/gitlab-ce:13.12.15-ce.0
```

That brought the GitLab stack back into the time period the repo expected.

## Consul Had Disappeared

The next failure was Consul:

```text
manifest for bitnami/consul:1-debian-10 not found
```

The image tag the repo referenced no longer existed. I replaced it with HashiCorp's official Consul image:

```yaml
hashicorp/consul:1.9.5
```

This was another reminder that old labs often fail not because the design is bad, but because the external ecosystem keeps moving.

## Grafana and Loki Needed to Go Back in Time

After Consul, Grafana and Loki failed. The repo's Loki configuration was written for an older Loki version, but `grafana/loki:latest` expected newer configuration syntax.

The fix was to pin the observability stack:

```yaml
grafana/grafana:7.5.7
grafana/loki:2.2.1
grafana/promtail:2.2.1
```

After that, the monitoring containers came up cleanly.

I also discovered that the repo provisions Grafana datasources, but the polished dashboard shown in the README was not actually included as dashboard JSON in the cloned repo. The datasources existed, but the dashboard asset itself appeared to be missing. That was a useful distinction: Grafana was working, but the dashboard from the screenshot was not something the repo fully shipped.

## The Arista Exporter Broke on Modern Python

The Arista eAPI exporter had its own version drift problem. It used:

```dockerfile
FROM python:3-slim
```

By 2026, that meant a much newer Python than the code expected.

The exporter also had unpinned Python dependencies. It failed because `yamlconfig` called `yaml.load()` in a way newer PyYAML no longer allowed:

```text
TypeError: load() missing 1 required positional argument: 'Loader'
```

I fixed that by changing the exporter image to Python 3.8 and pinning compatible dependencies:

```text
requests==2.25.1
prometheus-client==0.9.0
falcon==2.0.0
yamlconfig==0.3.1
PyYAML==5.4.1
argparse
pyeapi==0.8.4
```

After rebuilding the exporter, Prometheus began scraping the Arista devices successfully.

That was a big milestone. The telemetry path was alive.

## Ansible Galaxy Pulled the Future Into the Past

The GitLab pipeline got farther, but then the Batfish validation job failed.

The repo used Ansible 2.10.6, but the unpinned Galaxy requirements pulled modern collections:

```text
arista.eos 12.2.0
ansible.netcommon 8.6.2
ansible.utils 6.1.0
```

Those collections did not support Ansible 2.10. The first visible error was a missing Jinja filter:

```text
no filter named 'ipaddr'
```

The template used:

```jinja2
{% set peer_ip = neighbor.ipv4 | ipaddr('address') %}
```

For this lab, the values were simple enough that I replaced it with:

```jinja2
{% set peer_ip = neighbor.ipv4.split('/')[0] %}
```

Then I pinned compatible Ansible collections:

```yaml
---
collections:
  - name: lvrfrc87.git_acp
    version: 2.2.0
  - name: arista.eos
    version: 1.3.0
  - name: ansible.netcommon
    version: 1.5.0

roles:
  - name: batfish.base
```

The validation job moved forward.

## Batfish Also Needed a Matching Version

The next Batfish error was:

```text
HTTPConnectionPool(host='batfish', port=9997): Max retries exceeded
```

The pipeline was using:

```text
pybatfish==2021.4.12.882
```

but the container was using the modern `batfish/allinone` image. I pinned Batfish to the matching 2021 image:

```yaml
batfish/allinone:2021.04.12.882
```

That aligned the Python client and the service API.

## The Alpine Hosts Could Not Be Reached

The final stage of the pipeline tested host-to-host reachability. It failed over SSH:

```text
alpine@host-1: Permission denied (publickey)
alpine@host-2: Permission denied (publickey)
```

At first this looked like an SSH key mismatch. I compared the key used by the runner with the `authorized_keys` inside the Alpine containers, and they matched.

The real clue came from the SSH server logs inside the host containers:

```text
User alpine not allowed because account is locked
```

The Alpine image had changed behavior. The Dockerfile created the `alpine` user, but the account was locked. Modern OpenSSH refused the login before the key could help.

The fix was to unlock the lab user during image build:

```dockerfile
RUN adduser -u 1000 -G wheel -s /bin/sh -D alpine && \
    echo "alpine:alpine" | chpasswd && \
    echo "%wheel ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
```

For a disposable local lab, that was acceptable. For anything real, obviously not.

## One Last Ansible Problem: Remote Python

Once SSH worked, the ping test still failed:

```text
ModuleNotFoundError: No module named 'ansible.module_utils.six.moves'
```

The Ansible `shell` module expected a remote Python environment compatible with the Ansible module bundle. These were minimal Alpine containers, and I did not need full Ansible module execution just to run `ping`.

The fix was simple: use `raw`.

Original:

```yaml
- name: Ping destination
  shell: "ping -c 1 -w 2 {{ ip_to_ping }}"
  register: output
```

Fixed:

```yaml
- name: Ping destination
  raw: "ping -c 1 -w 2 {{ ip_to_ping }}"
  register: output
```

That bypassed remote Python and ran the command directly over SSH.

Finally, the pipeline passed.

## What I Learned

The biggest lesson was that the architecture of the lab was still valuable, but the implementation had aged.

The original idea is excellent:

```text
Git -> CI -> validation -> deployment -> test -> monitoring
```

That is exactly the NetDevOps story I wanted to see.

But old labs age in predictable ways:

- `latest` images drift.
- package managers change behavior.
- upstream image tags disappear.
- Ansible collections move ahead of old Ansible versions.
- network automation APIs change.
- Linux distro defaults change.
- SSH defaults change.
- dashboards and runtime assets get left out of repos.

## Would I Use This Lab Again?

Yes, but with a specific purpose.

I would use this lab to understand the classic all-in-one NetDevOps workflow. It is a good example of how the pieces fit together:

- GitLab
- GitLab Runner
- Ansible
- Batfish
- Arista cEOS
- Prometheus
- Grafana
- Loki
- Consul

But I would not use it as the foundation for a modern long-term lab.

For that, I would start with:

```text
Containerlab + netlab
```

Then add:

```text
Arista AVD for Arista automation
Nautobot or NetBox for source of truth
Batfish for validation
Prometheus/Grafana/SuzieQ for observability
GitHub Actions or GitLab CI for pipelines
```

That modular approach is easier to maintain than one large Compose-era demo stack.

## Final Thoughts

Getting this lab working was frustrating, but useful. Every failure exposed a real operational lesson:

- pin your dependencies,
- avoid relying on `latest`,
- make CI reproducible,
- document runtime-generated artifacts,
- do not assume old images will exist forever

In the end, the lab worked. More importantly, debugging it made the NetDevOps workflow feel less abstract. I did not just run a pipeline; I had to understand every layer of it.

That might have been the most valuable part of the whole exercise.
