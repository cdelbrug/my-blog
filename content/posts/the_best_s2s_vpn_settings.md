---
title: "The Best Site to Site VPN Settings"
date: 2026-01-07T19:19:00-05:00
author: "Caleb Delbrugge"
---

# Introduction

When configuring a Site to Site VPN, you may be wondering what the best settings are such as:  
- IKEv1 or v2
- Encryption
- Integrity
- Lifetime
- PFS

It's a good question, as I've tried to figure that out myself. I stumbled upon this recommendation from NIST called "Suite-B" - and it seems to answer the question. There's an RFC for it below.

[RFC 6379 - Suite B Cryptographic Suites for IPsec](https://datatracker.ietf.org/doc/html/rfc6379)  

There's two options they provide for encryption
- Suite-B-GCM-128
- Suite-B-GCM-256

## Phase 1

- IKEv2
- Encryption type: AES-CBC-128 or AES-CBC-256
- Hash type: SHA256 or SHA384
- Diffie Hellman group: 19 or 20
- Lifetime: 28800 seconds
- Data Lifetime: Unlimited
- Authentication: PSK or Certificate

Other settings in phase 1 include whether to use a pre-shared key or certificate for authentication.
There's also the lifetime, which is how often to recycle the keys. For this, I refer to Palo Alto firewalls default setting, which is 8 hours.

## Phase 2

- Encryption type: AES-GCM-128 or AES-GCM-256
- Hash: Null (built-in to GCM)
- Diffie Hellman group: 19 or 20
- Lifetime: 3600 seconds
- Data Lifetime: Unlimited
- Authentication: PSK or Certificate

Again, using 1 hour lifetime to match the default on the Palo.

# Configuration

In my examples, I'll be using Suite-B-GCM-128
## Palo Alto
I don't have CLI for this yet but here's steps for the GUI

### Phase 1
```
Network > Network Profiles > IKE Gateways

Name: <PEER-NAME>-IKE-GW
Version: IKEv2 Only
Interface: <OUTSIDE-INTERFACE>
Peer IP Type: IP
Peer IP: <PEER-IP>
PSK: <PSK>
Peer Identification IP Address: <PEER-IP>

Advanced Options:
- IKE Crypto Profile: Suite-B-GCM-128
- Liveness Check: 5 seconds

Network > Interfaces > Tunnel > Add
Zone > New > <PEER-NAME>
IPv4 > Add > <PEERING-IP>
```

### Phase 2

```
Network > Network Profiles > IPSec Tunnels > Add

Name: <PEER-NAME>-IPSEC-TUNNEL
Interface: <TUNNEL-IF>
IKE Gateway: <PEER-NAME>-IKE-GW
IPSec Crypto Profile: Suite-B-GCM-128

```
## ASA

If you're still running ASA code, that's awesome. It's pretty good at VPNs, and is still supported on Firepower hardware as far as I know.

### Phase 1
```
crypto ikev2 policy 100
 encryption aes
 integrity sha256
 group 19
 lifetime seconds 28800
```
You'll want to disable idle timeout on the ASA for VPNs. Otherwise, if no data has passed for a while, it will tear down the tunnel. Sometimes, the remote end isn't even aware of it.
```
group-policy <NAME> internal
group-policy <NAME> attributes
 vpn-idle-timeout none

tunnel-group <PEER-IP> type ipsec-l2l
tunnel-group <PEER-IP> ipsec-attributes
 ikev2 local-authentication pre-shared-key <PSK>
 ikev2 remote-authentication pre-shared-key <PSK>
tunnel-group <PEER-IP> general-attributes
 default-group-policy <NAME>
```
### Phase 2

Setting the "kilobytes unlimited" will disable Data Lifetime, which is a feature to recycle keys after a certain amount of data is sent. It's best to leave it off due to some firewalls not supporting it or having it off by default. If the firewalls in the tunnel mismatch on this, it can cause unexpected tunnel stability if one is not aware of the other doing this.

```
crypto ipsec ikev2 ipsec-proposal SUITE-B-GCM-128-PROPOSAL
 protocol esp encryption aes-gcm
 protocol esp integrity null

crypto ipsec profile SUITE-B-GCM-128-PROFILE
  set ikev2 ipsec-proposal SUITE-B-GCM-128-PROPOSAL
  set security-association lifetime seconds 28800 kilobytes unlimited
  set pfs group19
```
### Virtual Tunnel Interface
```
interface tunnel <1-100>
  nameif <IF-NAME>
  ip address <PEERING-IP> <MASK>
  tunnel source interface <OUTSIDE-INTERFACE-NAME>
  tunnel destination <PEER-IP>
  tunnel mode ipsec ipv4
  tunnel protection ipsec profile SUITE-B-GCM-128-PROFILE
```
### Routing

Now you need to decide if you will perform static or dynamic routing. If you are going to have any kind of redundancy, dynamic is recommended.

#### Dynamic

BGP is the standard for dynamic routing over VPN. The BGP peer will be the peer's tunnel interface IP.

#### Static

For static routes, you can just point the route to the peer's tunnel interface IP.

### NAT

#### Outbound

NATs depend on what direction the traffic is going. Who is initiating the connection? If you only need to initiate connectivity to the peer's network, then you can just use an overload NAT. You'll want to use a public IP, one that you own. The quickest is to just use the IP address of your firewall's public interface. Using a public IP also prevents overlaps from happening on the remote side. If you have multiple VPNs in other datacenters for redundancy, then this also works out by preventing overlapping routes on the peer's end back to you.

#### Inbound

If the peer needs to connect to services on your end, then there's two options:
1. Use a one-to-one IP NAT.
2. Use PAT (Port Address Translation).

If you have limited public IPs to use for one-to-one NATs, then PAT is a good choice. You specify multiple ports on the same IP address to translate to different IPs and ports on the backend.