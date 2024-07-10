---
title: "F5 301b v1.1 Objectives"
date: 2024-04-29T20:14:22+08:00
---

These were the official 11.5 objectives but modified by me. They have been cleaned up even removed in some cases.

This objective list I prefer over the current ones that are official.  
{{< line_break >}}
\================================================================  
Section 1 - Maintain Application and LTM Device Health  
\================================================================  

Objective 1.01 Given a scenario, determine the appropriate profile setting modifications.
- 1.01a - Given a scenario of client or server side buffer issues, packet loss, or congestion, select the appropriate TCP or UDP profile to correct the issue
- 1.01b - Given a scenario determine when an application would benefit from HTTP Compression and/or Web Acceleration profile

Objective 1.02 Given a subset of an LTM configuration or statistics, determine which objects to remove or consolidate to simplify the LTM configuration
- 1.02a - Evaluate which iRules can be replaced with a profile or policy setting
- 1.02b - Evaluate which host virtual servers would be better consolidated into a network virtual server
- 1.03a - Identify redundant and/or unused objects
- 1.03b - Identify unnecessary monitoring
- 1.03c - Interpret configuration and performance statistics 
- 1.03d - Explain the effect of removing functions from the LTM device configuration

Objective 1.04 Given a scenario, determine the appropriate upgrade and recovery steps required to restore functionality to LTM devices
- 1.04a - Identify the appropriate methods for a clean install
- 1.04b - Identify the TMSH sys software install options required to install a new version
- 1.04c - Identify the steps required to upgrade the LTM device such as: license renewal, validation of upgrade path, review release notes, etc.
- 1.04d - Identify how to copy a config to a previously installed boot location/slot
- 1.04e - Identify valid rollback steps for a given upgrade scenario

Objective 1.05 Given a scenario, determine the appropriate upgrade steps required to minimize application outages
- 1.05a - Explain how to upgrade an LTM device from the GUI
- 1.05b - Describe the effect of performing an upgrade in an environment with device groups and traffic groups
- 1.05c - Explain how to perform an upgrade in a high availability group

Objective 1.06 Describe the benefits of custom alerting within an LTM environment
- 1.06a - Describe how to specify the OIDs for alerting
- 1.06b - Explain how to log different levels of local traffic message logs
- 1.06c - Explain how to trigger custom alerts for testing purposes
- 1.07a - List and describe custom alerts: SNMP, email and Remote Syslog
- 1.07b - Identify the location of custom alert configuration files
- 1.07c - Identify the available levels for local traffic logging

\================================================================  
Section 2 - Identify and Resolve Application Issues  
\================================================================  

Objective 2.01 Determine which iRule to use to resolve an application issue
- 2.01a - Determine which iRule events and commands to use
- 2.01b - Given a specific iRule event determine what commands are available

Objective 2.02 Explain the functionality of a simple iRule
- 2.02a - Interpret information in iRule logs to determine the iRule and iRule events where they occurred
- 2.02b - Describe the results of iRule errors

Objective 2.03 Given specific traffic and configuration containing a simple iRule determine the result of the iRule on the traffic
- 2.03a - Use an iRule to resolve application issues related to traffic steering and/or application data

Objective 2.04 Interpret AVR information to identify performance issues or application attacks
- 2.04a - Explain how to modify profile settings using information from the AVR 
- 2.04b - Explain how to use advanced filters to narrow output data from AVR 
- 2.04c - Identify potential latency increases within an application

Objective 2.05 Interpret AVR information to identify LTM device misconfiguration
- 2.05a - Explain how to use AVR to trace application traffic 
- 2.05b - Explain how latency trends identify application tier bottlenecks

Objective 2.06 Given a set of headers or traces, determine the root cause of an HTTP/HTTPS application problem
- 2.06a - Explain how to interpret response codes
- 2.06b - Explain the function of HTTP headers within different HTTP applications (Cookies, Cache Control, Vary, Content Type & Host)
- 2.06c - Explain HTTP methods (GET, POST, etc.)
- 2.06d - Explain how to decode POST data.
- 2.07a - Investigate the cause of a specific response code
- 2.07b - Investigate the cause of an SSL Handshake failure
- 2.07c - Predict the browser caching behavior when application data is received (headers and HTML)

Objective 2.09 Given a direct trace, a trace through the LTM device, and other relevant information, compare the traces to determine a solution to an HTTP/HTTPS application problem
- 2.09b - Given a failed HTTP request and LTM configuration data determine if the connection is failing due to the LTM configuration

Objective 2.10 Given a scenario, determine which protocol analyzer tool and its options are required to resolve an application issue
- 2.10a - Identify application issues based on a protocol analyzer trace and determine solution
- 2.10b - Explain how to follow a conversation from client side and server side traces
- 2.10c - Explain how SNAT and OneConnect affect protocol analyzer traces
- 2.10d - Explain how to decrypt SSL traffic for protocol analysis
- 2.10e - Explain how to recognize the different causes of slow traffic (e.g., drops, RSTs, retransmits, ICMP errors, demotion from CMP)

Objective 2.14 Given a trace, identify monitor issues
- 2.14a - Explain how to capture and interpret monitor traffic using protocol analyzer
- 2.14b - Explain how to obtain needed input and output data to create the monitors

Objective 2.15 Given a monitor issue, determine an appropriate solution
- 2.15a - Determine appropriate monitor and monitor timing based on application and server limitations
- 2.15b - Describe how to modify monitor settings to resolve monitor problems

\================================================================  
Section 3 - Identify and Resolve LTM Device Issues  
\================================================================  

Objective 3.01 Interpret log file messages and/or command line output to identify LTM device issues
- 3.01a - Interpret log file messages to identify LTM device issues
- 3.01b - Interpret the qkview heuristic results
- 3.01c - Identify appropriate methods to troubleshoot NTP
- 3.01d - Identify license problems based on the log file messages and statistics

Objective 3.02 Identify the appropriate command to use to determine the cause of an LTM device problem
- 3.02a - Identify hardware problems based on the log file messages and statistics
- 3.02b - Identify resource exhaustion problems based on the log file messages and statistics
- 3.02c - Identify connectivity problems based on the log files
- 3.02d - Determine the appropriate log file to examine to determine the cause of the problem

Objective 3.03 Analyze performance data to identify a resource problem on an LTM device
- 3.03a - Analyze performance data to identify a resource problem on an LTM device

Objective 3.04 Given a scenario, determine the cause of an LTM device failover
- 3.04a - Explain the effect of network failover settings on the LTM device 
- 3.04b - Explain the relationship between serial and network failover 
- 3.04c - Differentiate between unicast and multicast network failover modes
- 3.04d - Identify the cause of failover using logs and statistics

Objective 3.05 Given a scenario, determine the cause of loss of high availability and/or sync failure
- 3.05a - Explain how the high availability concepts relate to one another
- 3.05b - Explain the relationship between device trust and device groups
- 3.05c - Identify the cause of config sync failures
- 3.05d - Explain the relationship between traffic groups and LTM objects
- 3.05e - Interpret log messages to determine the cause of high availability issues



