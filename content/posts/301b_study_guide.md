---
title: "F5 301b Study Guide"
date: 2023-04-30T20:14:22+08:00
---

# Section 1 - Maintain Application and LTM Device Health


# Objective 1.01 Given a scenario, determine the appropriate profile setting modifications


## SSL

[K13385: Overview of the Proxy SSL feature](https://support.f5.com/csp/article/K13385)  

Proxy SSL does NOT terminate SSL on the load balancer. Use it to optimize the SSL connection.  
Use Proxy SSL in an SSL profile to forward a client cert to a server for certification authentication


## Stream Profile

[Overview of the Stream Profile](https://support.f5.com/csp/article/K39394712)  
[Article: K7027 - Replacing multiple strings using a Stream profile](https://support.f5.com/csp/article/K7027)  

“Content rewrite”  
Standard Virtual Server setting  
Replaces string in a data stream (TCP) from the client and server  
Can replace string if it spans across multiple segments by buffering just enough data to do so  
If HTTP profile is applied, search is done in HTTP payload only <span style="text-decoration:underline;">(not headers)</span> in each segment. Otherwise, the whole TCP segment is searched and this is when you can replace HTTP Headers!  

Compatible with HTTP Profile Chunking


### Parameters

Source: What to look for in the content  
Target: What to replace the content with  


### Replace Multiple Sets of Strings

Leave ‘source’ in profile blank

Use a unique delimiter in the ‘target’ field and a space between the source/target pairs.

Don’t put &lt; > unless you want to literally.

**@**&lt;search string1>**@**&lt;replacement string1>**@** **@**&lt;search string2>**@**&lt;replacement string2>**@**

Note: The first character in the field defines the delimiter bounding each field for this replacement and must not appear anywhere else in the target string. In certain versions, the delimiter can be a period (.), asterisk (*), forward slash (/), dash (-), colon (:), underscore (_), question mark (?), equals (=), at (@), comma (,), or ampersand (&) character.

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



### Recommendations



* Restrict rewrites to Content-Type: text*/
* A stream profile must be on VIP before using the STREAM command in an iRule. The default Stream profile may be used with all parameters blank
* Data is changed once per operation
* A compression profile is required to rewrite compressed data 
* Source and Target are case-sensitive unless you use RegEx


## HTTP

[Choosing appropriate profiles for HTTP traffic](https://support.f5.com/csp/article/K4707?sr=46615374)

[Overview of HTTP Chunking](https://support.f5.com/csp/article/K5379)

[K14775: Configuring an HTTP profile to rewrite URLs so that redirects from an HTTP server specify the HTTPS protocol (10.x, 11.x, and 12.x)](https://support.f5.com/csp/article/K14775)

**Drawbacks** - Uses CPU, more memory utilized for compression/caching


### Options



* Request Header Insert
* Request Header Erase
* Insert X-Forwarded-For
* Accept XFF - Distrust or trust XFF header from client to use in statistics for AVR
* Fallback host, Fallback on error codes
* Redirect Rewrite
* Server Header Name
* Cookie encryption
* Protocol enforcement - Header size, count, Pipelining
* Strict Transport Security (HSTS)
    * Protect against stripping attacks by downgrading to HTTP. Force all content in a page to use HTTPS. The F5 inserts a header in the response to the client.


#### Redirect Rewrite Options

The F5 will see an HTTP redirect from a server on a 443 VIP and change it to HTTPS.

Great for SSL offload unaware servers that need to send some redirects.

If the F5 does not do this, one of the following will happen:



* If there’s no HTTP VIP, causes request failure
* Request goes to HTTP VIP and HTTP pool member and causes failure
* Request goes to HTTP VIP and server, which gets redirected to HTTPs and causes redirect loop

<span style="text-decoration:underline;">Matching</span>

Only rewrite courtesy redirects to HTTPS

Courtesy redirects = Sent by server, the F5 adds / to specify a directory if original resource isn’t found.



* http://f5.com/stuff → https://f5.com/stuff**/**

<span style="text-decoration:underline;">Nodes</span> - Change the redirect containing the node’s IP to the VIP

<span style="text-decoration:underline;">All</span> - Rewrites all HTTP 301, 302, 303, 305, or 307 redirects to HTTPS


## HTTP Compression

Profiles > Services > HTTP Compression

Good for WAN clients with slow internet and/or high latency

Reads **Accept-encoding **from client, removes and sends request to server \
Inserts **Content-Encoding** header in response from Server, and specifies clients preferred method, **gzip** or **deflate**

**Compression Levels gzip**

1 - 9, highest is more compression. Anything more than 1 will degrade performance in some way

Depending on load, gzip compression level will automatically go down.


## Web Acceleration (Cache)

Profiles > Services > Web Acceleration



* Must have HTTP profile assigned to VIP
* The server only has to serve content to the BIG-IP once every expiration period for the content
* The cache size in a profile is shared among all VIPs it is applied to.

Types of web acceleration profiles: Basic, Optimized, optimized caching

When to use caching



* High demand objects - server sends data to BIG IP when content expires
* Static content - CSS files, javascript, images/logos, audio.

Items that are cacheable - RFC2616 HTTP/1.1



* 200, 203, 206, 300, 301, 410 responses
* GET methods by default - other methods are supported, even non HTTP methods specified in URI list or iRule
* Based on User-Agent and Accept-Encoding headers
* If the server says it's cacheable, the F5 does it.

Cache-Control Header specifies which content cannot be cacheable



* private, no-store, no-cache - Do not cache
* Set-Cookie - Not cached, as it is usually authentication intended for a particular user session

Cannot cache - HEAD, PUT, DELETE, TRACE, CONNECT by default

Objects that are cached continue to be served even if the pool member is offline, until the object expires. This is by design. Create an iRule if you want this changed.


### Show and Delete Entries

**show ltm profile ramcache **&lt;profile_name>** [ host **&lt;vip_address:port> **max-response **&lt;how_many_entries_to_display> **]**

**delete ltm profile ramcache **&lt;profile_name>** [ host **&lt;vip_address:port> **uri /**&lt;object_name> **]**

**delete ltm profile ramcache **&lt;profile_name>** **- Deletes all entries


## XML

Route to pools, pool members, or VIP based on content in XML file. Specify content in profile and reference in iRule.


## OneConnect



* Re-use server-side connections using Keep-Alive.
* Use an HTTP profile with it or it could do funny things.
* Don’t use on SSL pass-through
* Opens a new connection to the server if needed when the LB algorithm chooses that server. LB algorithm is not overridden
* Load balances each HTTP request individually from the same connection.
* Will close connection after Maximum Reuse or Max Age is exceeded or the server closes it due to Idle timeout
* If SNAT is enabled, the mask is applied to the SNAT IP, which may not work well.

<span style="text-decoration:underline;">OneConnect Transformations</span> - Replaces client's HTTP/1.1 Connection: close with X-Cnection: close to the server. The server will ignore the header. F5 does not send Keep-Alive in HTTP/1.1 as it's the default. The F5 inserts Connection: Keep-Alive for HTTP/1.0 requests. The server must support Connection: Keep-Alive if HTTP/1.0 and HTTP/0.9 is used. A FastHTTP profile will instead replace it with Xonnection: close

<span style="text-decoration:underline;">Source Mask</span> - Specify subnet mask. This is used to specify when to use an idle connection for a client IP coming in



* 0.0.0.0 - Re-use idle connections for all connections regardless of client-IP. Multiple requests from different clients can come over one connection, appearing as if it is from one IP
* 255.255.255.0 - Allow clients with the same first 3 octets to reuse the same connections
* 255.255.255.255 - Only the same IP can re-use an idle connection

<span style="text-decoration:underline;">Maximum Size</span> - How many idle connections to keep connected to servers

<span style="text-decoration:underline;">Maximum Reuse</span> - How many requests to send over one connection. Set to below the max of the server Keep-Alive limit to prevent the server connection to close and entering the TIME_WAIT state

<span style="text-decoration:underline;">Maximum Age</span> - How long to reuse server connection before deleting it

<span style="text-decoration:underline;">Idle Timeout Override</span> - Override idle timeout in TCP profile


#### Troubleshooting



* Fast servers may have fewer open connections and more idle connections than slow servers because they finish connections quicker.
* Persistence can offset if slow servers do not respond quickly enough and more connections go to faster servers.


# - 1.01a - Given a scenario of client or server side buffer issues, packet loss, or congestion, select the appropriate TCP or UDP profile to correct the issue


## TCP Profile

**Proxy Buffer (Content spool)**: 

The F5 buffers data from the server if the client has not acknowledged data quick enough. This allows the server to focus on other requests.

(Airport analogy - Bags are offloaded to the metal rollers to allow people to rest and do other things.) 

May want to increase this with clients that have packet loss or are latent. Keeping both the same and high is optimal so data is buffered as soon as the client ACKs content in the send buffer.

<span style="text-decoration:underline;">Proxy Buffer High:</span> The amount of data in the buffer in which to close the receive window from the server.

<span style="text-decoration:underline;">Proxy Buffer Low:</span> Amount of data in the buffer that triggers the receive window to open.

(The amount of space free on the metal rollers to start accepting more bags.- 3 feet)

<span style="text-decoration:underline;">Send Buffer**: **</span>Data that was sent but not yet acknowledged - kept just in case it needs retransmitted.

<span style="text-decoration:underline;">Receive Window</span>: How much to send the F5 that can be outstanding/unacknowledged

<span style="text-decoration:underline;">Congestion Control**: **</span>Use Woodside if some clients are wireless 

**tcp-mobile-optimized**



* Buffers and window increased to allow for latency/packet loss
* Initial Congestion Window Size: 16: Sends more data unacknowledged to begin with
* Nagle - Enabled: Needing ACKs of full segment size to send more when data is outstanding. Fewer packets on latent slow networks.

**tcp-wan-optimized**



* Increased buffers
* Send Buffer and Receive window the same as mobile-optimized
* SACK and Nagle Enabled

**tcp-lan-optimized**



* Buffers, window the same as wan-optimized
* Slow Start: Disabled
* Nagle - Disabled: Nagle may hold data, when its not needed
* Acknowledge on Push: Enabled: Helpful for Windows and MacOS with small send buffers.


## TCP

**Stream-oriented** - TCP will send a continuous stream of data and divide up packets to fit into IP datagrams.

TCP Window Size, AKA Send Window:

_Send Window**:**_ Amount of data that can be sent and be outstanding/unacknowledged. A 16-bit field (2^16 = 65535)



* This can be increased by using the scale factor in the Options field during the handshake, value 0 to 14.
* Take the unscaled Window, multiply it by 2^(0-14)
* Example - 65535 * (2^14) = 1,073,725,440 bytes (a gigabyte)

Maximum Segment Size (MSS) - The maximum segment size the local host _will _accept. It usually is 40 bytes less than the MTU size. 

Default is 536 bytes, 40 bytes less than default IP MTU


# - 1.01b - Given a scenario determine when an application would benefit from HTTP Compression and/or Web Acceleration profile

Compression - Good for WAN clients with low bandwidth or latency

Caching



* High demand objects - server sends data to BIG IP when content expires
* Static content - CSS files, javascript, images, audio.


# Objective 1.02 Given a subset of an LTM configuration, determine which objects to remove or consolidate to simplify the LTM configuration


## Virtual Server


### Matching Order

Most Specific > Least specific 



* Destination
* Source
* Port

VIP listener has precedence over NAT

Types



* Standard
* Forwarding (Layer 2) - No pools
* Forwarding (IP) - No pools
* Performance (HTTP)
* Performance (Layer 4)
* Stateless
* Reject
* DHCP Relay

**Address translation** - Send to different IP (Pool member) Default enabled. Disable if the pool member has the same IP as the VIP

**Port** **translation** - Automatically enabled on a Standard VIP. Connections to VS:80 -> Server:8909. Disable it if you have a multi-port VIP and want traffic sent to pool members that are listening on several ports

**Auto Last Hop** - Send return traffic from the pool of servers to the MAC address that sent the client request. This is useful if there is not a default route on the F5 and there is no route for the source. Also works to reply return traffic through transparent devices like an IPS.

**Last Hop Pool** - Send return traffic through this pool

**Clone Pool (Client)** - Specify a pool that all client traffic will get cloned to. The service port is irrelevant in the pool config.

**Clone Pool (Server)**

**Rewrite Profile** - Content Rewrite profile for HTML web application rewrites [K99872325: Modifying HTML tag attributes using an HTML profile](https://support.f5.com/csp/article/K99872325)


# - 1.02a - Evaluate which iRules can be replaced with a profile or policy setting

Use built in things over iRules.

LTM Policies override iRules

HTTP Profile


# - 1.02b - Evaluate which host virtual servers would be better consolidated into a network virtual server

If you wanted to direct clients destined to a network to a pool of firewalls.


# Objective 1.03 Given a set of LTM device statistics, determine which objects to remove or consolidate to simplify the LTM configuration


# - 1.03a - Identify redundant and/or unused objects

HTTP profile on VIP without iRule or HTTP inspect/modification requirements.


# - 1.03b - Identify unnecessary monitoring

ICMP and TCP is unnecessary

Same monitors applied to the pool member and the pool


# - 1.03c - Interpret configuration and performance statistics

**show ltm pool**

**show ltm virtual**


# - 1.03d - Explain the effect of removing functions from the LTM device configuration

SSL profiles and certificates



* Note: Existing connections continue to use the old SSL certificate until the connections complete or are renegotiated or until TMM is restarted.
* Impact of procedure: Performing the following procedure should not have any impact on the existing traffic and new traffic will utilize the new certificate.

HTTP profiles

SNAT


## Removing a Pool member from a pool

Removes existing persistence entries. 

If the cookie pool member is not available, the F5 makes a new load balancing decision.

Existing connections remain until idle timeout or client close.


# Objective 1.04 Given a scenario, determine the appropriate upgrade and recovery steps required to restore functionality to LTM devices


## Increase disk space

[K14952: Extending disk space on BIG-IP VE](https://support.f5.com/csp/article/K14952)

Supported directories that can be increased:



* /var
* /var/log
* /config
* /shared

Verify free space -** list sys disk**

Increase space - **modify sys disk directory /var new-size** &lt;desired value in KB>

Verify - **show sys disk directory**

**reboot**


## Console

Bits - 19200 \
Data bits - 8 \
Parity - N \
Stop bit - 1 \
Flow control - N


## Management IP Config

GUI:  System -> Platform

TMSH: **modify sys management-**


## GUI Issues

Check or restart **httpd** and **tomcat**

**bigstart restart httpd tomcat**


## Daemons

**restart sys service &lt;name>**

The **mcpd** daemon is the Master Control Program. Allows two-way communication between userland (applications outside the kernel) processes and TMM processes. 

If this is down, traffic management does not function and nothing can be configured. Other daemons will also not function correctly.

MCP is responsible for health checks

**show sys mcp-state**



* --------------------------------------------------------
* Sys::mcpd State:
* --------------------------------------------------------
* Running Phase                            running
* Last Configuration Load Status  full-config-load-succeed
* End Platform ID received:           true


## MOS (Maintenance Operating System)

[Article: K14245 - Overview of the recovery tasks performed from the MOS (11.x - 15.x)](https://support.f5.com/csp/article/K14245)

Reboot into it from CLI: **mosreboot**

Boot into it using Console:

Select TMOS Maintenance from grub boot menu:

   GNU GRUB  version 0.97  (619K LOWER / 3653927K UPPER MEMORY)

 ******************************************************************

 * BIG-IP 12.1.3.7 Build 0.0.2 - drive sda.1                      *

 * TMOS maintenance                                               *

 *                                                                *

 ******************************************************************

You will be logged in as root automatically

Capabilities:



* Mount filesystems
* Use **e2fsck **to repair file system
* Set mgmt IP
* Transfer files
* Install IOS using **image2disk**


## e2fsck

For ext2/ext3/ext4 filesystems


### Error on boot about HDD


```
[/sbin/fsck.ext3 (1) -- /shared] fsck.ext3 -a /dev/mapper/vg--db--sda-dat.share.1 dat.share.1 contains a file system with errors, check forced.
 Error reading block 5259469 (Attempt to read block from filesystem resulted in short read) while reading indirect blocks of inode 2622816.
```


**e2fsck -yf** /dev/mapper/vg--db--sda-dat.share.1

-y : Assume yes to all questions

-f : Force check even if it seems clean


## fsck

[Article: K73827442 - Forcing a file system check on the next system reboot (12.x - 15.x)](https://support.f5.com/csp/article/K73827442)

[Article: K60432403 - Recovering from a failed device start due to file system errors](https://support.f5.com/csp/article/K60432403)

[F5 upgrade error DevCentral](https://www.devcentral.f5.com/s/question/0D51T00006wzaNH/f5-upgrade-error)

[K61521075: BIG-IP VE failed to boot after outage with console message: UNEXPECTED INCONSISTENCY; RUN fsck MANUALLY.](https://support.f5.com/csp/article/K61521075) - May not be on test


### Run on next reboot

**touch /forcefsck**

**reboot**

OR

**shutdown -rF now**


### Error on boot about HDD


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

*login*

**fsck -y **Automatically attempt to fix any filesystem corruption errors automagically.


## Replace down device

Only a couple steps that might be on the test:

Remove serial cable if necessary or disable it - **modify sys db failover.usetty01 value disable**

Update Master Key on the new device to what the old device had, so that the UCS loads



* Retrieve key from failed device - **f5mku -K**
* Replace master key on new unit **f5mku -r &lt;key>**


## Reload Configuration

[Article: K13030 - Forcing the mcpd process to reload the BIG-IP configuration](https://support.f5.com/csp/article/K13030)

Binary file that is loaded on boot mismatches text file config (bigip.conf)

**# touch /service/mcpd/forceload**

**# reboot**

Logs after reboot


```
The configuration DB has been initialized with 26130560 bytes of memory
Configuration restored from binary image.
```



## Single Configuration File

Single Configuration File (SCF) - Used to configure everything, in one file

bigip.conf - Configuration objects, VIPs, pools, profiles, iRules, authentication settings. Partitions have a separate ‘partitions’ folder at the root in the UCS file that have a bigip.conf in them.

bigip_base.conf - Base level configuration like VLANs, Self-IPs, Management IP settings, DSC settings, provisioning.

bigip_user.conf - User account config

bigip_script.conf - Custom iApp templates

SCF files are intended to help configure additional BIG-IP systems; SCFs are not intended to back up and restore a full BIG-IP system configuration or to restore a BIG-IP configuration to a later BIG-IP version.



* You want to replicate a configuration across multiple BIG-IP systems.
* You want to migrate a configuration from one platform type to another (for example, from BIG-IP VE to a hardware platform).


### Saving an SCF file

Save the running configuration to an SCF



* **save sys config file** <SCF_filename> **[passphrase <password>]**

Specify a custom name for the TAR file when saving the configuration to an unencrypted SCF:



* **save sys config file** &lt;SCF_filename> **tar-file** &lt;TAR-FILE_filename>

Note: When you use the tar-file option, the actual file does not have .tar file extension by default, unless specified. Additionally, you cannot specify the passphrase option when you use the tar-file option.

The SCF file and the TAR file are saved to the **/var/local/scf/** directory.

Files referenced [keys, external monitor files and external data group files] in config are saved in **/var/local/scf/&lt;name>.tar**


### Loading a SCF

When loading a SCF file, the current config is backed up in **/var/local/scf/backup.scf**

You cannot load an SCF onto another software version

To install an SCF:



**load sys config file** <SCF_filename> **[passphrase <passphrase>]**

To load the unencrypted configuration with a TAR file that has a custom name:



* **load sys config file **&lt;SCF_filename>** tar-file **&lt;TAR-FILE_filename>


### Diff SCF files

**show sys config-diff** &lt;SCF_filename> [SCF_filename_or_blank_for_running_config]


## UCS Contents & Info



* BIG-IP Config files
* Product licenses. New system should have a base license already. Use the -no-license flag. Or contact F5 to associate license with new F5 system
* SSL certs/keys
* User/passwords
* DNS zone files
* Allows restoring config to newer versions (10.x to 11.x)
* Hardware must be the same or do **no-platform-check**
* Restoring configuration for an unlicensed module will error.
* You can restore a configuration without the master key matching, but it won’t decrypt encrypted passphrases. 

root@(bigip1)(cfg-sync Standalone)(Active)(/Common)(tmos)# **load sys ucs test.ucs** ?

Options:

 ** no-license**         This option mostly is for RMA use. It loads full configuration from a UCS file except the license file.

  **no-platform-check**  Bypass platform check.

  **passphrase **        Passphrase for (un)encrypting UCS.

  **platform-migrate**   Don't load or modify some objects specific to a particular device.

  **reset-trust**        Reset device and trust domain certificates and keys when loading a UCS.


### Restore without passphrase

**0107102b:3: Master Key decrypt failure - decrypt failure - final**

Restore the UCS and then replace the encrypted passphrases with the unencrypted ones in the bigip.conf file


# - 1.04a - Identify the appropriate methods for a clean install


## Clean Install

All mass-storage devices are wiped. Useful if the device no longer boots from any volumes. 


### Bootable USB

Must be done on a Linux workstation

4G or larger USB

Dependencies



* Linux 2.6.x kernel
* Perl 5.8 or later
* **cpio** - Manages CPIO archive files
* **mke2fs **- Creates Linux ext2 file system
* **extlinux** - Lightweight bootloader that starts computers with the Linux kernel

Unmount, plug into BIG-IP and reboot: **umount /mnt/&lt;directory>**

You will be in MOS and prompted to perform an **automatic** **clean** **install**, just hit **[Enter]**


### USB DVD-ROM Drive - Recommended

Burn .iso image file to DVD.

You will be in MOS and prompted to perform an **automatic** **clean** **install**, just hit **[Enter]**


### PXE Boot

Can be used to install when physical access is not available and to multiple systems simultaneously. 

The server providing the image must be on the same network as the management port and provide HTTP, TFTP and DHCP services. 


# - 1.04b - Identify the TMSH sys software install options required to install a new version

**install sys software** { **hotfix** | **image** } &lt;filename> (**create-volume**)**  volume** &lt;HDx.x>


# - 1.04c - Identify the steps required to upgrade the LTM device such as: license renewal, validation of upgrade path, review release notes, etc.


## Upgrade Code


### Preparation

Make sure there’s enough space on the hard drive. Use “df -h” - 50GB required


### Download code



* **/shared/images**
* GUI - System > Software Management > Image List > Import...


### Verify image



* **md5sum** /shared/images/&lt;name>.iso


### List available images for installation

**list sys software image**

System > Software Management > Image | Hotfix List


### Install to inactive volume

**install sys software** { **hotfix** | **image** } &lt;iso> [**create-volume**]**  volume** &lt;HDx.x>

**GUI - **System > Software Management > Image | Hotfix List > Install > HDx.x

Delete volume: **delete sys software volume** HD1.&lt;x>


### Reactivate the license (Traffic impacting)

If you fail to do this, Reboot into the old volume and reactivate.

Terms



* **License Check date**: A date specific to each software release - check online F5 solutions article
* **Service Check date: **Date from the last time activated or when the service contract ends, whichever is earlier.

If the service check date is missing or is earlier than the license check date, then the system initializes, but does not load the configuration. To allow the configuration to load, you must update the service check date in the license file by reactivating the system's license.

**TMSH**

Determine if re-activation is needed: **grep “Service check date” /config/bigip.license**. This should be a later date that the license check date of the software you’re upgrading to

**GUI**

System > License > Re-activate… > Automatic selected > Next

01070424:5: Full configuration initialization phase triggered.

01070427:5: Initialization complete. The MCP is up and running


### Confirm Current Config will Load Upon Reboot

load sys config verify


### Copy configuration to new volume and reboot into it

TMSH



* **cpcfg --source=HD**x.x **HD**x.x
* **reboot volume HD**x.x - OR - **switchboot**

GUI - System > Software Management > Boot Locations > HDx.x > Install Configuration [yes] > Activate


# - 1.04d - Identify how to copy a config to a previously installed boot location/slot

**cpcfg --source=HD**x.x **HD**x.x

GUI - System > Software Management > Boot Locations > HDx.x >** Install Configuration [yes] **> Activate


# - 1.04e - Identify valid rollback steps for a given upgrade scenario

Reboot into the old volume



* **reboot volume HD**x.x - OR - **switchboot**


# Objective 1.05 Given a scenario, determine the appropriate upgrade steps required to minimize application outages


# - 1.05a - Explain how to upgrade an LTM device from the GUI

See above


# - 1.05b - Describe the effect of performing an upgrade in an environment with device groups and traffic groups

Move traffic groups to the active unit before upgrading the standby


# - 1.05c - Explain how to perform an upgrade in a high availability group

Upgrade secondary / standby first

Failover to secondary - run sys failover standby

Upgrade primary / standby


# Objective 1.06 Describe the benefits of custom alerting within an LTM environment


# - 1.06a - Describe how to specify the OIDs for alerting

alert &lt;name> “&lt;message>” { \
snmptrap OID=”.1.3.6.1.4.1.3375.2.4.0.&lt;**300 - 999>**” \
}


# - 1.06b - Explain how to log different levels of local traffic message logs

TMSH - **modify sys db &lt;name>.level value &lt;log-level-string>**

GUI - System > Logs > Configuration > Options

The severity level is in between the colons

Jul 22 07:38:28 nusiaalbipve02 mcpd[1844]: 01070638:5: Pool member 10.0.0.154:80 monitor status forced down.


### Modify Linux host syslog levels



* **list sys syslog all-properties**
* **modify sys syslog **&lt;function>**-from **&lt;function>-**to **&lt;level>
* **modify sys syslog daemon-from warning daemon-to emerg**


# - 1.06c - Explain how to trigger custom alerts for testing purposes


## Built-in Alerts



1. Find trap in **/etc/alertd/alert.conf**
2. Grep for that alert for the full logging info- 
    1. **cd /etc/alertd/**
    2. **grep &lt;alert name> *.h**
3. Find Log Info
    3. 0 LOG_NOTICE 01070640 BIGIP_MCPD_MCPDERR_NODE_ADDRESS_MON_STATUS "Node %s address %s monitor status %s."
    4. Facility 0
    5. Log Level: Notice
    6. Alert code: 01070640
    7. Descriptive Message: "Node %s address %s monitor status %s."

Use **logger **to generate an alert -** logger -p local0.notice “**01070640:5: Node 10.128.20.20 monitor status down.”


# Objective 1.07 Describe how to set up custom alerting for an LTM device


# - 1.07a - List and describe custom alerts: SNMP, email and Remote Syslog


## Configure Syslog

System > Logs > Configuration > Remote Logging


## SNMP

Place custom MIB files in **/config/snmp/&lt;name>.tcl**

Default MIBs under **/usr/share/snmp/mibs**


### Traps

Default SNMP Traps - **/etc/alertd/alert.conf **(Don’t add or remove traps from here) \
User-defined SNMP Traps - **/config/user_alert.conf**

System > SNMP > Traps > Destination

System > SNMP > Traps > Configuration


### Custom Traps



1. Create backup of **/config/user_alert.conf**
2. Open and add new SNMP traps in proper format

     \
alert &lt;name> “&lt;message>” { \
snmptrap OID=”.1.3.6.1.4.1.3375.2.4.0.&lt;**300 - 999**>” \
}

3. <span style="text-decoration:underline;">Message - </span> A syslog message that will match and send the trap. Can use literal or regular expressions. For regex, put parentheses around the expression - (.*) for example.
4. <span style="text-decoration:underline;">300 - 999</span> - A unique OID, don’t use ones that are in F5-BIGIP-COMMON-MIB.txt file
5. The **user_alert.conf **file is appended to the **alertd.conf **file upon service start, so may need to restart **snmpd **service to take effect


## Email SNMP traps

**ssmtp **is the process for emailing

**modify sys outbound-smtp mailhub **&lt;mail-server:port>

**/config/user_alert.conf**


    alert &lt;name> “&lt;message>” { \
snmptrap OID=”.1.3.6.1.4.1.3375.2.4.0.&lt;**300 - 999**>”,

	email toaddress="dg-network@config.com, noc@config.com" \
	fromaddress="root" ←- The FQDN will be the hostname \
	body="There’s an alert!"

	}

Save file and restore permissions -  **chmod 444**

Restart alertd and snmpd - **restart sys service alertd, restart sys service snmpd**

**Test Email**

**echo “**ssmtp test mail**” | mail -vs “**Test email for SOL13180**” **myemail@mydomain.com


# - 1.07b - Identify the location of custom alert configuration files

/config/user_alert.conf


# - 1.07c - Identify the available levels for local traffic logging

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


# Section 2 - Identify and Resolve Application Issues


# Objective 2.01 Determine which iRule to use to resolve an application issue


# - 2.01a - Determine which iRule events and commands to use

CLIENT_ACCEPTED - Connection entry added 3-way handshake completes for Standard VIP, initial SYN for Performance L4.

LB_SELECTED - Pool member selected.

HTTP_REQUEST - Client request headers

HTTP_RESPONSE - Server response


# - 2.01b - Given a specific iRule event determine what commands are available


## Events


### LB_SELECTED

active_members

active_members -list: Determine how many members are currently **available** in a pool by returning IP and port numbers.


## Operators

Relational: contains, equals, starts_with

Logical: and, if, else


## Commands

Statement: **pool, node, virtual**

Query: **IP::remote_addr, IP::addr, HTTP::host**

Data manipulation: **HTTP::header remove**

Utility: **URI::decode**

HTTP::method

HTTP::request


### HTTP::header

HTTP::header value &lt;header-name> - Returns value from header name

HTTP::header values &lt;header-name> - Returns list of values from header name.

HTTP::header names

HTTP::header exists &lt;header-name>

HTTP::header insert ["lws"] [[list] &lt;header1-name> &lt;header1-value> &lt;header2-name> &lt;header2-value> ] - ‘“lws”’, the system adds linear white space to long header values.

HTTP::header is_redirect

HTTP::header replace &lt;header-name> [&lt;new-header-value>]

HTTP::header remove &lt;header-name>


### HTTP::uri

Full URI including query


### HTTP::path

HTTP_REQUEST, SERVER_CONNECTED & more

/main/index.jsp

Does not include the query (?test.xml)


### HTTP::query


# Objective 2.02 Explain the functionality of a simple iRule

iRules are processed in order in which they appear in the GUI and CMD

Events are processed in a linear fashion so all CLIENT_ACCEPTED events in all iRules finish processing before the first HTTP_REQUEST event can fire.

Once one Event in all iRules is processed, action is taken if matched

HTTP::redirect takes precedence over pool command

**&lt;Insert iRules here that direct to pools or nodes based on URI, host header, or IP address that even do snat as well. Use switch, if, elseif, starts_with, contains>**


# - 2.02a - Interpret information in iRule logs to determine the iRule and iRule events where they occurred

Logging in an iRule

log local0.&lt;level-optional> “log here!”

Levels



* emerg
* alert
* crit
* error
* warn
* notice
* info
* debug


# - 2.02b - Describe the results of iRule errors

Error Message: 01220001:3: TCL error



* Stops iRule processing and may send a TCP RST
* Can happen if the wrong variable is unset at the end


# Objective 2.03 Given specific traffic and configuration containing a simple iRule determine the result of the iRule on the traffic

[event](https://clouddocs.f5.com/api/irules/event.html)

[switch](https://clouddocs.f5.com/api/irules/switch.html)

[return](https://clouddocs.f5.com/api/irules/return.html)

[class](https://clouddocs.f5.com/api/irules/class.html)

The * in front of a URL will make it so only the URL with stuff in front of that asterisk will match. If there is nothing in front, then that entry won’t be matched. Example, *default.com, make sure to add “default.com” to the rule. 

An asterisk afterwards works as ‘contains’

glob - Allows the use of limited regex on URL/URI’s (* or  ?) that is more performant

The ‘-‘ hyphen is an OR statement

A “;” semicolon will let you have two commands on one line



* Will it only work if there’s one event in the iRule?

return - Exit from current event in the _current _iRule. 

event disable [all] - Stop going through the current event in all iRules, Stop going through all iRule events on this connection

-- = Stop option processing - useful if the item might start with a hyphen. Put before ‘-value’

“” = Blank value

The "switch -glob" and "string match" commands use a "glob style" matching that is a small subset of regular expressions, but allows for wildcards and sets of strings. 

Make use of the "equals", "contains", "starts_with", and "ends_with" iRule operators, or the glob matching mentioned above.  They perform significantly faster, and do the exact same thing as regex. Regular expressions are very CPU intensive and should only be used when there are no other options

Use ‘switch’ instead of multiple ‘if’ lines, unless you need more than one condition.

switch “-exact” is the default if no arguments are specified

switch -glob - Supports small subset of regex (same as ‘string match’)

The switch section can contain a ‘default’ action labeled as such, like forwarding to a pool or dropping the packet if there are no matches.

**string match** ?-nocase? _pattern string_



* See if pattern matches string; return 1 if it does, 0 if it does not. If -nocase is specified, then the pattern attempts to match against the string in a case insensitive manner. For the two strings to match, their contents must be identical except that the following special sequences may appear in pattern:

* - Matches any sequence of characters in string, including a null string.



- Matches any single character in string


<p id="gdcalert1" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: Definition term(s) &uarr;&uarr; missing definition? </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert2">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>






class - Allows querying of data-groups

class match - Sees if item matches in the data-group


### Example iRules

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


# - 2.03a - Use an iRule to resolve application issues related to traffic steering and/or application data


## Insecure Content on Page

Webpages/HTML responses contain links to http pages. You can use a Stream profile or/and iRule to replace http with https in the text of the HTML response page.

Do this in HTTP_RESPONSE

[https://support.f5.com/csp/article/K31100432#irule](https://support.f5.com/csp/article/K31100432#irule)

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


## Allow Internet Explorer to download files through SSL

[Article: K8093 - Using iRules to modify Cache-control headers to allow Internet Explorer to download files through SSL](https://support.f5.com/csp/article/K8093)

These headers are used to prevent proxy devices from caching a local copy of the file when handling connections.



* Cache-control:no-store, Cache-control:no-cache, or Pragma:no-cache

Use this to fix the problem and allow the files to be downloaded:

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


# Objective 2.04 Interpret AVR information to identify performance issues or application attacks


# - 2.04a - Explain how to modify profile settings using information from the AVR 

Maybe you could increase the buffers on the TCP profile


# - 2.04b - Explain how to use advanced filters to narrow output data from AVR 

Statistics > Analytics > HTTP > Custom Page > Add Widget

**Capture Options**

Choose what to capture



* Request and/or response that includes: Headers, Body, or All

Choose capture filters



* Virtual Server
* Node
* Protocols
* HTTP Response Codes
* HTTP Methods
* URL
* User Agent
* Client IP
* Requests and/or Responses strings


# - 2.04c - Identify potential latency increases within an application

See if the latency goes up


# Objective 2.05 Interpret AVR information to identify LTM device misconfiguration

How can AVR project misconfiguration?


# - 2.05a - Explain how to use AVR to trace application traffic 

Statistics > Analytics > HTTP > Overview

Page Load Time analytics works on browsers that meet the following requirements:



* Navigation Timing by W3C
* Accepts cookies from visited site
* JavaScript enabled for site

**The following can only be changed in the default ‘analytics’ profile**

Transaction Sampling (Default profile)



* Sampling improves system performance. 
* F5 recommends that you enable sampling if you generally use more than 50 percent of the system CPU resources, or if you have at least 100 transactions in 5 minutes for each entity.
* Cannot Capture traffic if Enabled.

**View Captures** - System > Logs > Captured Transactions. Limited to 1000 entries

**Notification Type**



* Syslog (System > Logs > Local Traffic)
* SNMP - Sys > SNMP > Traps > Destination (Auto sets up syslog)
* E-mail - SMTP under analytics profile


# - 2.05b - Explain how latency trends identify application tier bottlenecks

If the latency gets higher, see if pool members have been failing which results in others taking more load.

If there isn’t SSL offloading, try it to reduce latency/server load.


# Objective 2.06 Given a set of headers or traces, determine the root cause of an HTTP/HTTPS application problem

**HTTP/0.9**



* GET only
* One request per connection

**HTTP/1.0**



* Introduced media
* One request per connection
* Backwards compatible with HTTP/0.9
* Only 1 domain per IP, cannot specify Host

**HTTP/1.1**



* Backwards compatible with HTTP/0.9 and 1.0
* Host header required

Improvements (Mostly performance)



* **Multiple Hostname Support**: Multiple domains per IP via Host header (required)
* **Persistent Connections**: Multiple requests per TCP connection
* **Cache and Proxy Support**


# - 2.06a - Explain how to interpret response codes


<table>
  <tr>
   <td><strong>Status Code Format</strong>
   </td>
   <td><strong>Meaning</strong>
   </td>
   <td><strong>Description</strong>
   </td>
  </tr>
  <tr>
   <td><strong>1xx</strong>
   </td>
   <td><strong>Informational Message</strong>
   </td>
   <td>Provides general information; does not indicate success or failure of a request.
   </td>
  </tr>
  <tr>
   <td><strong>2xx</strong>
   </td>
   <td><strong>Success</strong>
   </td>
   <td>The method was received, understood and accepted by the server.
   </td>
  </tr>
  <tr>
   <td><strong>3xx</strong>
   </td>
   <td><strong>Redirection</strong>
   </td>
   <td>The request did not fail outright, but additional action is needed before it can be successfully completed.
   </td>
  </tr>
  <tr>
   <td><strong>4xx</strong>
   </td>
   <td><strong>Client Error</strong>
   </td>
   <td>The request was invalid, contains bad syntax or could not be completed for some other reason that the server believes was the client's fault.
   </td>
  </tr>
  <tr>
   <td><strong>5xx</strong>
   </td>
   <td><strong>Server Error</strong>
   </td>
   <td>The request was valid but the server was unable to complete it due to a problem of its own.
   </td>
  </tr>
</table>


If the code received is not understood by the client, like 491 for example, a 400 code is displayed. The x00 codes are generic.

400 - Bad Request - Missing Host header or bad request

401 - Method Not Allowed


<table>
  <tr>
   <td><strong>500</strong>
   </td>
   <td>Internal Server Error
   </td>
   <td>Generic error message indicating that the request could not be fulfilled due to a server problem.
   </td>
  </tr>
  <tr>
   <td><strong>501</strong>
   </td>
   <td>Not Implemented
   </td>
   <td>The server does not know how to carry out the request, so it cannot satisfy it.
   </td>
  </tr>
  <tr>
   <td><strong>502</strong>
   </td>
   <td>Bad Gateway
   </td>
   <td>The server, while acting as a gateway or proxy, received an invalid response from another server it tried to access on the client's behalf.
   </td>
  </tr>
  <tr>
   <td><strong>503</strong>
   </td>
   <td>Service Unavailable
   </td>
   <td>The server is temporarily unable to fulfill the request for internal reasons. This is often returned when a server is overloaded or down for maintenance.
   </td>
  </tr>
  <tr>
   <td><strong>504</strong>
   </td>
   <td>Gateway Timeout
   </td>
   <td>The server, while acting as a gateway or proxy, timed out while waiting for a response from another server it tried to access on the client's behalf.
   </td>
  </tr>
</table>



# - 2.06b - Explain the function of HTTP headers within different HTTP applications (Cookies, Cache Control, Vary, Content Type & Host) / - 2.07c - Predict the browser caching behavior when application data is received (headers and HTML)

**_Cache-Control - _**Directions to cache or not, a response or request.



* This overrides any default caching control for a device
* Pragma header is the HTTP/1.0 equivalent

<table>