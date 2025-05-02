---
title: "F5 301b v1 Objectives"
date: 2023-04-26T20:14:22+08:00
---

As far as I know these are the original objectives first published for the exam.  
These are no longer official. They were found in the old study guide which can no longer be found.   
{{< line_break >}}


\=================================================================  
Section 1 – Troubleshoot Basic Virtual Server Connectivity Issues  
\=================================================================

Objective - 1.01 - Given a scenario, determine the appropriate profile setting modifications
- 1.01a – Determine the effect of changing profile settings on application traffic
- 1.01b – Determine if changing profile settings will affect the LTM device or the application
- 1.01c – Explain the effect of modifying buffer settings in the TCP profile
- 1.01d – Explain the effect of modifying timeout settings in the TCP/UDP profile
- 1.01e – Determine the effect of TCP/UDP profile interaction with other profiles (e.g. OneConnect, HTTP, FTP, etc. For example, how modifying settings in TCP profile can break FTP)
- 1.01f – Describe how the source of traffic affects TCP/UDP profile settings that should be selected
- 1.01g – Determine the appropriate settings to use based on a given application behavior
- 1.01h – Determine the appropriate netmask settings when using a OneConnect profile

Objective - 1.02 - Given a subset of an LTM configuration, determine which objects to remove or consolidate to simplify the LTM configuration
- 1.02a - Interpret configuration
- 1.02b - Identify redundant application functions
- 1.02c - Describe how to consolidate redundant application functions
- 1.02d - Determine the appropriate redundant function to remove/keep
- 1.02e - Explain the effect of removing functions from the LTM device configuration

Objective - 1.03 - Given a set of LTM device statistics, determine which objects to remove or consolidate to simplify the LTM configuration
- 1.03a - Interpret performance statistics
- 1.03b - Identify redundant application functions
- 1.03c - Describe how to consolidate redundant application functions
- 1.03d - Determine the appropriate redundant function to remove/keep
- 1.03e - Explain the effect of removing functions from the LTM device configuration

Objective - 1.04 - Given a scenario, determine the appropriate upgrade and recovery steps required to restore functionality to LTM devices
- 1.04a - Explain how to upgrade a vCMP environment
- 1.04b - Explain how to upgrade an LTM device from the GUI
- 1.04c - Describe the effect of performing an upgrade in an environment with device groups and traffic groups
- 1.04d - Explain how to perform an upgrade in a high availability group
- 1.04e - Describe the process for using SCF and UCS files

Objective - 1.05 - Given a scenario, determine the appropriate upgrade steps required to minimize application outages
- 1.05a - Explain how to upgrade a vCMP environment
- 1.05b - Explain how to upgrade an LTM device from the GUI
- 1.05c - Describe the effect of performing an upgrade in an environment with device groups and traffic groups
- 1.05d - Explain how to perform an upgrade in a high availability group
- 1.05e - Describe the process for using SCF and UCS files

Objective - 1.06 - Describe the benefits of custom alerting within an LTM environment 51
- 1.06a - Describe Enterprise Manager, its uses, and alerting capabilities
- 1.06b - Describe AVR alerting capabilities

Objective - 1.07 - Describe how to set up custom alerting for an LTM device 54
- 1.07a - List and describe custom alerts: SNMP, email and Remote Syslog Identify the location of custom alert configuration files
- 1.07b - Identify the location of custom alert configuration files
- 1.07c - Identify the available levels for local traffic logging
- 1.07d - Identify the steps to trigger custom alerts for testing purposes

\=================================================================  
Section 2 - Troubleshoot basic hardware issues  
\=================================================================  

Objective - 2.01 Determine which iRule to use to resolve an application issue 69
- 2.01a - Determine which iRule events and commands to use

Objective - 2.02 Explain the functionality of a given iRule 70
- 2.02a - Interpret information in iRule logs to determine the iRule and iRule events where they occurred
- 2.02b - Describe the results of iRule errors

Objective - 2.03 Interpret AVR information to identify performance issues or application attacks 73
- 2.03a - Explain how to use AVR to trace application traffic
- 2.03b - Explain how latency trends identify application tier bottlenecks
- 2.03c - Explain browser requirements to obtain page load time data
- 2.03d - Explain how to use advanced filters to narrow output data from AVR
- 2.03e - Explain how to delete AVR data

Objective - 2.04 Interpret AVR information to identify LTM device misconfiguration 81
- 2.04a - Explain how to use AVR to trace application traffic
- 2.04b - Explain how latency trends identify application tier bottlenecks
- 2.04c - Explain how to use advanced filters to narrow output data from AVR
- 2.04d - Explain how to modify profile settings using information from the AVR
- 2.04e - Explain how to delete AVR data

Objective - 2.05 Given a scenario, determine the appropriate headers for an HTTP/HTTPS application 82
- 2.05a - Explain how to interpret response codes
- 2.05b - Explain the function of HTTP headers within different HTTP applications (Cookies, Cache Control, Vary, Content Type & Host)
- 2.05c - Explain HTTP methods (GET, POST, etc.)
- 2.05d - Explain the difference between HTTP versions (i.e., 1.0 and 1.1)

Objective - 2.06 Given a set of headers or traces, determine the root cause of an HTTP/HTTPS application problem 118
- 2.06a - Explain how to interpret response codes
- 2.06b - Explain the function of HTTP headers within different HTTP applications (Cookies, Cache Control, Vary, Content Type & Host)
- 2.06c - Explain HTTP methods (GET, POST, etc.)
- 2.06d - Explain the difference between HTTP versions (i.e., 1.0 and 1.1)
- 2.06e - Explain how to decrypt HTTPS data
- 2.06f - Explain how to decode POST data

Objective - 2.07 Given a set of headers or traces, determine a solution to an HTTP/HTTPS application problem 124
- 2.07a - Explain how to interpret response codes
- 2.07b - Explain circumstances under which HTTP chunking is required
- 2.07c - Explain the function of HTTP headers within different HTTP applications (Cookies, Cache Control, Vary, Content Type & Host)
- 2.07d - Explain HTTP methods (GET, POST, etc.)
- 2.07e - Explain the difference between HTTP versions (i.e., 1.0 and 1.1)
- 2.07f - Explain how to decrypt HTTPS data
- 2.07g - Explain how to decode POST data

Objective - 2.08 Given a direct trace and a trace through the LTM device, compare the traces to determine the root cause of an HTTP/HTTPS application problem 127
- 2.08c - Explain the function of HTTP headers within different HTTP applications (Cookies, Cache Control, Vary, Content Type & Host)
- 2.08d - Explain HTTP methods (GET, POST, etc.)
- 2.08e - Explain the difference between HTTP versions (i.e., 1.0 and 1.1)
- 2.08f - Explain how to decrypt HTTPS data
- 2.08g - Explain how to decode POST data
- 2.08g - Given a set of circumstances determine which persistence is required

Objective - 2.09 Given a direct trace and a trace through the LTM device, compare the traces to determine the root cause of an HTTP/HTTPS application problem 133
- 2.09a - Explain how to interpret response codes
- 2.09b - Explain circumstances under which HTTP chunking is required
- 2.09c - Explain the function of HTTP headers within different HTTP applications (Cookies, Cache Control, Vary, Content Type & Host)
- 2.09d - Explain HTTP methods (GET, POST, etc.)
- 2.09e - Explain the difference between HTTP versions (i.e., 1.0 and 1.1)
- 2.09f - Explain how to decrypt HTTPS data
- 2.09g - Explain how to decode POST data
- 2.09h - Given a set of circumstances determine which persistence is required

Objective - 2.10 Given a scenario, determine which protocol analyzer tool and its options are required to resolve an application issue 135
- 2.10a - Explain how to use the advanced flags in the protocol analyzers (e.g., tcpdump, -e, -s, -v, -X)
- 2.10b - Explain how to decrypt SSL traffic for protocol analysis
- 2.10c - Explain how to use DB keys to enhance the amount of data collected with protocol analyzers
- 2.10d - Determine which protocol analyzer options are safe to use based on traffic load and hardware model

Objective - 2.11 Given a trace, determine the root cause of an application problem 140
- 2.11a - Identify application issues based on a protocol analyzer trace
- 2.11b - Explain how to follow a conversation from client side and server side traces
- 2.11c - Explain how SNAT and OneConnect effect protocol analyzer traces
- 2.11d - Explain how to decrypt SSL traffic for protocol analysis
- 2.11e - Explain how time stamps are used to identify slow traffic
- 2.11f - Explain how to recognize the different causes of slow traffic (e.g., drops, RSTs, retransmits, ICMP errors, demotion from CMP)

Objective - 2.12 Given a trace, determine a solution to an application problem 149
- 2.12a - Identify application issues based on a protocol analyzer trace
- 2.12b - Explain how to follow a conversation from client side and server side traces
- 2.12c - Explain how SNAT and OneConnect effect protocol analyzer traces
- 2.12d - Explain how to decrypt SSL traffic for protocol analysis
- 2.12e - Explain how time stamps are used to identify slow traffic
- 2.12f - Explain how to recognize the different causes of slow traffic (e.g., drops, RSTs, retransmits, ICMP errors, demotion from CMP)

Objective - 2.13 Given a scenario, determine from where the protocol analyzer data should be collected 150
- 2.13a - Explain how to decrypt SSL traffic for protocol analysis
- 2.13b - Explain how to recognize the different causes of slow traffic (e.g., drops, RSTs, retransmits, ICMP errors, demotion from CMP)

Objective - 2.14 Given a trace, identify monitor issues 151
- 2.14a - Describe the appropriate output for an EAV monitor
- 2.14b - Identify the different types of monitors
- 2.14c - Describe the characteristics of the different types of monitors

Objective - 2.15 Given a monitor issue, determine an appropriate solution 158
- 2.15a - Describe the appropriate output for an EAV monitor
- 2.15b - Describe the steps necessary to generate advanced monitors
- 2.15c - Identify the different types of monitors
- 2.15d - Describe the characteristics of the different types of monitors
- 2.15e - Describe the input parameters passed to EAV monitors
- 2.14f - Describe the construct of an EAV monitor

\=================================================================  
Section 3 – Troubleshoot basic performance issues 162  
\=================================================================  

Objective - 3.01 Interpret log file messages and or command line output to identify LTM device issues 162
- 3.01a - Identify LTM device logs
- 3.01b - Identify the LTM device statistics in the GUI and command line
- 3.01c - Describe the functionality and uses of the AOM

Objective - 3.02 Identify the appropriate command to use to determine the cause of an LTM device problem 168
- 3.02a - Identify LTM device bash commands and TMSH commands needed to identify issues
- 3.02b - Explain how a virtual server processes a request (most specific to least specific)

Objective - 3.03 Given a scenario, determine the cause of an LTM device failover 172
- 3.03a - Interpret log messages to determine the cause of high availability issues
- 3.03b - Explain the effect of network failover settings on the LTM device
- 3.03c - Describe configuration settings (e.g., port lockdown, network timeouts, packet filters, remote switch settings)
- 3.03d - Explain the relationship between serial and network failover
- 3.03e - Differentiate between unicast and multicast network failover modes
- 3.03f - Identify the cause of failover using logs and statistics

Objective - 3.04 Given a scenario, determine the cause of loss of high availability and/or sync failure 186
- 3.04a - Explain how the high availability concepts relate to one another
- 3.04b - Explain the relationship between device trust and device groups
- 3.04c - Identify the cause of ConfigSync failures
- 3.04d - Explain the relationship between traffic groups and LTM objects
- 3.04e - Interpret log messages to determine the cause of high availability issues


