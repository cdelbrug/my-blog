---
title: "F5 301b Study Guide"
date: 2023-04-30T20:14:22+08:00
---

# Introduction

Hey all,

This is the study guide I've been building since I started studying for this exam way back in 2017. Hopefully it can help fill any gaps you may have. I've sourced the material from solution articles, experience, and manuals.
{{< line_break >}}
{{< line_break >}}

## Section 1 - Maintain Application and LTM Device Health
{{< line_break >}}
## Objective 1.01 Given a scenario, determine the appropriate profile setting modifications
{{< line_break >}}
### SSL

[K13385: Overview of the Proxy SSL feature](https://support.f5.com/csp/article/K13385)

Proxy SSL does NOT terminate SSL on the load balancer. Use it to optimize the SSL connection.

Use Proxy SSL in an SSL profile to forward a client cert to a server for certification authentication
{{< line_break >}}
### Stream Profile

[Overview of the Stream Profile](https://support.f5.com/csp/article/K39394712)

[Article: K7027 - Replacing multiple strings using a Stream profile](https://support.f5.com/csp/article/K7027)

“Content rewrite”

Standard Virtual Server setting

Replaces string in a data stream (TCP) from the client and server

Can replace string if it spans across multiple segments by buffering just enough data to do so

If HTTP profile is applied, search is done in HTTP payload only (not headers) in each segment. Otherwise, the whole TCP segment is searched and this is when you can replace HTTP Headers!

Compatible with HTTP Profile Chunking

#### Parameters

Source: What to look for in the content

Target: What to replace the content with

#### Replace Multiple Sets of Strings

Leave ‘source’ in profile blank

Use a unique delimiter in the ‘target’ field and a space between the source/target pairs.

Don’t put < > unless you want to literally.

**@**<search string1>**@**<replacement string1>**@** **@**<search string2>**@**<replacement string2>**@**

Note: The first character in the field defines the delimiter bounding each field for this replacement and must not appear anywhere else in the target string. In certain versions, the delimiter can be a period (.), asterisk (\*), forward slash (/), dash (-), colon (:), underscore (\_), question mark (?), equals (=), at (@), comma (,), or ampersand (&) character.

**Example:**

```
ltm profile stream h1_marquee {
 app-service none
 defaults-from stream
 source <h1>Sunshine</h1>
 target <marquee><h1>Moon</h1></marquee>
}
```

You don’t need to use an iRule

```
when HTTP_RESPONSE {
 STREAM::expression "@123@456@ @abc@xyz@"
 STREAM::enable
}
```

#### Recommendations

- Restrict rewrites to Content-Type: text\*/
- A stream profile must be on VIP before using the STREAM command in an iRule. The default Stream profile may be used with all parameters blank
- Data is changed once per operation
- A compression profile is required to rewrite compressed data
- Source and Target are case-sensitive unless you use RegEx

### HTTP

[Choosing appropriate profiles for HTTP traffic](https://support.f5.com/csp/article/K4707?sr=46615374)

[Overview of HTTP Chunking](https://support.f5.com/csp/article/K5379)

[K14775: Configuring an HTTP profile to rewrite URLs so that redirects from an HTTP server specify the HTTPS protocol (10.x, 11.x, and 12.x)](https://support.f5.com/csp/article/K14775)

**Drawbacks** - Uses CPU, more memory utilized for compression/caching

#### Options

- Request Header Insert

- Request Header Erase

- Insert X-Forwarded-For

- Accept XFF - Distrust or trust XFF header from client to use in statistics for AVR

- Fallback host, Fallback on error codes

- Redirect Rewrite

- Server Header Name

- Cookie encryption

- Protocol enforcement - Header size, count, Pipelining

- Strict Transport Security (HSTS)

  - Protect against stripping attacks by downgrading to HTTP. Force all content in a page to use HTTPS. The F5 inserts a header in the response to the client.

#### Redirect Rewrite Options

The F5 will see an HTTP redirect from a server on a 443 VIP and change it to HTTPS.

Great for SSL offload unaware servers that need to send some redirects.

If the F5 does not do this, one of the following will happen:

- If there’s no HTTP VIP, causes request failure
- Request goes to HTTP VIP and HTTP pool member and causes failure
- Request goes to HTTP VIP and server, which gets redirected to HTTPs and causes redirect loop

Matching

Only rewrite courtesy redirects to HTTPS

Courtesy redirects = Sent by server, the F5 adds / to specify a directory if original resource isn’t found.

- http://f5.com/stuff → https://f5.com/stuff/

Nodes - Change the redirect containing the node’s IP to the VIP

All - Rewrites all HTTP 301, 302, 303, 305, or 307 redirects to HTTPS

### HTTP Compression

Profiles > Services > HTTP Compression

Good for WAN clients with slow internet and/or high latency

Reads **Accept-encoding** from client, removes and sends request to server  
Inserts **Content-Encoding** header in response from Server, and specifies clients preferred method, **gzip** or **deflate**

**Compression Levels gzip**

1 - 9, highest is more compression. Anything more than 1 will degrade performance in some way

Depending on load, gzip compression level will automatically go down.

### Web Acceleration (Cache)

Profiles > Services > Web Acceleration

- Must have HTTP profile assigned to VIP
- The server only has to serve content to the BIG-IP once every expiration period for the content
- The cache size in a profile is shared among all VIPs it is applied to.

Types of web acceleration profiles: Basic, Optimized, optimized caching

When to use caching

- High demand objects - server sends data to BIG IP when content expires
- Static content - CSS files, javascript, images/logos, audio.

Items that are cacheable - RFC2616 HTTP/1.1

- 200, 203, 206, 300, 301, 410 responses
- GET methods by default - other methods are supported, even non HTTP methods specified in URI list or iRule
- Based on User-Agent and Accept-Encoding headers
- If the server says it's cacheable, the F5 does it.

Cache-Control Header specifies which content cannot be cacheable

- private, no-store, no-cache - Do not cache
- Set-Cookie - Not cached, as it is usually authentication intended for a particular user session

Cannot cache - HEAD, PUT, DELETE, TRACE, CONNECT by default

Objects that are cached continue to be served even if the pool member is offline, until the object expires. This is by design. Create an iRule if you want this changed.

#### Show and Delete Entries

**show ltm profile ramcache** <profile_name> **\[ host** <vip_address:port> **max-response** <how_many_entries_to_display> **]**

**delete ltm profile ramcache** <profile_name> **\[ host** <vip_address:port> **uri /**<object_name> **]**

**delete ltm profile ramcache** <profile_name> \*\*\*\*- Deletes all entries

### XML

Route to pools, pool members, or VIP based on content in XML file. Specify content in profile and reference in iRule.

### OneConnect

- Re-use server-side connections using Keep-Alive.
- Use an HTTP profile with it or it could do funny things.
- Don’t use on SSL pass-through
- Opens a new connection to the server if needed when the LB algorithm chooses that server. LB algorithm is not overridden
- Load balances each HTTP request individually from the same connection.
- Will close connection after Maximum Reuse or Max Age is exceeded or the server closes it due to Idle timeout
- If SNAT is enabled, the mask is applied to the SNAT IP, which may not work well.

OneConnect Transformations - Replaces client's HTTP/1.1 Connection: close with X-Cnection: close to the server. The server will ignore the header. F5 does not send Keep-Alive in HTTP/1.1 as it's the default. The F5 inserts Connection: Keep-Alive for HTTP/1.0 requests. The server must support Connection: Keep-Alive if HTTP/1.0 and HTTP/0.9 is used. A FastHTTP profile will instead replace it with Xonnection: close

Source Mask - Specify subnet mask. This is used to specify when to use an idle connection for a client IP coming in

- 0.0.0.0 - Re-use idle connections for all connections regardless of client-IP. Multiple requests from different clients can come over one connection, appearing as if it is from one IP
- 255.255.255.0 - Allow clients with the same first 3 octets to reuse the same connections
- 255.255.255.255 - Only the same IP can re-use an idle connection

Maximum Size - How many idle connections to keep connected to servers

Maximum Reuse - How many requests to send over one connection. Set to below the max of the server Keep-Alive limit to prevent the server connection to close and entering the TIME_WAIT state

Maximum Age - How long to reuse server connection before deleting it

Idle Timeout Override - Override idle timeout in TCP profile

#### Troubleshooting

- Fast servers may have fewer open connections and more idle connections than slow servers because they finish connections quicker.
- Persistence can offset if slow servers do not respond quickly enough and more connections go to faster servers.

## - 1.01a - Given a scenario of client or server side buffer issues, packet loss, or congestion, select the appropriate TCP or UDP profile to correct the issue

### TCP Profile

**Proxy Buffer (Content spool)**:

The F5 buffers data from the server if the client has not acknowledged data quick enough. This allows the server to focus on other requests.

(Airport analogy - Bags are offloaded to the metal rollers to allow people to rest and do other things.)

May want to increase this with clients that have packet loss or are latent. Keeping both the same and high is optimal so data is buffered as soon as the client ACKs content in the send buffer.

Proxy Buffer High: The amount of data in the buffer in which to close the receive window from the server.
Proxy Buffer Low: Amount of data in the buffer that triggers the receive window to open.

(The amount of space free on the metal rollers to start accepting more bags.- 3 feet)

Send Buffer: Data that was sent but not yet acknowledged - kept just in case it needs retransmitted.
Receive Window: How much to send the F5 that can be outstanding/unacknowledged
Congestion Control: Use Woodside if some clients are wireless

**tcp-mobile-optimized**

- Buffers and window increased to allow for latency/packet loss
- Initial Congestion Window Size: 16: Sends more data unacknowledged to begin with
- Nagle - Enabled: Needing ACKs of full segment size to send more when data is outstanding. Fewer packets on latent slow networks.

**tcp-wan-optimized**

- Increased buffers
- Send Buffer and Receive window the same as mobile-optimized
- SACK and Nagle Enabled

**tcp-lan-optimized**

- Buffers, window the same as wan-optimized
- Slow Start: Disabled
- Nagle - Disabled: Nagle may hold data, when its not needed
- Acknowledge on Push: Enabled: Helpful for Windows and MacOS with small send buffers.

### TCP

**Stream-oriented** - TCP will send a continuous stream of data and divide up packets to fit into IP datagrams.

TCP Window Size, AKA Send Window:

_Send Window_**:** Amount of data that can be sent and be outstanding/unacknowledged. A 16-bit field (2^16 = 65535)

- This can be increased by using the scale factor in the Options field during the handshake, value 0 to 14.
- Take the unscaled Window, multiply it by 2^(0-14)
- Example - 65535 \* (2^14) = 1,073,725,440 bytes (a gigabyte)

Maximum Segment Size (MSS) - The maximum segment size the local host _will_ accept. It usually is 40 bytes less than the MTU size.

Default is 536 bytes, 40 bytes less than default IP MTU

## - 1.01b - Given a scenario determine when an application would benefit from HTTP Compression and/or Web Acceleration profile

Compression - Good for WAN clients with low bandwidth or latency

Caching

- High demand objects - server sends data to BIG IP when content expires
- Static content - CSS files, javascript, images, audio.

Objective 1.02 Given a subset of an LTM configuration, determine which objects to remove or consolidate to simplify the LTM configuration

### Virtual Server

#### Matching Order

Most Specific > Least specific

- Destination
- Source
- Port

VIP listener has precedence over NAT

Types

- Standard
- Forwarding (Layer 2) - No pools
- Forwarding (IP) - No pools
- Performance (HTTP)
- Performance (Layer 4)
- Stateless
- Reject
- DHCP Relay

**Address translation** - Send to different IP (Pool member) Default enabled. Disable if the pool member has the same IP as the VIP

**Port** **translation** - Automatically enabled on a Standard VIP. Connections to VS:80 -> Server:8909. Disable it if you have a multi-port VIP and want traffic sent to pool members that are listening on several ports

**Auto Last Hop** - Send return traffic from the pool of servers to the MAC address that sent the client request. This is useful if there is not a default route on the F5 and there is no route for the source. Also works to reply return traffic through transparent devices like an IPS.

**Last Hop Pool** - Send return traffic through this pool

**Clone Pool (Client)** - Specify a pool that all client traffic will get cloned to. The service port is irrelevant in the pool config.

**Clone Pool (Server)**

**Rewrite Profile** - Content Rewrite profile for HTML web application rewrites [K99872325: Modifying HTML tag attributes using an HTML profile](https://support.f5.com/csp/article/K99872325)

## - 1.02a - Evaluate which iRules can be replaced with a profile or policy setting

Use built in things over iRules.

LTM Policies override iRules

HTTP Profile

## - 1.02b - Evaluate which host virtual servers would be better consolidated into a network virtual server

If you wanted to direct clients destined to a network to a pool of firewalls.

Objective 1.03 Given a set of LTM device statistics, determine which objects to remove or consolidate to simplify the LTM configuration

## - 1.03a - Identify redundant and/or unused objects

HTTP profile on VIP without iRule or HTTP inspect/modification requirements.

## - 1.03b - Identify unnecessary monitoring

ICMP and TCP is unnecessary

Same monitors applied to the pool member and the pool

## - 1.03c - Interpret configuration and performance statistics

**show ltm pool**

**show ltm virtual**

## - 1.03d - Explain the effect of removing functions from the LTM device configuration

SSL profiles and certificates

- Note: Existing connections continue to use the old SSL certificate until the connections complete or are renegotiated or until TMM is restarted.
- Impact of procedure: Performing the following procedure should not have any impact on the existing traffic and new traffic will utilize the new certificate.

HTTP profiles

SNAT

### Removing a Pool member from a pool

Removes existing persistence entries.

If the cookie pool member is not available, the F5 makes a new load balancing decision.

Existing connections remain until idle timeout or client close.

Objective 1.04 Given a scenario, determine the appropriate upgrade and recovery steps required to restore functionality to LTM devices

### Increase disk space

[K14952: Extending disk space on BIG-IP VE](https://support.f5.com/csp/article/K14952)

Supported directories that can be increased:

- /var
- /var/log
- /config
- /shared

Verify free space - **list sys disk**

Increase space - **modify sys disk directory /var new-size** <desired value in KB>

Verify - **show sys disk directory**

**reboot**

### Console

Bits - 19200  
Data bits - 8  
Parity - N  
Stop bit - 1  
Flow control - N

### Management IP Config

GUI:  System -> Platform

TMSH: **modify sys management-**

### GUI Issues

Check or restart **httpd** and **tomcat**

**bigstart restart httpd tomcat**

### Daemons

**restart sys service <name>**

The **mcpd** daemon is the Master Control Program. Allows two-way communication between userland (applications outside the kernel) processes and TMM processes.

If this is down, traffic management does not function and nothing can be configured. Other daemons will also not function correctly.

MCP is responsible for health checks

**show sys mcp-state**
```
--------------------------------------------------------
Sys::mcpd State:
--------------------------------------------------------
Running Phase                            running
Last Configuration Load Status           full-config-load-succeed
End Platform ID received:                true
```
### MOS (Maintenance Operating System)

[Article: K14245 - Overview of the recovery tasks performed from the MOS (11.x - 15.x)](https://support.f5.com/csp/article/K14245)

Reboot into it from CLI: **mosreboot**

Boot into it using Console:

Select TMOS Maintenance from grub boot menu:
```
   GNU GRUB  version 0.97  (619K LOWER / 3653927K UPPER MEMORY)
 ******************************************************************
 * BIG-IP 12.1.3.7 Build 0.0.2 - drive sda.1                      *
 * TMOS maintenance                                               *
 *                                                                *
 ******************************************************************
```

You will be logged in as root automatically

Capabilities:

- Mount filesystems
- Use **e2fsck** to repair file system
- Set mgmt IP
- Transfer files
- Install IOS using **image2disk**

### e2fsck

For ext2/ext3/ext4 filesystems

#### Error on boot about HDD
```
[/sbin/fsck.ext3 (1) -- /shared] fsck.ext3 -a /dev/mapper/vg--db--sda-dat.share.1 dat.share.1 contains a file system with errors, check forced.
 Error reading block 5259469 (Attempt to read block from filesystem resulted in short read) while reading indirect blocks of inode 2622816.
```

**e2fsck -yf** /dev/mapper/vg--db--sda-dat.share.1

**-y** : Assume yes to all questions  
**-f** : Force check even if it seems clean  

### fsck

[Article: K73827442 - Forcing a file system check on the next system reboot (12.x - 15.x)](https://support.f5.com/csp/article/K73827442)

[Article: K60432403 - Recovering from a failed device start due to file system errors](https://support.f5.com/csp/article/K60432403)

[F5 upgrade error DevCentral](https://www.devcentral.f5.com/s/question/0D51T00006wzaNH/f5-upgrade-error)

[K61521075: BIG-IP VE failed to boot after outage with console message: UNEXPECTED INCONSISTENCY; RUN fsck MANUALLY.](https://support.f5.com/csp/article/K61521075)

#### Run on next reboot

**touch /forcefsck**  
**reboot**

OR

**shutdown -rF now**

#### Error on boot about HDD

```
set.1./var contains a file system with errors, check forced
set.1./var: Directory inode 81930, block 0, offset 0: directory corrupted
set.1./var: UNEXPECTED INCONSISTENCY; RUN fsck MANUALLY. (i.e., without -a or -p options) [FAILED]
*** An error occurred during the file system check.
*** Dropping you to a shell; the system will reboot
*** when you leave the shell.
*** Warning -- SELinux is active
*** Disabling security enforcement for system recovery.
*** Run 'setenforce 1' to reenable.
Give root password for maintenance
(or type Control-D to continue)
```

**CTRL + D**

\*login\*

**fsck -y** Automatically attempt to fix any filesystem corruption errors automagically.

### Replace down device

Remove serial cable if necessary or disable it - **modify sys db failover.usetty01 value disable**

Update Master Key on the new device to what the old device had, so that the UCS loads

- Retrieve key from failed device - **f5mku -K**
- Replace master key on new unit **f5mku -r <key>**

### Reload Configuration

[Article: K13030 - Forcing the mcpd process to reload the BIG-IP configuration](https://support.f5.com/csp/article/K13030)

Binary file that is loaded on boot mismatches text file config (bigip.conf)

**# touch /service/mcpd/forceload**

**# reboot**

Logs after reboot

The configuration DB has been initialized with 26130560 bytes of memory

Configuration restored from binary image.

### Single Configuration File

Single Configuration File (SCF) - Used to configure everything, in one file

bigip.conf - Configuration objects, VIPs, pools, profiles, iRules, authentication settings. Partitions have a separate ‘partitions’ folder at the root in the UCS file that have a bigip.conf in them.

bigip_base.conf - Base level configuration like VLANs, Self-IPs, Management IP settings, DSC settings, provisioning.

bigip_user.conf - User account config

bigip_script.conf - Custom iApp templates

SCF files are intended to help configure additional BIG-IP systems; SCFs are not intended to back up and restore a full BIG-IP system configuration or to restore a BIG-IP configuration to a later BIG-IP version.

- You want to replicate a configuration across multiple BIG-IP systems.
- You want to migrate a configuration from one platform type to another (for example, from BIG-IP VE to a hardware platform).

#### Saving an SCF file

Save the running configuration to an SCF

- **save sys config file** <SCF_filename> **\[passphrase** <passphrase>]

Specify a custom name for the TAR file when saving the configuration to an unencrypted SCF:

- **save sys config file** <SCF_filename> **tar-file** <TAR-FILE_filename>

Note: When you use the tar-file option, the actual file does not have .tar file extension by default, unless specified. Additionally, you cannot specify the passphrase option when you use the tar-file option.

The SCF file and the TAR file are saved to the **/var/local/scf/** directory.

Files referenced \[keys, external monitor files and external data group files] in config are saved in **/var/local/scf/<name>.tar**

#### Loading a SCF

When loading a SCF file, the current config is backed up in **/var/local/scf/backup.scf**

You cannot load an SCF onto another software version

To install an SCF:

- **load sys config file <SCF_filename> \[passphrase <passphrase>]**

To load the unencrypted configuration with a TAR file that has a custom name:

- **load sys config file** <SCF_filename> **tar-file** <TAR-FILE_filename>

### Diff SCF files

**show sys config-diff** <SCF_filename> \[SCF_filename_or_blank_for_running_config]

### UCS Contents & Info

- BIG-IP Config files
- Product licenses. New system should have a base license already. Use the -no-license flag. Or contact F5 to associate license with new F5 system
- SSL certs/keys
- User/passwords
- DNS zone files
- Allows restoring config to newer versions (10.x to 11.x)
- Hardware must be the same or do **no-platform-check**
- Restoring configuration for an unlicensed module will error.
- You can restore a configuration without the master key matching, but it won’t decrypt encrypted passphrases.

root@(bigip1)(cfg-sync Standalone)(Active)(/Common)(tmos)# **load sys ucs test.ucs** ?

Options:

**no-license**         This option mostly is for RMA use. It loads full configuration from a UCS file except the license file.

**no-platform-check**  Bypass platform check.

**passphrase**         Passphrase for (un)encrypting UCS.

**platform-migrate**   Don't load or modify some objects specific to a particular device.

**reset-trust**        Reset device and trust domain certificates and keys when loading a UCS.

### Restore without passphrase

**0107102b:3: Master Key decrypt failure - decrypt failure - final**

Restore the UCS and then replace the encrypted passphrases with the unencrypted ones in the bigip.conf file

## - 1.04a - Identify the appropriate methods for a clean install

### Clean Install

All mass-storage devices are wiped. Useful if the device no longer boots from any volumes.

#### Bootable USB

Must be done on a Linux workstation

4G or larger USB

Dependencies

- Linux 2.6.x kernel
- Perl 5.8 or later
- **cpio** - Manages CPIO archive files

- **mke2fs - Creates Linux ext2 file system**
- **extlinux - Lightweight bootloader that starts computers with the Linux kernel**

Unmount, plug into BIG-IP and reboot: **umount /mnt/<directory>**

You will be in MOS and prompted to perform an **automatic** **clean** **install**, just hit **\[Enter]**

#### USB DVD-ROM Drive - Recommended

Burn .iso image file to DVD.

You will be in MOS and prompted to perform an **automatic** **clean** **install**, just hit **\[Enter]**

#### PXE Boot

Can be used to install when physical access is not available and to multiple systems simultaneously.

The server providing the image must be on the same network as the management port and provide HTTP, TFTP and DHCP services.

## - 1.04b - Identify the TMSH sys software install options required to install a new version

**install sys software** { **hotfix** \| **image** } <filename> (**create-volume**)  **volume** <HDx.x>

## - 1.04c - Identify the steps required to upgrade the LTM device such as: license renewal, validation of upgrade path, review release notes, etc.

### Upgrade Code

#### Preparation

Make sure there’s enough space on the hard drive. Use “df -h” - 50GB required

#### Download code

- **/shared/images**
- GUI - System > Software Management > Image List > Import...

#### Verify image

- **md5sum** /shared/images/<name>.iso

#### List available images for installation

**list sys software image**

System > Software Management > Image | Hotfix List

#### Install to inactive volume

**install sys software** { **hotfix** \| **image** } <iso> \[**create-volume**]  **volume** <HDx.x>

**GUI -** System > Software Management > Image | Hotfix List > Install > HDx.x

Delete volume: **delete sys software volume** HD1.<x>

#### Reactivate the license (Traffic impacting)

If you fail to do this, Reboot into the old volume and reactivate.

Terms

- **License Check date**: A date specific to each software release - check online F5 solutions article
- **Service Check date: Date from the last time activated or when the service contract ends, whichever is earlier.**

If the service check date is missing or is earlier than the license check date, then the system initializes, but does not load the configuration. To allow the configuration to load, you must update the service check date in the license file by reactivating the system's license.

**TMSH**

Determine if re-activation is needed: **grep “Service check date” /config/bigip.license**. This should be a later date that the license check date of the software you’re upgrading to

**GUI**

System > License > Re-activate… > Automatic selected > Next

01070424:5: Full configuration initialization phase triggered.

01070427:5: Initialization complete. The MCP is up and running

#### Confirm Current Config will Load Upon Reboot

load sys config verify

#### Copy configuration to new volume and reboot into it

TMSH

- **cpcfg --source=HD**x.x **HD**x.x
- **reboot volume HD**x.x - OR - **switchboot**

GUI - System > Software Management > Boot Locations > HDx.x > Install Configuration \[yes] > Activate

## - 1.04d - Identify how to copy a config to a previously installed boot location/slot

**cpcfg --source=HD**x.x **HD**x.x

GUI - System > Software Management > Boot Locations > HDx.x > **Install Configuration \[yes]** > Activate

## - 1.04e - Identify valid rollback steps for a given upgrade scenario

Reboot into the old volume

- **reboot volume HD**x.x - OR - **switchboot**

Objective 1.05 Given a scenario, determine the appropriate upgrade steps required to minimize application outages

## - 1.05a - Explain how to upgrade an LTM device from the GUI

See above

## - 1.05b - Describe the effect of performing an upgrade in an environment with device groups and traffic groups

Move traffic groups to the active unit before upgrading the standby

## - 1.05c - Explain how to perform an upgrade in a high availability group

Upgrade secondary / standby first

Failover to secondary - run sys failover standby

Upgrade primary / standby

Objective 1.06 Describe the benefits of custom alerting within an LTM environment

## - 1.06a - Describe how to specify the OIDs for alerting

alert <name> “<message>” {  
snmptrap OID=”.1.3.6.1.4.1.3375.2.4.0.<**300 - 999>**”  
}

## - 1.06b - Explain how to log different levels of local traffic message logs

TMSH - **modify sys db <name>.level value <log-level-string>**

GUI - System > Logs > Configuration > Options

The severity level is in between the colons

Jul 22 07:38:28 nusiaalbipve02 mcpd\[1844]: 01070638:5: Pool member 10.0.0.154:80 monitor status forced down.

### Modify Linux host syslog levels

- **list sys syslog all-properties**
- **modify sys syslog <function>-from <function>-to <level>**
- **modify sys syslog daemon-from warning daemon-to emerg**

## - 1.06c - Explain how to trigger custom alerts for testing purposes

### Built-in Alerts

1. Find trap in **/etc/alertd/alert.conf**

2. Grep for that alert for the full logging info-

   1. **cd /etc/alertd/**
   2. **grep <alert name> \*.h**

3. Find Log Info

   1. 0 LOG_NOTICE 01070640 BIGIP_MCPD_MCPDERR_NODE_ADDRESS_MON_STATUS "Node %s address %s monitor status %s."
   2. Facility 0
   3. Log Level: Notice
   4. Alert code: 01070640
   5. Descriptive Message: "Node %s address %s monitor status %s."

Use **logger** to generate an alert - **logger -p local0.notice “**01070640:5: Node 10.128.20.20 monitor status down.”

Objective 1.07 Describe how to set up custom alerting for an LTM device

## - 1.07a - List and describe custom alerts: SNMP, email and Remote Syslog

### Configure Syslog

System > Logs > Configuration > Remote Logging

### SNMP

Place custom MIB files in **/config/snmp/<name>.tcl**

Default MIBs under **/usr/share/snmp/mibs**

#### Traps

Default SNMP Traps - **/etc/alertd/alert.conf** (Don’t add or remove traps from here)  
User-defined SNMP Traps - **/config/user_alert.conf**

System > SNMP > Traps > Destination

System > SNMP > Traps > Configuration

#### Custom Traps

1. Create backup of **/config/user_alert.conf**
2. Open and add new SNMP traps in proper format

alert <name> “<message>” {  
snmptrap OID=”.1.3.6.1.4.1.3375.2.4.0.<**300 - 999**>”  
}

3. Message -  A syslog message that will match and send the trap. Can use literal or regular expressions. For regex, put parentheses around the expression - (.\*) for example.
4. 300 - 999 - A unique OID, don’t use ones that are in F5-BIGIP-COMMON-MIB.txt file
5. The **user_alert.conf** file is appended to the **alertd.conf** file upon service start, so may need to restart **snmpd** service to take effect

### Email SNMP traps

**ssmtp** is the process for emailing

**modify sys outbound-smtp mailhub** <mail-server:port>

**/config/user_alert.conf**

alert <name> “<message>” {  
snmptrap OID=”.1.3.6.1.4.1.3375.2.4.0.<**300 - 999**>”,

email toaddress="dg-network@config.com, noc@config.com"  
fromaddress="root" ←- The FQDN will be the hostname  
body="There’s an alert!"

}

Save file and restore permissions -  **chmod 444**

Restart alertd and snmpd - **restart sys service alertd, restart sys service snmpd**

**Test Email**

**echo “**ssmtp test mail**” | mail -vs “**Test email for SOL13180**”** myemail@mydomain.com

## - 1.07b - Identify the location of custom alert configuration files

/config/user_alert.conf

## - 1.07c - Identify the available levels for local traffic logging

local0 - /var/log/ltm

local4 - /var/log/ltm

Emergency (0)

Alert (1)

Critical (2)

Error (3)

Warning (4)

Notifications (5)

Information (6)

Debug (7)

**E**very **A**lley **C**at **E**ats **W**atery **N**oodles **I**n **D**oors

Section 2 - Identify and Resolve Application Issues

Objective 2.01 Determine which iRule to use to resolve an application issue

## - 2.01a - Determine which iRule events and commands to use

CLIENT_ACCEPTED - Connection entry added 3-way handshake completes for Standard VIP, initial SYN for Performance L4.

LB_SELECTED - Pool member selected.

HTTP_REQUEST - Client request headers

HTTP_RESPONSE - Server response

## - 2.01b - Given a specific iRule event determine what commands are available

### Events

#### LB_SELECTED

active_members

active_members -list: Determine how many members are currently **available** in a pool by returning IP and port numbers.

### Operators

Relational: contains, equals, starts_with

Logical: and, if, else

### Commands

Statement: **pool, node, virtual**

Query: **IP::remote_addr, IP::addr, HTTP::host**

Data manipulation: **HTTP::header remove**

Utility: **URI::decode**

HTTP::method

HTTP::request

#### HTTP::header

HTTP::header value <header-name> - Returns value from header name

HTTP::header values <header-name> - Returns list of values from header name.

HTTP::header names

HTTP::header exists <header-name>

HTTP::header insert \["lws"] \[\[list] <header1-name> <header1-value> <header2-name> <header2-value> ] - ‘“lws”’, the system adds linear white space to long header values.

HTTP::header is_redirect

HTTP::header replace <header-name> \[<new-header-value>]

HTTP::header remove <header-name>

#### HTTP::uri

Full URI including query

#### HTTP::path

HTTP_REQUEST, SERVER_CONNECTED & more

/main/index.jsp

Does not include the query (?test.xml)

#### HTTP::query

Objective 2.02 Explain the functionality of a simple iRule

iRules are processed in order in which they appear in the GUI and CMD

Events are processed in a linear fashion so all CLIENT_ACCEPTED events in all iRules finish processing before the first HTTP_REQUEST event can fire.

Once one Event in all iRules is processed, action is taken if matched

HTTP::redirect takes precedence over pool command

**<Insert iRules here that direct to pools or nodes based on URI, host header, or IP address that even do snat as well. Use switch, if, elseif, starts_with, contains>**

## - 2.02a - Interpret information in iRule logs to determine the iRule and iRule events where they occurred

Logging in an iRule

log local0.<level-optional> “log here!”

Levels

- emerg
- alert
- crit
- error
- warn
- notice
- info
- debug

## - 2.02b - Describe the results of iRule errors

Error Message: 01220001:3: TCL error

- Stops iRule processing and may send a TCP RST
- Can happen if the wrong variable is unset at the end

Objective 2.03 Given specific traffic and configuration containing a simple iRule determine the result of the iRule on the traffic

[event](https://clouddocs.f5.com/api/irules/event.html)  
[switch](https://clouddocs.f5.com/api/irules/switch.html)  
[return](https://clouddocs.f5.com/api/irules/return.html)  
[class](https://clouddocs.f5.com/api/irules/class.html)  

The \* in front of a URL will make it so only the URL with stuff in front of that asterisk will match. If there is nothing in front, then that entry won’t be matched. Example, \*default.com, make sure to add “default.com” to the rule.

An asterisk afterwards works as ‘contains’

glob - Allows the use of limited regex on URL/URI’s (\* or  ?) that is more performant

The ‘-‘ hyphen is an OR statement

A “;” semicolon will let you have two commands on one line

- Will it only work if there’s one event in the iRule?

return - Exit from current event in the _current_ iRule.

event disable \[all] - Stop going through the current event in all iRules, Stop going through all iRule events on this connection

\-- = Stop option processing - useful if the item might start with a hyphen. Put before ‘-value’

“” = Blank value

The "switch -glob" and "string match" commands use a "glob style" matching that is a small subset of regular expressions, but allows for wildcards and sets of strings.

Make use of the "equals", "contains", "starts_with", and "ends_with" iRule operators, or the glob matching mentioned above.  They perform significantly faster, and do the exact same thing as regex. Regular expressions are very CPU intensive and should only be used when there are no other options

Use ‘switch’ instead of multiple ‘if’ lines, unless you need more than one condition.

switch “-exact” is the default if no arguments are specified

switch -glob - Supports small subset of regex (same as ‘string match’)

The switch section can contain a ‘default’ action labeled as such, like forwarding to a pool or dropping the packet if there are no matches.

**string match** ?-nocase? _pattern string_

- See if pattern matches string; return 1 if it does, 0 if it does not. If -nocase is specified, then the pattern attempts to match against the string in a case insensitive manner. For the two strings to match, their contents must be identical except that the following special sequences may appear in pattern:

\* - Matches any sequence of characters in string, including a null string.

? - Matches any single character in string

class - Allows querying of data-groups

class match - Sees if item matches in the data-group

### Example iRules

```
when HTTP_REQUEST {
    switch -glob [string tolower [HTTP::uri]] {
        # only match ‘/example/blah’ and not ‘/example/’
        "/example/*" { 
            pool server4
            snat 10.0.0.1 
        }
        # only match literal URI with nothing after it
        "/example4" { 
            pool server3
            snat 10.0.0.2 
        }
        default { 
            pool default_pool 
        }
    }
}
```
Example 2
```
when HTTP_REQUEST {
    if { [class match [IP::client_addr] equals "localusers_dg" ] } {
        pool server2
    }
    elseif { [class match [IP::client_addr] equals "remote_users_dg"] } {
        pool remote_pool
        snat 10.2.2.2
    elseif { [string match [IP::client_addr] equals "1.1.1.1"] } {
        drop
    else
      drop
    }
}
```

## - 2.03a - Use an iRule to resolve application issues related to traffic steering and/or application data

### Insecure Content on Page

Webpages/HTML responses contain links to http pages. You can use a Stream profile or/and iRule to replace http with https in the text of the HTML response page.

Do this in HTTP_RESPONSE  
<https://support.f5.com/csp/article/K31100432#irule>  

```
when HTTP_REQUEST {
   # Disable the stream filter for all requests
   STREAM::disable
   # LTM does not decompress response content, so if the server has compression enabled
   # and it cannot be disabled on the server, we can prevent the server from
   # sending a compressed response by removing the compression offerings from the client
   HTTP::header remove "Accept-Encoding"
}
when HTTP_RESPONSE {
   # Check if response type is text
      if {[HTTP::header value Content-Type] contains "text"}{
      # Replace http:// with https://
      STREAM::expression {@http://@https://@}
      # Enable the stream filter for this response only
      STREAM::enable
      }
}
```

### Allow Internet Explorer to download files through SSL

[Article: K8093 - Using iRules to modify Cache-control headers to allow Internet Explorer to download files through SSL](https://support.f5.com/csp/article/K8093)

These headers are used to prevent proxy devices from caching a local copy of the file when handling connections.

- Cache-control:no-store, Cache-control:no-cache, or Pragma:no-cache

Use this to fix the problem and allow the files to be downloaded:
```
when HTTP_RESPONSE {
   if { [HTTP::header value Content-Type] contains "application/pdf" } {
      HTTP::header replace Pragma public
      HTTP::header replace Cache-Control public
   }
   elseif { [HTTP::header value Content-Type] contains "application/vnd.ms-excel" } {
      HTTP::header replace Pragma public
      HTTP::header replace Cache-Control public
   }
   elseif { [HTTP::header value Content-Type] contains "application/msword" } {
      HTTP::header replace Pragma public
      HTTP::header replace Cache-Control public }
   else {
      return
   }
}
```

## Objective 2.04 Interpret AVR information to identify performance issues or application attacks

## - 2.04a - Explain how to modify profile settings using information from the AVR 

Maybe you could increase the buffers on the TCP profile

## - 2.04b - Explain how to use advanced filters to narrow output data from AVR 

Statistics > Analytics > HTTP > Custom Page > Add Widget

**Capture Options**

Choose what to capture

- Request and/or response that includes: Headers, Body, or All

Choose capture filters

- Virtual Server
- Node
- Protocols
- HTTP Response Codes
- HTTP Methods
- URL
- User Agent
- Client IP
- Requests and/or Responses strings

## - 2.04c - Identify potential latency increases within an application

See if the latency goes up

Objective 2.05 Interpret AVR information to identify LTM device misconfiguration

How can AVR project misconfiguration?

## - 2.05a - Explain how to use AVR to trace application traffic 

Statistics > Analytics > HTTP > Overview

Page Load Time analytics works on browsers that meet the following requirements:

- Navigation Timing by W3C
- Accepts cookies from visited site
- JavaScript enabled for site

**The following can only be changed in the default ‘analytics’ profile**

Transaction Sampling (Default profile)

- Sampling improves system performance.
- F5 recommends that you enable sampling if you generally use more than 50 percent of the system CPU resources, or if you have at least 100 transactions in 5 minutes for each entity.
- Cannot Capture traffic if Enabled.

**View Captures** - System > Logs > Captured Transactions. Limited to 1000 entries

**Notification Type**

- Syslog (System > Logs > Local Traffic)
- SNMP - Sys > SNMP > Traps > Destination (Auto sets up syslog)
- E-mail - SMTP under analytics profile

## - 2.05b - Explain how latency trends identify application tier bottlenecks

If the latency gets higher, see if pool members have been failing which results in others taking more load.

If there isn’t SSL offloading, try it to reduce latency/server load.

Objective 2.06 Given a set of headers or traces, determine the root cause of an HTTP/HTTPS application problem

**HTTP/0.9**

- GET only
- One request per connection

**HTTP/1.0**

- Introduced media

- One request per connection
- Backwards compatible with HTTP/0.9
- Only 1 domain per IP, cannot specify Host

**HTTP/1.1**

- Backwards compatible with HTTP/0.9 and 1.0
- Host header required

Improvements (Mostly performance)

- **Multiple Hostname Support**: Multiple domains per IP via Host header (required)
- **Persistent Connections: Multiple requests per TCP connection**
- **Cache and Proxy Support**

## - 2.06a - Explain how to interpret response codes

|                        |                           |                                                                                                                                               |
| :--------------------: | :-----------------------: | --------------------------------------------------------------------------------------------------------------------------------------------- |
| **Status Code Format** |        **Meaning**        | **Description**                                                                                                                               |
|        **1xx**         | **Informational Message** | Provides general information; does not indicate success or failure of a request.                                                              |
|        **2xx**         |        **Success**        | The method was received, understood and accepted by the server.                                                                               |
|        **3xx**         |      **Redirection**      | The request did not fail outright, but additional action is needed before it can be successfully completed.                                   |
|        **4xx**         |     **Client Error**      | The request was invalid, contains bad syntax or could not be completed for some other reason that the server believes was the client's fault. |
|        **5xx**         |     **Server Error**      | The request was valid but the server was unable to complete it due to a problem of its own.                                                   |

If the code received is not understood by the client, like 491 for example, a 400 code is displayed. The x00 codes are generic.

400 - Bad Request - Missing Host header or bad request

401 - Method Not Allowed

|         |                       |                                                                                                                                                           |
| :-----: | :-------------------: | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **500** | Internal Server Error | Generic error message indicating that the request could not be fulfilled due to a server problem.                                                         |
| **501** |    Not Implemented    | The server does not know how to carry out the request, so it cannot satisfy it.                                                                           |
| **502** |      Bad Gateway      | The server, while acting as a gateway or proxy, received an invalid response from another server it tried to access on the client's behalf.               |
| **503** |  Service Unavailable  | The server is temporarily unable to fulfill the request for internal reasons. This is often returned when a server is overloaded or down for maintenance. |
| **504** |    Gateway Timeout    | The server, while acting as a gateway or proxy, timed out while waiting for a response from another server it tried to access on the client's behalf.     |

## - 2.06b - Explain the function of HTTP headers within different HTTP applications (Cookies, Cache Control, Vary, Content Type & Host) / - 2.07c - Predict the browser caching behavior when application data is received (headers and HTML)

**_Cache-Control_** _-_ Directions to cache or not, a response or request.

- This overrides any default caching control for a device
- Pragma header is the HTTP/1.0 equivalent

|                             |                       |                                                                                                                                                                                                                                                                                                                                                  |
| :-------------------------: | :-------------------: | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Cache-Control Directive** | **HTTP Message Type** | **Description**                                                                                                                                                                                                                                                                                                                                  |
|       **_no-cache_**        | _Request or Response_ | The cache must check with the server to ensure that the cached data is still valid.                                                                                                                                                                                                                                                              |
|        **_public_**         |      _Response_       | May be cached by any cache, including a shared one (a cache used by many clients).                                                                                                                                                                                                                                                               |
|        **_private_**        |      _Response_       | Intended for only a particular user and should not be placed into a shared cache.                                                                                                                                                                                                                                                                |
|       **_no-store_**        | _Request or Response_ | Should not be stored in a cache. (a malicious cache operator could simply ignore the directive.)                                                                                                                                                                                                                                                 |
|        **_max-age_**        | _Request or Response_ | In a request, indicates that the client is willing to accept a response whose age is no greater than the value specified. In a response, indicates the maximum age of the response before it is considered “stale”—this is an alternative to the use of the _Expires_ header, which can fall victim to clock skew, and takes precedence over it. |
|       **_min-fresh_**       |       _Request_       | Specifies that the client wants a response that is not only not stale at the time the request is received, but that will remain “fresh” for the specified number of seconds.                                                                                                                                                                     |
|       **_max-stale_**       |       _Request_       | If sent without a parameter, indicates that the client is willing to accept a stale reply (one that has expired). If a numeric parameter is included, it indicates how stale, in seconds, the response may be.                                                                                                                                   |

**_If-Match_**: Tells the server to respond if the ETags match

**_If-Modified-Since_\*\***_:_\*\* Return the requested entity only if the resource was modified since <date/time>. Server will respond with “304 Not Modified” if it hasn’t been.

**_If-None-Match_\*\***_:_** Respond only if the ETag does **_not_\*\* match.

**_Content-Type:_** Media type and subtype

### META Headers

Included in an HTML file. “HTTP-EQUIV” meta tags are just like HTTP headers and take precedence.

Most problematic:

- **_no-cache_**: Forces a refresh even if it hasn’t changed
- **_refresh_**: Used to mimic a 302 redirect, but causes the browser to revalidate all objects referenced by the tag.

### ETag

An identifier for a resource. If resource changes, ETag must change. Helps with caching.

### HTTP Vary Header and Caching

[BIG-IP HTTP Cache and HTTP Vary header Functionality](https://support.f5.com/csp/article/K5157?sr=46612722)

[CACHE::userkey](https://clouddocs.f5.com/api/irules/CACHE__userkey.html)

**Vary**: Prevent duplicate cache entries. A header that the originating web server sends, intended for the caching device to read.

It contains things like User-Agent, if a client requests the same resource twice, with the same User-Agent, then don’t validate it

with me and just serve them the cached resource. This can cause duplicate entries of the same resource if the two users have different User-Agent strings.

- Only **User-Agent** and **Accept-Encoding** are recognized

iRule commands to help with this issue:

- **CACHE::userkey <keystring> - Creates a user-defined caching group. Overrides _Vary_ header.**
- **CACHE::useragent - Can specify a group of _User-Agents_ to serve cached content to for the same request**
- **CACHE::acceptencoding - Can specify a group of _Accept-Encoding_ headers \_\_to serve cached content to for the same request, like gzip and deflate would be the same cached entry instead of separate (duplicate)**

You can also use those commands even if the _Vary_ header isn’t present.

### Caching Commands - Show and Delete Entries

**show ltm profile ramcache** <profile_name> **max-response** <how_many_entries_to_display>

**show ltm profile ramcache** <profile_name> **host** <vip_address:port>

**delete ltm profile ramcache** <profile_name> **host** <vip_address:port> **uri /<object_name>**

**delete ltm profile ramcache** <profile_name> \*\*\*\*- Deletes all entries

## - 2.06c - Explain HTTP methods (GET, POST, etc.)

### HTTP Methods

**GET:** Proxy servers can intercept this and serve the data for you via caching. Conditions can be sent that require certain things, like _If-Modified-Since_ or _If-Match_. The server will only reply if those conditions are

then met. This is called a _conditional_ GET. You can also request certain parts of a large file with a _partial_ GET, which will have the _Range_ header inserted.

**HEAD:** Used to check if the resource is what is wanted, replies with headers only

**POST:** Used to submit data from the client. The query string is sent in the body of the submission of POST, instead of the header like GET does. The ‘? is not included

**_OPTIONS (New in HTTP/1.1):_** _Requests what communication options are available for a URI or specify an asterisk \* if about the server itself._

**_PUT:_** _Specifies to copy a file to a server. Usually FTP is used so this isn’t common. The difference with this and POST, is that POST specified the program to process the request, while this specifies the URI to_

_where the store the file._

**_DELETE:_** _Requests that the specified resource be deleted_

**_TRACE:_** _Requests a copy of the client’s request for diagnostic purposes._

**_Idempotent Methods:_** _Requests that have the same result no matter how many times they are issued. Usually GET and HEAD._

## - 2.06d - Explain how to decode POST data.

### Decode POST Data

POST has the query string sent in the Body of the submission of POST, instead of the header like GET does. The ‘? is not included.

- user=joe+and+bob&food=burger

URL’s must be ASCII, characters not in ASCII must be encoded into such. Encoded will be % followed by two hexadecimal digits.

Spaces cannot be in the URL

“ “ = + or %20

@ = %40

**URI::decode** Is a function used to decode a URI, under HTTP_REQUEST

## ~~- 2.07a - Investigate the cause of a specific response code~~

## - 2.07b - Investigate the cause of an SSL Handshake failure

### Handshake Failures

Date/time incorrect on either side causes conflicts in certificate validity.

The protocol versions are identified as follows

- SSLv3 - 3.0
- TLS1.0 - 3.1
- TLS1.1 - 3.2
- TLS1.2 - 3.3

### SSLDUMP failure examples

##### Server rejects client’s protocol version
```
S>CV0.0 Alert
level fatal
value handshake_failure
```
##### Server rejects client’s cipher suites
```
C>SV3.2 Handshake
ClientHello
Version 3.2
TLS_<ciphers>

S>CV3.2 Alert
level fatal
value handshake_failure
```
##### Device does not accept SSL
```
C>SV3.1 Handshake
ClientHello
Version3.2

period of waiting

C>S TCP FIN
```
Enable SSL debug in /var/log/ltm -  **modify sys db log.ssl.level value debug** (Default is Warning)

~~Objective 2.07 Given a set of headers or traces, determine a solution to an HTTP/HTTPS application problem~~

~~- 2.07a - Investigate the cause of a specific response code~~

~~- 2.07b - Investigate the cause of an SSL Handshake failure~~

~~- 2.07c - Predict the browser caching behavior when application data is received (headers and HTML)~~

~~Objective 2.08 Given a direct trace, a trace through the LTM device, and other relevant information, compare the traces to determine the root cause of an HTTP/HTTPS application problem~~

~~- 2.08a - Given a failed HTTP request and LTM configuration data determine if the connection is failing due to the LTM configuration~~

## Objective 2.09 Given a direct trace, a trace through the LTM device, and other relevant information, compare the traces to determine a solution to an HTTP/HTTPS application problem

~~- 2.09a - Investigate the cause of an SSL Handshake failure~~

## - 2.09b - Given a failed HTTP request and LTM configuration data determine if the connection is failing due to the LTM configuration

Packet Process Order: Packet filter > Connection table > Virtual Server > NAT > SNAT > Self IP

### Auto Last Hop

Global Setting under: System > Configuration > Local Traffic > General > Auto Last Hop \[ X ]

Send the response back to the MAC of the client requestor, overrides the routing table. Useful if there’s no matching route for the client, or if transparent IPS devices don’t modify the source IP and a response could go to the wrong interface.

Can be overridden locally with Enabled, Disabled, or Default (inherit global) on VIPs, and SNATs

### Failed Request Captures

Pictures of Wireshark or TCPDUMP traces that show these issues

1. ClientSSL profile needs applied - ClientHello sent, no response, timeout after 300 seconds, C>S FIN
2. ServerSSL profile needs applied - ClientHello sent, no response, timeout after 300 seconds, C>S FIN
3. Server needs configured with SSL - ClientHello sent, no response, timeout after 300 seconds, C>S FIN
4. SNAT needs enabled - S, S, S, R

### Persistence

**show ltm persistence persist-records**

Statistics > Module Statistics > Local Traffic > Persistence Records

May need to enable: **modify sys db ui.statistics.modulestatistics.localtraffic.persistencerecords value true**

An iRule can override a persistence profile decision

SSL Persistence - Verifies SSL session ID, when there is no client SSL profile applied. IP changes can happen and not affect this. If client SSL applied, use iRule

#### Cookie

Insert, Rewrite, and Passive are resistant to persistence mirroring, and failover. Everything needed is in the cookie.

Expiration

- Session (Default) – Valid until the browser is closed

- Time-based

Override Connection Limit: Pool member limits overridden. Virtual Server ones are not.

##### Insert (Default) 

**BIGipServer<pool_name>** cookie is inserted. The server port and address are encoded. Unable to maintain persistence across virtual servers (HTTP/HTTPS) if pool member ports are different.

##### Rewrite

Intercepts **Set-Cookie** header named **BIGipCookie** from the server and puts in there **BIGipServer<pool_name>**. Cookie field needs to be blank with 120 x 0’s, or 75 for backwards compatibility (has caveats).

Use over passive mode whenever possible

##### Passive 

Cookie read from the server to determine how to persist based on the encoded address of the pool member and timeout. Not recommended if rewrite is an option.

##### Hash

Web server generates a cookie and LTM creates a hash of it to determine where to send it. Must specify a timeout in seconds

Persistence can be ignored for Cookie, Universal and Hash if different persistence entries are sent over the same TCP connection via Keep-Alive.

[The BIG-IP system may appear to ignore persistence information for Keep-Alive connections (f5.com)](https://support.f5.com/csp/article/K7964)

- Use OneConnect to evaluate persistence for each HTTP request.

Fallback persistence - source IP only

#### Source Address 

Timeout: Default 180 seconds

Match across Services, VIPs, Pools

Hash: CARP or Default

Mask

Override Connection Limit

#### Destination Address

Good for caching servers

#### Hash

Uses data from iRule (persist HTTP header)

Two algorithms, Default and CARP

#### Universal Persistence

[Overview of universal persistence (f5.com)](https://support.f5.com/csp/article/K7392)

- Use iRule to persist off of data.
- Use OneConnect to load balance effectively
- **persist uie** - persist off of HTTP_REQUEST client-side request
- **persist add uie** - used on HTTP_RESPONSE lookup of text to persist from

#### Matching Across Options

Pools with different node IPs or pools with multiple members with the same node IP may cause issues

Available in the following persistence profiles:

- Source Address
- Destination Address
- Cookie hash
- SSL
- Universal
- Hash
- Microsoft RDP
- SIP

Match Across Services - VIPs or Pools that have the same IP but different ports. Uses the node IP address to find a persistence match in other pools. Fixes secure checkout HTTP to HTTPS issues.

Match Across Virtual Servers - VIPs that have different IPs and Ports. Uses the node IP address to find a persistence match in other pools.

Match Across Pools - Once a pool and member is selected, always return to that pool even if iRule says different and the URL is different.

- Useful if you have one VIP redirecting to multiple pools using an iRule
- Beware if the same client connects to another VIP that has Match Across Pools enabled, they will get directed to the same pool behind the other VIP

### _Packet Filters_

- _Applies to incoming traffic only_
- _Expressions are written in [libpcap ](http://www.tcpdump.org/manpages/pcap-filter.7.html)syntax like tcpdump commands - “src host and src host”_
- _Can be applied to all VLANs to evaluate traffic coming into the BIG-IP from everywhere_
- _Must Enable, default it is disabled_

#### _Criteria_

- _Source IP_
- _Destination IP and port_
- _Protocol_
- _VLAN_

#### _Options_

**_Unhandled Packet Action_** _- Accept, Discard, or Reject are the options. Default is to Accept_

**_Filter established connections_** _- Default disabled. Checking this can degrade performance and does not improve security_

**_Send ICMP error on packet reject_** _- Default disabled. When enabled, BIG-IP sends an ICMP type 3 (destination unreachable), code 13 (administratively prohibited) packet when one is rejected._

_When disabled, sends ICMP reject that is protocol-dependent (TCP reset?)_

##### *Exemptions* 

_VLANs - Default is None_

_Protocols_

- _ARP (default accepted)_

- _Important ICMP (default accept)_

_MAC addresses - Default is None (subject to packet filtering)_

_IP Addresses_

##### _Actions_

- _Accept, Discard - Stops processing down the rules_
- _Reject - Sends reject packet - Depends on **Send ICMP error on packet reject** option_
- _Continue - Acknowledges packet for logging/statistical purposes and continues_

### SNAT

Local Traffic > Address Translation

[The order of precedence for local traffic object listeners](https://support.f5.com/csp/article/K9038)

[K7820: Overview of SNAT features](https://support.f5.com/csp/article/K7820#bp)

- Well known ports are usable as source ports
- Can use more than one floating self IP in automap
- Each SNAT address can have more than 65,535 translations
  - Source/Dest IP and Port numbers create unique translation
- Maximize SNAT IP use - Adjust TCP/UDP idle timeout - 60 seconds for HTTP, 120 - 300 seconds for FTP (Control port will remain idle while data is transferring)

#### Pools

- Avoids port exhaustion
- Improve performance for FastHTTP VIP
- Addresses are used via least connections to members
- Configurable Idle timeout value (Unlike Automap)
- F5 only uses IPs in the egress VLAN when there are IPs from multiple VLANs in the same pool on multiple VIPs

Automap - Uses the self-IPs that egresses to the destination server according to the route table. Uses floating IP if HA.

#### SNAT List

A global configuration for SNAT

Origin - All addresses | Address list

Translation - Address list, pool, automap

Source Port

- Preserve
- Preserve Strict - Reset client connection if cannot get same source port
- Change - Use next available port

VLAN

Stateful Failover Mirror \[   ]  
Auto last hop \[ Default | Enabled | Disabled  ]

#### SNAT Translation List

SNATs will appear here that you configured, such as a pool.

Can only configure these options from this menu

- TCP/UDP/IP Idle timeout (Defaults Indefinite)

- Connection Limit
- ARP toggle
- Traffic group selection

### FastL4 Profile

[K01155812: Overview of the Performance (Layer 4) virtual server](https://support.f5.com/csp/article/K01155812)

[K16446: The BIG-IP system now allows a Performance (Layer 4) virtual server to have an associated HTTP profile](https://support.f5.com/csp/article/K16446)

A reset is sent immediately before 3 way handshake when the pool members are unavailable

Able to assign HTTP profile and read HTTP with iRules or gather statistics with AVR

Assigned to the following VIP Types

- Performance (Layer 4)
- Forwarding (Layer 2)
- Forwarding (IP)

**Limitations**

- No HTTP optimizations
- No TCP optimizations for server offloading
- SNAT/SNAT Pools demote PVA to Assisted
- iRule events limited to CLIENT_ACCEPTED and SERVER_CONNECTED
- No OneConnect
- Limited Persistence - Source and destination address, Universal
- No compression
- No VIP authentication
- No HTTP Pipelining

### FastHTTP Profile

Assign to a Performance (HTTP) VIP Type

Packet-by-packet decisions

SNAT Pool can increase performance

Single TCP Connection, not Full proxy

When to use?

- Traffic with few network problems, such as dropped or out-of-order packets
- Traffic generated by well-behaved clients and server
- Traffic with protocol headers contained within a single packet
- Traffic produced by load generators

Advantages

- Fast HTTP combines TCP, HTTP, and OneConnect profiles into a single profile that is optimized for network performance.
- The Fast HTTP profile is designed to reduce network latency and system CPU usage.
- Compatible with persistence

Disadvantages

- SNAT Required
- No IPv6 support
- No SSL offload
- No compression
- No caching
- No PVA acceleration
- No virtual server authentication
- No state mirroring
- No HTTP pipelining
- No TCP optimizations
- Limited iRule Header expansion
- iRules are limited to L4 and a subset of HTTP header operations and pool/pool member selection
- Out of order packets are dropped

**Settings:**

Request Header Insert

XFF Insert

Objective 2.10 Given a scenario, determine which protocol analyzer tool and its options are required to resolve an application issue

HTTPWatch

Fiddler

### ssldump

Best to capture traffic with tcpdump, then decrypt with ssldump. It is possible to decrypt live traffic with ssldump though.

Flags

- \-i - Interface

- \-r - Read pcap

- \-A - Display all fields (default is most interesting only)

- \-e - Print actual timestamps (1604107938.9487 (0.1394)  C>S  Handshake)

  - (Without - relative) 0.1238 (0.1238)  C>S  Handshake

- \-d - Display app data, including before session

- \-M - Export a (symmetric) pre-master secret log file

- \-k - Specify private key file path

### TCPDUMP

[Packet trace analysis](https://support.f5.com/csp/article/K1893)

[Tcpdump – Interpreting Output](http://packetpushers.net/masterclass-tcpdump-interpreting-output/)

[Overview of packet tracing with the tcpdump utility](https://support.f5.com/csp/article/K411)

[Capturing internal TMM information with tcpdump](https://support.f5.com/csp/article/K13637)

[TCPDUMP](http://www.tcpdump.org/tcpdump_man.html)

\-i - Interface, physical or VLAN - mgmt = eth0 (do not capture on interface with colon)

- If you don’t specify one, it defaults to eth0 (mgmt)
- Use 0.0 (loopback) to capture traffic in all route-domains and VLANs
- Specifying VLAN disables hardware checksum L4 UDP/TCP, and calculates it in software
- :p - Follow and capture traffic through the entire flow - SNAT and OneConnect included. Only works correctly if using ‘src host <x.x.x.x.>’ and ‘dst host <x.x.x.x>’ filters. If you use ‘host <x.x.x.x>’ it will capture everything.
- :nnn - Low, Medium, High details from TMM

\-n - Disable name resolution

\-nn - Disable name and port resolution

\-e - Print MAC addresses

\-v\[vv] Verbose

- Displays IP headers, options, and amount of packets captured when using -w

\-w - Write to file (tcpdump > file-name - writes text only)

\-C - In megabytes the size of the circular buffer file

\-W - Number of max files to create for circular buffers

\-r - Read binary file

- F5 adds pseudo header to beginning of file that includes - tcpdump syntax, code version, hostname, platform, product

\-s, --snapshot-length=snaplen  - How much data in **bytes** to capture in each packet - 0 = 65535. Example, -s200 - Default 68 bytes.

- Only necessary when displaying/reading and not writing to a file.

\-X - Display hex and ASCII of the actual data

\-c \[count] -  Exit after so many packets are captured. Useful to prevent capturing too much and affecting the system performance.

\-K - Disable checksum (Good if there is checksum offload on the NIC)

\-q - Quiet mode, only posts src/dst IP/port, length and DF bit or not, NO FLAGS

\-ttt - Display time delta since last packet

\-Z - User

#### Operators

and | or | not

#### Filters

**src | dst \[ net | host | <ip> ]**

#### Output

**IPv4**

IP is printed at the beginning if -e is not used.

If -v is used:

tos _tos,_ ttl _ttl_, id _id_, offset _offset_, flags \[_flags_], proto _proto_, length _ip-packet-length_, options (_options_)

**TCP/UDP**

_src_ > _dst_: Flags \[_tcpflags_], seq _data-seqno_, ack _ackno_, win _window_, urg _urgent_, options \[_opts_], length _len_

Flags: S, F, P, R, U(URG), W (ECN CWR), E (ECN-Echo), or ‘.’ (ACK), or ‘none’ if no flags are set.

IP Src, Dst, and flags are always present, others are conditional.

16:09:39.397858 IP 10.128.10.50.80 > 10.128.10.1.55945: Flags \[P.], seq 1074417:1074792, ack 413, win 4792, options \[nop,nop,TS val 3976789900 ecr 226910010], length 375: HTTP

- seq 1074417:1074792 - Beginning and ending sequence (matches length)
- length - TCP data payload length (not headers)

**SYN Packet**

16:09:37.343607 IP 10.128.10.1.55945 > 10.128.10.50.80: Flags \[SEW], seq 1505695674, win 65535, options \[mss 1460,nop,wscale 5,nop,nop,TS val 226908491 ecr 0,sackOK,eol], length 0

## - 2.10a - Identify application issues based on a protocol analyzer trace and determine solution

Virtual Host: Apache setting to allow another host header binding.

### No Response

- SNAT

  - Source and Destination pool member in same subnet
  - Pool member doesn’t have LB as gateway

## - 2.10b - Explain how to follow a conversation from client side and server side traces

### Following a Trace

- Use 0.0:p (loopback interface) and specify source and destination IPs
- Match timestamps on both sides (client and server)
- Match source port number

### Following a Trace in Wireshark

- In the F5 ETH trailer, the flow ID and peer ID can be used to follow a full trace of the connection. Copy them as a Filter and put ‘or’ in it. This will catch both sides of the connection.

## - 2.10c - Explain how SNAT and OneConnect affect protocol analyzer traces

### SNAT

Automap will use floating self-IP of interface - will preserve the source port by default

### OneConnect

tcpdump won’t show 3 way handshake on the server-side and will show multiple HTTP requests on server side over the same connection, while the client side will show multiple connections from multiple clients.

## - 2.10d - Explain how to decrypt SSL traffic for protocol analysis (v11.5)

[K10209: Overview of packet tracing with the ssldump utility](https://support.f5.com/csp/article/K10209)

### Read application data - Symmetric keys

1. Disable ECDHE in the SSL profile
2. Capture traffic with tcpdump
3. Create pre-master secret (symmetric key)

**ssldump -r <capture-file.pcap> -k <private-key.key> -M <pre-master-secret-log>**

The output of the pre-master-secret log file can now be loaded into Wireshark. (Edit > Preferences > Protocols > SSL > PMS key log filename.

## - 2.10e - Explain how to recognize the different causes of slow traffic (e.g., drops, RSTs, retransmits, ICMP errors, demotion from CMP)

ICMP Errors - Packet filter being used and sending ICMP error on Reject

ICMP destination unreachable - can be multiple things, like the destination host cannot be reached by the gateway. Or there isn’t a route that the gateway has to the destination

### Drops

- No SNAT

  - Source and Destination pool member in same subnet
  - Pool member doesn’t have LB as gateway

- MTU size

  - Switches can silently drop packets that have too big of a MTU size, only routers can fragment

- TM.RejectUnmatched **false**

  - Packet received on non listening port or invalid sequence (segment) - _Silent Drop_

- TM.MaxRejectRate - 250 TCP RSTs/ICMP UNR / second

  - Too many resets are being sent, _silently drop_

### Resets

**show net rst-cause**

db variable **tm.rstcause.log value enable** - Displays “RST sent from <source> to <destination> and the reason in logs

db variable **tm.rstcause.pkt value enable** - Displays RST cause in TCP payload

- Can use **tcpdump -i <int>:nn** to display the cause without this set.

Reaping - RST when high-water mark is reached and no new connections are allowed

TM.RejectUnmatched **true**

- SEQ received on VIP or self-ip that doesn’t match current connection
- SYN received on VIP or self-ip for port that isn’t listening

VIPs

- Reject VIP - Always
- Connection Limit reached
- VLAN not allowed

Pools

- All down
- Nodes or Pool Connection limit reached

Profiles

- Idle timeout reached (**Reset on Timeout** set) Default 300 seconds

  - If multiple profiles with idle timeout, use smallest value

- 8 re-transmissions

- 3 SYNs

SNATs

- 3 Unacknowledged SYNs processed
- Idle timeout
- Port exhaustion

Monitors

- RST when TCP and TCP_Halfopen passess.
- HTTP monitor may RST after receive string is received

iRule - **reject** command

Packet filter - **reject** action

### Duplicate ACKs

- Packet loss
- Resource Exhaustion
- Extreme increase in latency

### Slow Traffic

**Dropped packets**

- Duplicate ACKs
- Duplicate SEQ

### CMP - Cluster Multiprocessing

Creates separate TMM processes per CPU to share the load across all.

Demotion - iRule with Global variable, **set::** or **$::** demotes CMP to single.

**show ltm virtual <vs-name>**

- CMP - **enabled/disabled**
- CMP Mode - **all-cpus**, **single, none**, **disabled**

Objective 2.14 Given a trace, identify monitor issues

## - 2.14a - Explain how to capture and interpret monitor traffic using protocol analyzer

Capture traffic from non-floating self-IP to pool member

## - 2.14b - Explain how to obtain needed input and output data to create the monitors

Retrieve a health check file and response from cURL or use openssl, telnet, or ncat.
```
curl http://10.128.10.10/health-file -H “Host: f5demo.com”
curl http://f5demo.com --resolve “f5demo.com:80:10.128.10.10”
nc -v 10.128.10.10 443
telnet 10.128.10.10 443
openssl s_client -connect 10.128.10.10:443
```
Objective 2.15 Given a monitor issue, determine an appropriate solution

## - 2.15a - Determine appropriate monitor and monitor timing based on application and server limitations

[Troubleshooting Health Monitors](https://support.f5.com/csp/article/K12531)

[Content length limits for HTTP(S) monitors](https://support.f5.com/csp/article/K3451)

[ftpd man page](https://linux.die.net/man/8/ftpd)

[Overview of the FTP health monitor](https://support.f5.com/csp/article/K11481312)

[Response code support for FTP](https://support.f5.com/csp/article/K13612)

[Configuring an active FTP monitor for pool members that listen on non-standard FTP port](https://support.f5.com/csp/article/K12921)

[K61010136: Overview of the inband monitor](https://support.f5.com/csp/article/K61010136)

Timeout should be 3 times the interval + 1.

The appropriate monitor should be TCP, or HTTP for most apps. ICMP should not be used.

### Types

#### Active

- Extended Content Verification (ECV) - Verify content - status.html - ONLINE
- Extended Application Verification (EAV) - Verify path or service - Port 443, or HEAD /

#### Passive

- Passively monitors traffic, does not generate any

- Only 1 called “inband”

- Marks down if client requests timeout to the server or get a TCP reset

- Cannot specify the responses to mark down/up on

- Fast to mark servers down

- Slow to mark servers up

- Failures - 3

- Failure Interval - 30 seconds

- Response Time - 10 seconds

- Retry Time - 300 seconds. Time before re-checking a down pool member’s status. If 0, won’t be turned back on without manual intervention or another monitor with ‘Time Until Up’ is used.

- ~~Apply HTTP monitor and enable Up Interval, set it to high and Time Until Up. This ensures that once the member is up, stop monitoring it so often and instead use the inband monitor to look for errors.~~
  - ~~The Interval set will be used when its down~~

### Pools

[K47726919: FQDN ephemeral nodes on the BIG-IP system do not repopulate after configuration load](https://support.f5.com/csp/article/K47726919)

#### Options

**FQDN** - Queries DNS, adds all results as members.

Members will show as a separate node and display as “nodename-10.1.1.100” (11.x) or  “\_auto_10.1.1.100” (12.+)

**Auto Populate** - Remove and add members as DNS record changes (checks every hour by default or can use the TTL). You can restart **bigd** to force a lookup.

**IP Encapsulation**

- GRE/IPIP - nPath/Direct Server Return - Creates IP encapsulation tunnel to server. The server de-encapsulates the packet and then directly responds to the client in the original IP header.
- [AskF5 | Manual Chapter: Configuring Layer 3 nPath Routing](https://techdocs.f5.com/en-us/bigip-15-0-0/big-ip-local-traffic-manager-implementations/configuring-layer-3-npath-routing.html)

**Slow Ramp** - Slowly increases connections sent to a server coming up. The member will not be considered active in a priority group until the Slow ramp time is done.

**Allow SNAT -** Uncheck to override Virtual Server SNAT.

**Reselect Tries** (Use with TCP profile only, not fastL4) - Number of times to try and select a new pool member once one has failed for a client. Used with inband monitor

**TCP Request Queue** - Queue connection if connection limit is reached.

##### Action on Service Down

What happens when a pool member is marked down.

None - _Default_. Removes existing persistence entries. If the cookie pool member is not available, the F5 makes a new load balancing decision. Existing connections remain until idle timeout or client close.

Reject (Reset in TMSH) - RST or ICMP Unreachable is sent to the client.

Drop - Silently deletes connections - OK for UDP short-lived (DNS queries)

Reselect - Moves client connection without tearing it down.

- VS with port and address translation disabled (FastL4)
- Pool members that don’t care about the state of the connection (Cache servers, firewalls, routers, proxy servers)
- UDP VS
- If you did this on a TCP VS, the server would send a RST (no existing connection)

#### Members

**Down:** Persistence record is removed, no new connections will be directed to that member.

**Disabled:** Receive new connections from persistent connections. No new connections allowed from clients without a persistence record

**Forced Offline:** Persistence record is removed, no new connections will be directed to that member.

Existing connections can end gracefully and will receive an ICMP Echo Unreachable or TCP RST if the idle timeout is reached.

**Connection Rate Limit** - Optimal 300 - 5000 connections per second. Max is 100000.

##### Priority Groups

Backup Servers will be placed in lower priority than others.

Priority Group Activation**:** Disabled,  Less than..

- Less than **0** members - Disables it

Members

- 0 is lowest/last
- Higher = more priority
- If the Less than… is exceeded, the next highest priority group is activated (Less than 3.. Two left > Next highest activated )
- Connection limits do not mark them down and does not activate another group

### FTP

[ftpd man page](https://linux.die.net/man/8/ftpd)

[Overview of the FTP health monitor](https://support.f5.com/csp/article/K11481312)

[Response code support for FTP](https://support.f5.com/csp/article/K13612)

[Configuring an active FTP monitor for pool members that listen on non-standard FTP port](https://support.f5.com/csp/article/K12921)

Logs in, downloads file to /var/tmp

### External

[Requirements for external monitor output](https://support.f5.com/csp/article/K7444)

Any output from an external monitor to **stdout** will result in a member being Available. If the script fails, do not write failed status to **stdout**

Writing to stdout - ‘**echo** <string>’

## - 2.15b - Describe how to modify monitor settings to resolve monitor problems

Default setting for multiple monitors on a pool is for All to pass.

### HTTP

[Content length limits for HTTP(S) monitors](https://support.f5.com/csp/article/K3451)

[K83316932: Overview of the Manual Resume feature for BIG-IP LTM monitors](https://support.f5.com/csp/article/K83316932)

Limit of length to search - ~5k bytes - F5 will send a reset after it receives the string it needs or reaches the limit

**Up Interval** - How often to check when resource is up (default disabled)  
**Interval** - How often to check, if node is up or down  
**Time until Up** - How long to wait after successful check before marking up

**Send String**

- HTTP version differences

- 0.9 - GET /\\n or GET /\\r\\n

- 1.0 - GET / HTTP/1.0\\r\\n\\r\\n or GET / HTTP/1.0\\r\\n

- 1.1 - GET / HTTP/1.1\\r\\nHost: domain.com\\r\\n\\r\\n

  - Connection close optional at the end, Keep-Alive is default

- Marks server up if response is received and no Receive String is configured (200, 404, etc)

**Receive Disable String** - Mark member down (Disabled)

- Receive string will override Receive Disable string and mark the member up

**Manual Resume** - Require manual enabling when down.

**Reverse option** - Marks monitor down when the test is successful. Receive disable string unavailable  
**Transparent** - Force the monitor check through this pool member by using the pool member as a gateway to the Alias address/port. Good for gateway pools.  
**Alias Address and Port** - Actually test this address/port to determine if this monitor should pass and not the pool member IP:port. Use this in combination with Transparent mode \[√] to check through a device.

Section 3 - Identify and Resolve LTM Device Issues

Objective 3.01 Interpret log file messages and/or command line output to identify LTM device issues

### LCD Messages

[Manual Chapter: The 4000 Series Platform](https://techdocs.f5.com/kb/en-us/products/big-ip_ltm/manuals/product/b4000-platform/2.html#c_reuse_led_about)

#### Alert lights

|                 |           |
| --------------- | --------- |
| Blinking red    | Emergency |
| Solid red       | Alert     |
|                 | Critical  |
| Blinking yellow | Error     |
| Solid yellow    | Warning   |

Status LED

- Solid Green - Active
- Solid yellow - Standby

### Alert conditions

**Blinking red** - PSU, Bad HDD

**Solid red** - License, DDoS, CPU, Chassis

**Warning** - Unit going standby/active

## - 3.01a - Interpret log file messages to identify LTM device issues

Covered already under 3.02a/b

## - 3.01b - Interpret the qkview heuristic results

Not sure how to study for this

## - 3.01c - Identify appropriate methods to troubleshoot NTP

### Prefer NTP server
```
tmsh edit sys ntp all-properties
include “<server ip> prefer”
:wq!
list sys ntp servers
```
### Troubleshooting

[NTP's REFID](https://www.nwtime.org/ntps-refid/)

NTP = UDP port 123

**show sys service ntpd**

ntpd (pid  25898) is running…

**list sys ntp all-properties**

**# ntpq -np**

\-n    Disable hostnames

\-p    Print peers - required for below output or else enters interactive mode

### Output

|              |                                                                                                                                                                         |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Variable** | **Description**                                                                                                                                                         |
| [tally]     | * = Preferred server{{< line_break >}}+ = Next available server{{< line_break >}}space, x, period (.), dash (-), or pound (#)  = Peer not used - unavailable, not needed, requirements unmet |
| remote       | Peer address                                                                                                                                                            |
| refied       | Reference ID.CMDA..GPS.                                                                                                                                                 |
| st           | Stratum ID{{< line_break >}}0 = invalid/unspecified{{< line_break >}}1 = primary / reference clock (GPS/CMDA, etc.){{< line_break >}}2-15 = secondary source, refid changes to remote clock’s peer IP{{< line_break >}}16+ = unsynchronized     |
|              |                                                                                                                                                                         |
| t            | u: unicast address, b: broadcast, l: local address                                                                                                                      |
|              |                                                                                                                                                                         |
|              |                                                                                                                                                                         |
| reach        | 0 = server not reached                                                                                                                                                  |

#### Failed

```
remote       refid  st t when   poll reach delay offset jitter
==============================================================================
172.28.4.133 .INIT. 16 u 64     0    0.000 0.000 0000.00
```

**st 16 =** inoperative NTP server or not viable  
**refid .INIT** = Initializing request to server, not reached.  
  
```
remote       refid        st t when poll reach delay offset  jitter
==============================================================================
172.28.4.133 10.10.10.251 4  u 482  1024 377   0.815 -10.010 0.345
```

#### Normal
Example 1
```
remote         refid        st t when poll reach delay offset  jitter
==============================================================================
*172.16.24.120 10.30.40.251 4  u 482  1024 377   0.815 -10.010 0.345
+172.17.24.130 10.20.40.252 6  u 482  1024 179   0.215 -1.010  0.545
```  
Example 2
```
remote    refid        st t when poll reach delay  offset       jitter
==============================================================================
*10.0.0.1 172.18.20.40 2  u 174  1024 377   39.833 -0.447        1.107
+10.0.1.1 .GPS.        1  u 285  1024 377   55.782 -1.010 -0.661 1.562
```

## - 3.01d - Identify license problems based on the log file messages and statistics

[K7747: Error Message: SSL transaction (TPS) rate limit reached](https://support.f5.com/csp/article/K7747)

01260008:3: SSL transaction (TPS) rate limit reached

Objective 3.02 Identify the appropriate command to use to determine the cause of an LTM device problem

You can use **run util <command>** from TMSH if you don’t feel like going down to bash

**run util tcpdump -i 1.1 host 1.1.1.1**

Not all programs are there, such as ‘top’

## - 3.02a - Identify hardware problems based on the log file messages and statistics

**show sys log ltm \[ range** \<time-parameters> **]**

Max lines is 10000, and will not display it if so  
Seconds are optional  
The following require **range**

now-3d - 3 days ago.  
now+3h - 3 hours from now. ← This is impossible  
now-3m - 3 minutes ago.  
now+3w - 3 weeks from now. ← This is impossible

**show sys log ltm** 2022-09-15:00:00:00 ← Cannot get this to work.  
**show sys log ltm range now-1h** - Now to 1 hour ago  
**show sys log ltm range** 2022-09-15:00:00:00--2022-09-14:00:00:00  

If you don’t specify a destination in the range, it will be 5 minutes

```
SYNTAX
Date/Time Syntax
  now[ [ + | - ] <integer> [ d | h | w | m ] ]
  yyyy-mm-dd[ : | T ]hh:mm[:ss]
  mm-dd[-yyyy][ : | T ]hh:mm[:ss]
  mm/dd[/yyyy][ : | T ]hh:mm[:ss]

Date Range Syntax
  now[ [ + | - ] <integer> [ d | h | w | m ] ]--now[ [ + | - ] <integer> [ d | h | w | m ] ]
  yyyy-mm-dd[ : | T ]hh:mm[:ss]--yyyy-mm-dd[ : | T ]hh:mm[:ss]
  mm-dd[-yyyy][ : | T ]hh:mm[:ss]--indefinite
  epoch--mm/dd[yyyy][ : | T ]hh:mm\[:ss]
  now[ [ + | - ] <integer> [ d | h | w | m ] ]
```

### End User Diagnostics

[AskF5 | Manual: Field Testing F5 Hardware: F5 BIG-IP iSeries Platforms](https://techdocs.f5.com/kb/en-us/products/big-ip_ltm/releasenotes/related/eud-sf.html)

Options to run

1. Download ISO file and burn to CD. Use USB DVD drive in system
2. Download IM file, install it, then install EUD to a flash drive

It will automatically boot or select End User Diagnostics from the boot menu from console

Select Option 21 to reboot and exit. Any other way may cause instability

“Completed test with 0 errors”

Diag file **- /shared/log/eud.log**

Use command **eud_info** to see version installed

#### USB Flash drive

Download .im file to /var/tmp

Install it - **im** <file_name>.im

Loopback mount the downloaded file - **mkdir** **/tmp/eud; mount** **-o ro,loop** <file_name>.im **/tmp/eud**

Insert flash drive into BIG-IP

Run mkdisk and follow prompts to install EUD onto flash drive - **cd /tmp/eud; ./mkdisk**

### SSL Hardware Accelerator

[K16951: Overview of SSL hardware acceleration fail-safe](https://support.f5.com/csp/article/K16951)

crit tmm\[6789]: 01010025:2: Device error: cn0 device requires hard reset; trying soft reset

crit tmm2\[6789]: 01010025:2: Device error: cn2 PCI write master retry timeout

### Hard Drive Issues

[Article: K14426 - Hard disk error detection and correction improvements](https://support.f5.com/csp/article/K14426)

**pendsect** - Process that runs daily to check for HDD errors and logs to **/var/log/user.log** and **/var/log/messages.** Doesn’t apply to SSDs

|                                                                                                                            |                                                                          |
| -------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| **Log Entry**                                                                                                              | **Outcome**                                                              |
| no Pending Sectors detected                                                                                                | **No errors detected**                                                   |
| Recovered LBA:230000007{{< line_break >}} Drive /dev/sda partition UNKNOWN{{< line_break >}}File affected NONE                                                  | **Errors detected and corrected**                                        |
| seek(1) error\[25]  recovery of LBA:226300793 not complete{{< line_break >}}Drive: /dev/sda filesystem type: Undetermined{{< line_break >}}File affected: NONE | **Error detected and unrecoverable**                                     |
| crit smartd\[4652]: Device: /dev/sda, 1 Currently unreadable (pending) sectors                                             | **Possible Failure or just busy and recovers** - **/var/log/daemon.log** |

**Confirming Hard Drive Failures**

[Platform Diagnostics](https://techdocs.f5.com/content/kb/en-us/products/big-ip_ltm/manuals/product/f5-plat-diagnostics/_jcr_content/pdfAttach/download/file.res/F5_Platforms__Platform_Diagnostics.pdf)

[K15442: Using the BIG-IP platform diagnostics tool](https://support.f5.com/csp/article/K15442)

EUD SMART test

**# platform_check disk**

**(tmos)# run util platform_check disk**

Logs to:

- **/var/db/platform_check/platform_check.xml**
- **/var/log/platform_check**

Results: PASS, FAIL, or NOT RUN

Not supported in VE

## - 3.02b - Identify resource exhaustion problems based on the log file messages and statistics

### Memory

[Article: K16419 - Overview of BIG-IP memory usage](https://support.f5.com/csp/article/K16419)

[Overview of BIG-IP TMM Memory Usage](https://support.f5.com/csp/article/K16419)

[Understanding the 'top' output on BIG-IP](https://support.f5.com/csp/article/K16739)

[Help! Linux ate my RAM!](https://www.linuxatemyram.com/)

<https://web.archive.org/web/20110207101856/http://www.linuxforums.org/articles/using-top-more-efficiently_89.html>

[K5670: Overview of adaptive connection reaping (11.5.x and earlier)](https://support.f5.com/csp/article/K5670)

**top**

shift + m - Sort by memory usage

\+ Sort descending

\- Sort ascending

Measured in kilobytes

1 million kilobytes = 1 gigabyte

1000 kilobytes = 1 megabyte

nice process - Priority, lower number = more priority
```
top - 15:24:56 up 8 days,  4:52,  3 users,  load average: 0.31, 0.20, 0.08
Tasks: 252 total,   1 running, 251 sleeping,   0 stopped,   0 zombie
Cpu(s):  1.2%us,  0.7%sy,  0.3%ni, 97.4%id,  0.2%wa,  0.0%hi,  0.2%si,  0.0%st
Mem:   4063072k total,  3975468k used,	87604k free,   146572k buffers
Swap:  1023992k total, 	2796k used,  1021196k free,   468544k cached
  
PID USER  	PR  NI  VIRT  RES  SHR S %CPU %MEM	TIME+  COMMAND
10510 root  	RT   0 2430m 121m  95m S  5.6  3.1 686:26.35 tmm
 7438 root  	20   0  487m 104m  28m S  0.7  2.6  68:32.09 mcpd
 7662 root  	25   5 50636 9480 6272 S  0.7  0.2   7:59.31 merged
15757 root  	20   0  2444 1224  852 R  0.7  0.0   0:00.04 top
```

VIRT - How much memory the process can access

RES - How much the process is using in physical memory

Z - Zombie, terminated and waiting for parent process to finish

TIME+ Cumulative amount of time the process has used.

Load average: 1, 5, 15 minute interval. What jobs are queued to run. Over 1 per CPU is bad (4 for quad-core)

Tasks: Processes and states

- Zombie - parent process still around even though child is gone

CPU: Percentage of CPU in different modes - should equal 100 percent total

- us - user
- sy - system kernel processes
- ni - niced user processes
- id - Idle
- %wa - I/O_Wait, high = storage issue
- hi - hardware interrupts
- si - software interrupts
- st - CPU steal time, CPU isn’t available to VM and is working on another VM

Memory: Should show almost always used fully because of TMM, bad indicator

Swap: Rarely used, if its high, memory is high

buffers/cache: How much memory is being borrowed for disk caching - improves access to frequently used things on disk.

Note: A high Other Used memory usage on the BIG-IP system Dashboard may not indicate an issue, as Linux kernel allocates memory to buffers and disk caching that can be released as needed.

You can increase memory to the System (not TMM) with a **modify sys db** command or allocating Small, Medium, or Large to MGMT under the Provisioning GUI page.

[K26427018: Overview of Management provisioning](https://support.f5.com/csp/article/K26427018)

It seems VMs always need to be Large allocated on v14 and above.

##### A possible problem:

Available memory (or "free + buffers/cache") is close to zero

swap used increases or fluctuates

**show sys memory**

System memory

Host = Linux and non TMM processes

TMM

Subsystem - Memory used by TMM objects, hash tables

**show sys tmm-info** -- Shows multiple TMM Instances

TMM Used Memory compared to TMM Available Memory is the most important

TMM cannot be used in swap

Memory used by TMM is cleaned by sweepers periodically

##### Connection Reaping

High-water - 95% TMM kernel utilization. No new connections are allowed

Low-water - 85% TMM kernel utilization. Aggressive reaping of the most idle connections, may delete connections prior to idle timeout configured.

##### Log Entries

011e0002:4: sweeper_update aggressive mode activated - 85% TMM memory utilization will trigger aggressive idle connection deletion.

Adaptive reaping is activated from Dos Attacks, RAM Cache, Memory leaks

**LCD panel** - Blocking DoS Attack

### CPU

[Overview of BIG-IP TMM CPU Usage](https://support.f5.com/csp/article/K15468)

[Error Message: Clock advanced by <number> ticks](https://support.f5.com/csp/article/K10095?sr=24657578)

[K10337613: Idle Enforcer Functionality](https://support.f5.com/csp/article/K10337613)

**show sys proc-info \[ <process-name> ]** - Shows individual processes CPU usage  
**show sys tmm-info** - Show TMM CPU usage per process/core  
**show sys cpu** - System CPU is an aggregation of TMM and Control Plane CPU usage.  
**show sys performance**  
**top -** Load average: 1, 5, 15 minute interval. What jobs are queued to run. Over 1 per CPU is bad (4 for quad-core)  

**High CPU Causes**
- Memory paging, I/O operations
- Complex iRules
- A lot of HTTPS monitors
- Large complex configs (a ton of ssl profiles/certs/keys)
- Admin actions - dumping large number of connections or persistence
- Listing large ARP entries
- Large amount of traffic going through one TMM instance
- Using socks driver instead of vmxnet3

|                                                                                                    |                                                                                                                                                                                                                                                                                                                                     |
| -------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Clock advanced by <number> ticks                                                                | In VE, dedicate more resources or priority{{< line_break >}}The TMM heartbeat thread dead timer hit \[100 ms - physical, 300ms - virtual edition] and how far behind it is from the system clock{{< line_break >}}Can cause traffic disruption or failover if falls behind more than 10 seconds \[10000 milliseconds] - may not log all because action already happened |
| info kernel: ltm driver: Idle enforcer starting - tid: <pid> cpu: <core ID X>/<core ID Y> | CPU usage on the Data plane reaches 80%, the Control plane is paused momentarily and used for Data plane until a log message ‘exited’ is shown                                                                                                                                                                                      |

### Hard Drive

**df -h** - Display partitions and size/usage

**du <directory> -h** - Display size of files, -h = human readable sizes in K or G, automatically recursive into directories!

**lvscan** - Display storage per partition. Some are shared, like log, share, etc.

### Cache

[K13878: The ramcache.maxmemorypercent variable controls memory allocation for the HTTP cache (11.x - 15.x)](https://support.f5.com/csp/article/K13878)

/var/log/ltm

01070668:3: The amount of cache memory assigned to Web Acceleration profiles (840MB) exceeds the maximum amount (838MB) defined by Ramcache.MaxMemoryPercent (50)

ramcache.maxmemorypercent - Percentage amount of memory that the total HTTP cache is allowed to use based on total TMM memory. Default is 50 percent

Verify memory allocated to tmos - **show sys provision**
```
---------------------------------------------------------
Sys::Provision
Module CPU (%) Memory (MB) Host-Memory (MB) Disk (MB)
---------------------------------------------------------
apm 0 0 0 0
asm 0 0 0 0
avr 0 0 0 0
gtm 0 0 0 0
host 10 1296 0 82636
lc 0 0 0 0
ltm 1 0 0 0
pem 0 0 0 0
psm 0 0 0 0
tmos 89 6710 0 0
wam 0 0 0 0
wom 0 0 0 0
woml 0 0 0 0
```

TMM memory is 6710 MB

DB set to 50 percent

4 TMM instances

Formula: 6710 x .50 / 4 - 838MB is the max.

Memory for cache is not allocated until the profile is attached to a VIP. This error would not generate until you apply the profiles

HTTP cache storage is shared among all profiles of the same name. Consolidate profiles when possible

## - 3.02c - Identify connectivity problems based on the log files

**modify sys db** **tm.rstcause.log value enable** - Displays “RST sent from <source> to <destination> and the reason in logs

Everything else is self explanatory or I am not aware of any log entries that indicate a connectivity problem. SSL errors logging was not set to warning in 11.5.

## - 3.02d - Determine the appropriate log file to examine to determine the cause of the problem

|                                                               |                                      |
| ------------------------------------------------------------- | ------------------------------------ |
| **Problem**                                                   | **Log File**                         |
| Failed hard drive or hard drive issues                        | /var/log/kern.log/var/log/daemon.log |
| Hardware initialization                                       | /var/log/dmesg                       |
| Power Supplychmand\[8901]: 012a0040:4: PSU1 Status input lost | /var/log/ltm                         |
| Linux events                                                  | /var/log/messages                    |
| Logins and changes                                            | /var/log/audit                       |

## Objective 3.03 Analyze performance data to identify a resource problem on an LTM device

## - 3.03a - Analyze performance data to identify a resource problem on an LTM device

**Statistics > Performance Reports > Performance Reports**

Performance Graphs are easy to look at. If the RAM or CPU is maxed the graph will say.

**show sys traffic** - Shows packets dropped due to Errors and why

**show net rst-cause** - Cause of resets

## Objective 3.04 Given a scenario, determine the cause of an LTM device failover

### HA Groups (Fast Failover)

[Article: K16947 - F5 recommended practices for the HA group feature](https://support.f5.com/csp/article/K16947)

Used to failover when a trunk link(s) or pool member(s) goes down.

Assigned to a floating traffic group on all devices.

Faster than VLAN, system, or gateway failsafes

Don’t use Force Standby while this feature is enabled on < 13.0 devices, it will fail back to the Active unit. [Disable HA group first](https://support.f5.com/csp/article/K14515)

Don’t use Auto-Failback on <13.0 devices. It will keep failing back to the primary and cause flaps.

**Creating**

System > High Availability > HA Groups

**HA Score Calculation**

HA Score = Weights + Active Bonus  
The maximum overall HA Score is the sum of all HA group Pools and Trunk object weights, plus the active bonus value if the unit is active  
A single Pool or Trunk member object can weigh from 10 - 100  
There is no limit to the total HA Score of all Pool / Trunk object weights for HA group.  
Weight value (Required) 10 - 100, Assigned to pool or trunk object  
Threshold (Optional) - The minimum amount of available objects before making the TOTAL HA score immediately 0. If the threshold is 3, and there’s only 2 trunk links left out of 4, set the score to 0.  
- Beware of pools with priority groups as the members that are up could prevent a failover  
Active Bonus (Optional) - A bonus given to the device with the active traffic group. Used to prevent failover if flapping or minor changes occur to trunk links. If traffic-group is at 0, bonus is not applied  

### HA Order

Auto Failback - Failover to original active if it becomes available again. Recommended 40 to 60 (Default) seconds.

Failover order - Only devices in the same sync-failover group will appear here. Fails back to the first device in order when it becomes available.

If the first device is not available, no auto fallback occurs. If no devices are available, Load Aware is used instead

### Load Aware

BIG-IP Device Service Clustering: Administration pg. 90

At least 3 devices in a device-group are required to use this mode.

**Configure**

Traffic Group > HA Load Factor - Manually specified number to indicate the amount of traffic load under the traffic-group. Default value 10, range 1 - 1000. The higher = more load. Configure on every device, does not sync.

Device > HA Capacity - Manually specified to indicate how much the device can handle related to the other devices. Default is 0 (unlimited) which does not use HA Capacity to calculate a Load Score. The higher = more capacity

**Load Score Calculation**

The load score is calculated on a per device basis in the group. For a device’s score to be calculated, (the sum of all active traffic group’s load factors on that device + the traffic group’s load factors it is next active for / device capacity.)

In order for a next-active device to be chosen for a single traffic group, (the sum of the local active traffic group’s load factors + remote candidate traffic group’s load factor / device capacity) must be run on all devices. The device with the lowest score is chosen as next-active. The tiebreaker is highest management IP

**show cm device** - Displays HA Load Capacity, Mgmt IP, Current Load Factor, Next Active Device Load Factor

### Failsafes

If both units are affected by the same failsafe condition, they can both go into Standby

**Gateway Failsafe**

[K91881543: Configuring gateway fail-safe using the Configuration utility (11.x - 12.x)](https://support.f5.com/csp/article/K91881543)

[K15367: Configuring gateway fail-safe using tmsh (11.x - 13.x)](https://support.f5.com/csp/article/K15367)

1. Create a pool with the gateway IP in it
2. Create new Gateway ICMP monitors to test IP reachability through the gateway with Transparent mode and Alias address, or use the default gateway_icmp monitor
3. **System > High Availability > Fail-safe > Gateway**
4. Specify Pool and which device it belongs to.
5. Specify threshold - 1 would mean if there is less than 1 member, failover.
```
ltm pool gateway_pool {
    members {
        10.128.10.254:0 {
            address 10.128.10.254
        }
    }
    gateway-failsafe-device bigip1.localhost
    min-up-members 1
    min-up-members-action failover
    min-up-members-checking enabled
}
```

**VLAN Fail-safe**

[Overview of VLAN failsafe](https://support.f5.com/csp/article/K13297)

Passively listens for traffic on VLAN. If there is none, generate traffic and expect response before the timer expires, or perform failover action.

Timer half expired - Send ARP request to oldest entry (IPv4)

Timer three-quarter expired - Send ARP request to all entries in table, and ICMP ping to multicast 224.0.0.1

If the peer LB responds in 11 or 12 code, then failover is averted

Be sure that the port has **spanning-tree trunk portfast** enabled. Otherwise, STP convergence can cause very long failovers (90 seconds).

With **portfast** it should be 10 seconds.

**db failover.vlanfailsafe.resettimeronanyframe**

- False (Default) - Only reset timer when receiving response to ARP
- True - Reset timer if any traffic is received - including monitor responses

**Configure**

System > High Availability > Fail-safe > VLANs

Timeout: 90 seconds (default)

Action:  <>
```
net vlan external {
  failsafe enabled
}
```

**Miscellaneous Facts**

- VLAN failsafe configuration is not synced, make changes on all units
- allow-service on Floating Self-IPs needs to be default (untested)
- Set timer larger than it takes for links to come back up (STP)
- Use node monitor for ARP entries to show if you have to
- Do not configure it on the HA VLAN

## - 3.04a - Explain the effect of network failover settings on the LTM device 

Default timeout is 3 seconds

Default port is UDP 1026

Required for connection mirroring

## - 3.04b - Explain the relationship between serial and network failover 

[K2397: Comparison of hardwired failover and network failover features](https://support.f5.com/csp/article/K2397)

Hardware failover precedence over network failover. The unit will not failover if hardware is up and network is down.

## - 3.04c - Differentiate between unicast and multicast network failover modes

Unicast Failover IP - Used for LTMs primarily, and recommended to use on VIPRIONs as a backup.

Multicast Failover - Used by VIPRIONs - Multicast is sent out on the management interface, and all VIPRIONs receive it that are listening on that VLAN.

## - 3.04d - Identify the cause of failover using logs and statistics

[Troubleshooting failover events](https://support.f5.com/csp/article/K95002127#p8)  
[Overview of SSL hardware acceleration fail-safe](https://support.f5.com/csp/article/K16951)  
[How simultaneous failsafe events affect a redundant system](https://support.f5.com/csp/article/K12277)  
[K62511312: Cause of Failover - HA Group](https://support.f5.com/csp/article/K62511312)  
[K57155323: Cause of Failover - Gateway Fail-Safe](https://support.f5.com/csp/article/K57155323)  
[K48546348: Cause of Failover - Auto-failback](https://support.f5.com/csp/article/K48546348)  
[K90354463: Cause of Failover - VLAN failsafe](https://support.f5.com/csp/article/K90354463)  

### Logs

|                                                                                                                                                                                                         |                                                              |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| 010c002b:5: Traffic group <traffic_group> received a targeted failover command for 10.128.1.100                                                                                                      | Force standby command issued                                 |
| 010719fb:5: HA Group <ha-pool-name> score updated from 40 to 30. {{< line_break >}} 010c0045:5: Leaving active, group score 36 peer group score 40. {{< line_break >}} 010c0052:5: Standby for traffic group /Common/<traffic_group_name> | HA Group                                                     |
| 01140029:4: HA pool_memb_down /Common/g_pool fails action is failover                                                                                                                                   | Gateway failsafe                                             |
| 01140029:4: HA vlan_fs                                                                                                                                                                                  | VLAN Failsafe                                                |
| 010c006d:5: Initiating Auto-Failback.                                                                                                                                                                   | Auto-failback                                                |
| 01140029:5: HA daemon_heartbeat tmm fails action is failover and restart{{< line_break >}}01140106:2: Overdog daemon calling bigstart restart.                                                                            | System failsafe                                              |
| 01140043:0: Ha feature nic_failsafe reboot requested                                                                                                                                                    | High Speed Bus issue detected                                |
| 01070417:6: AUDIT - user admin - RAW: Request to run /usr/bin/cmd_sod go standby traffic-group-1 GUI                                                                                                    | Force standby in GUI clicked                                 |
| 00000000:0: go standby                                                                                                                                                                                  | **run sys failover standby** command ran.                    |
| **show sys log tmm \| grep -i SIGSEGV**                                                                                                                                                                 | Daemon failsafe - Search for SIGSEGV segmentation violations |
| **show sys log tmm \| grep -i 'device error'** {{< line_break >}} crypto-failsafe, switchboard-failsafe                                                                                                                     | Hardware failsafe                                            |
| proc-run, ready-for-world                                                                                                                                                                               | System (Daemon) Failsafe                                     |

### Statistics

[K14513346: The tmsh show /sys ha-group command output may display an incorrect HA group score value](https://support.f5.com/csp/article/K14513346)

|                                                                |                                                                                                                                                                                                       |
| -------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **show cm failover-status**                                    | Status ‘Ok’ or ‘Error’ with reason, Network failover IP address and ports of connections (UDP 1026)                                                                                                   |
| **show sys ha-status all-properties** {{< line_break >}} **show sys ha-group** | Failsafe conditions, Failsafe failure status yes/no, timeouts {{< line_break >}} See total HA score including active bonus score on current unit                                                                          |
| **show cm traffic-group all-properties**                       | View HA load factor and next active                                                                                                                                                                   |
| **run cm watch-sys-device**                                    | Live updates of failover status, and HA IPs                                                                                                                                                           |
| **run cm watch-trafficgroup-device**                           | Live failover condition, monitor fault, next active                                                                                                                                                   |
| **show sys failover cable**                                    | cable state unset - Network failover being used, no hardware failover detected - Max cable length - 50 ftcable set 0 - Active unit should display thiscable state 1 - Standby unit should detect this |
| **show sys failover**                                          | Display amount of time in current Active or Standby stateFailover active for 119d 00:43:18                                                                                                            |

## Objective 3.05 Given a scenario, determine the cause of loss of high availability and/or sync failure

## - 3.05a - Explain how the high availability concepts relate to one another

DSC supports same or different hardware

Device Trust - Multiple BIG-IPs establish trust by signing each other's certificates.  
Device Groups - A collection of BIG-IPs that have established trust and wish to Sync-Failover or Sync-Only with each other.  
Traffic Groups - A collection of traffic objects - VIPs, SNATs, NATs, Folders, and Self IPs, that can be transferred to another member in a Sync-Failover Device-Group if there is an issue.  
Folders - Configuration under a partition.  

Connection mirroring can only be done on identical hardware and is mirrored to the next-active device.

## - 3.05b - Explain the relationship between device trust and device groups

A peer BIG-IP must have device trust established before it can be put in a device-group

## - 3.05c - Identify the cause of config sync failures

[Troubleshooting ConfigSync and DSC](https://support.f5.com/csp/article/K13946)

[K65332506: Provisioning mismatch may cause configsync error messages](https://support.f5.com/csp/article/K65332506)

### Config-Sync Requirements

- Device Trust established
- Identical license and provisioned modules
- Same major software release (11.x)
- Times are synchronized
- The self-IPs port-lockdown is **NOT** set to **None,** which is not default in 11.5.0
- **Config must validate/load**

#### Port 4353 Connectivity

Not actually iQuery, it’s separate

1. Local **mcpd** connects locally to **tmm** on 6699
2. Local **tmm** connects to peer over 4353 over SSL
3. Remote **tmm** translates connection on port 4353 to 6699 and passes to **mcpd**
4. Full mesh connectivity between devices over **mcpd** is established
5. Connectivity will show as port 6699 under netstat

### Sync and Device Trust Problems

|                                                                       |                                                                                                                                                                                                                             |
| --------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Verify Commit IDs, CID origin, and CID time. Higher CID = more recent | **run cm watch-devicegroup-device**                                                                                                                                                                                         |
| Processes are up                                                      | **bigstart status devmgmtd mcpd sod tmm**                                                                                                                                                                                   |
| Mismatched provisioned modules                                        | 01260013:4: SSL Handshake failed for TCP <bigip1_configsync_ip>:4353 -> <bigip2_configsync_ip>:<random_port>{{< line_break >}}01260009:4: (null connflow): Connection error: ssl_basic_crypto_cb:694: **alert(20) Decryption error** |
| Verify configuration will load                                        | **load sys config verify**                                                                                                                                                                                                  |
| Sync status and recommended action                                    | **show cm sync-status**                                                                                                                                                                                                     |
| Devices are in the trust group                                        | **show cm device-group device_trust_group**                                                                                                                                                                                 |
| Config-sync IP is configured and trust connectivity is good           | **show cm device** {{< line_break >}} **netstat -pan \| grep -E 6699** - mcpd for device-trust {{< line_break >}} **run cm sniff-updates** - Listens for CMI comms                                                                                              |
| Verify Time                                                           | Verify time/date -  **# date**{{< line_break >}}NTP configured (required?){{< line_break >}}**list sys ntp** {{< line_break >}} **# ntpq -np**                                                                                                                             |
| Verify license, software version, and provisioning                    | **show sys license** {{< line_break >}} **show sys version / software** {{< line_break >}} **list sys provision**                                                                                                                                               |
| Verify port lockdown                                                  | **list net self allow-service**                                                                                                                                                                                             |

#### Sync Status Messages

**Disconnected**

- Communication problem between devices over 4353
- Device trust issue

All else fails - reset device trust

#### Device Trust

Only a CA or Peer can sign certificates. If a peer member is only a Subordinate in its Local Trust, then it cannot sign certificates of the other device. The peer would sign its own certificate for the peer (pretty confusing)

### Self-IPs

Can be a range

Can assign IP that is in the range of separate VLANs to be accessible from both VLANs

#### Port Lockdown

[K13250: Overview of port lockdown behavior (10.x - 11.x)](https://support.f5.com/csp/article/K13250)

Controlling what ports the self-IP can accept traffic on to prevent unauthorized management access, only allow access to other BIP IPs or necessary functions like DNS. Use out of band access instead  (Mgmt)

#### Options

Need to have ports allowed if you want to communicate from them as well. DNS to resolve hostnames from command line, ICMP, etc.

**Allow Default** - Default in 11.0.0 - 11.5.2

- ICMP
- OSPF
- 4353 TCP/UDP (iQuery/CMI)
- 443
- 161 TCP/UDP (SNMP)
- 22 TCP/UDP
- 53 TCP/UDP
- 520 TCP/UDP (RIP)
- 1026 UDP - Network Failover

**Allow All**

**Allow None -** Default in all other versions

- ICMP
- Redundant pairs allow exceptions from peer:
  - 1028 TCP - Persistence mirroring (11.0.0 - 11.3.0)
  - 1029 - 1043 (1055) TCP - Mirroring channels per traffic group (11.4.0+)
  - 4353 TCP - For Syncing via CMI (Centralized Management Infra) (11.0.0+)

#### Verification

Verify port lockdown - **list net self allow-service**

View default - **list net self-allow**

## - 3.05d - Explain the relationship between traffic groups and LTM objects

Traffic groups host these LTM objects for failover - VIPs, SNATs, NATs, Folders, Self IPs

## - 3.05e - Interpret log messages to determine the cause of high availability issues

[K34063049: Error Message: 0107143c:5: Connection to CMI peer <IP address> has been removed](https://support.f5.com/csp/article/K34063049)  
[K06416036: Error Message: 0107143a:5: CMI reconnect timer: <status>](https://support.f5.com/csp/article/K06416036)  
[K13667: BIG-IP device group members must run the same major, minor or maintenance software version](https://support.f5.com/csp/article/K13667)  
[K75975904: Troubleshooting network failover](https://support.f5.com/csp/article/K75975904)  
[K72087447: Error Message: HA Connection with peer \[address:port\] for traffic-group /Common/\[traffic-group_name\] lost. (11.x - 13.x)](https://support.f5.com/csp/article/K72087447)  
[K37457128: Error Message: info sod\[<PID>\]: 010c007b:6: Deleted unicast failover address <IP address> port <port> for device /<partition>/<hostname>.](https://support.f5.com/csp/article/K37457128)  
[K29449945: Error Message: 010c008b:5: Unable to send to unreachable unicast address <IP_address> port <port_number>](https://support.f5.com/csp/article/K29449945)  
[K17362757: Error Message: 010c0083:4: No failover status messages received for <timeout> seconds, from device <device> (<IP_address>)](https://support.f5.com/csp/article/K17362757)  

Usually a **sod** message is a problem with Network Failover

|                                                                                                                                                                                                |                                                                               |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| 01070712:3: Caught configuration exception (0), Can't parse MCP message, can't parse atomic message                                                                                            | Major Software mismatch                                                       |
| 0107143c:5: Connection to CMI peer 192.168.100.2 has been removed{{< line_break >}}0107143a:5: CMI reconnect timer: enabled{{< line_break >}}0107143a:5: CMI reconnect timer: disabled, all peers are connected                    | Device trust port 4353/6699 issues                                            |
| 0107142f:3: Can't connect to CMI peer 10.11.23.140, TMM outbound listener not yet created{{< line_break >}}0107142f:3: Can't connect to CMI peer 192.168.10.100, port:6699, Transport endpoint is not connected  | TMM not initialized on local or remote device                                 |
| 01340007:5: HA Connection with peer 192.168.100.2:32768 for traffic-group /Common/traffic-group-1 closing.                                                                                     | Received force offline message from peer                                      |
| 010c0078:5  Not listening for unicast failover packets on address 192.168.100.2 port 1026{{< line_break >}}010c007b:6: Deleted unicast failover address 192.168.100.2 port 1026 for device /Common/bigip1.f5.com | IP removed from network failover config                                       |
| 010c008b:5: Unable to send to unreachable unicast address 192.168.100.2 port 1026.                                                                                                             | No route found to network failover IP                                         |
| 010c0083:4: No failover status messages received for 3.100 seconds, from device /Common/bigip2.f5.com (192.168.100.2) (unicast: -> 192.168.100.1).                                             | Connection lost over Network failover UDP 1026                                |
| 01340002:3: HA Connection with peer 172.16.1.253:32768 for traffic-group /Common/traffic-group-1 lost.                                                                                         | Connection or persistence mirroring connection lost {{< line_break >}} Check port 1028 - 1043 TCP |
