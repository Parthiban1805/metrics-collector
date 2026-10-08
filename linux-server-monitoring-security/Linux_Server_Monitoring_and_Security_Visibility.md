# **1. Overview** 

A production Linux server requires two complementary capabilities: 

- **System Observability** – understanding the health and behaviour of the server. 

- **Security Monitoring** – detecting, investigating and responding to security threats. 

Netdata is primarily an infrastructure monitoring and observability platform. It provides real-time visibility into CPU, memory, disk, network, processes, services and other system metrics. 

Netdata is not a complete SIEM or Host-based Intrusion Detection System (HIDS). For detailed security auditing, malware detection, threat analysis and automated response, dedicated security tools should be used alongside Netdata. 

## **1.1 Solution Architecture** 

The approach combines one observability tool with four dedicated security tools. Each addresses a different question, and together they form a layered defence. 



<!-- Start of picture text -->
a + »¥<br>Metis | Logs /Keme Events Files<br>a |<br>Netdata auditd ‘ClamAV<br>Observability Auditing Malware Scanning<br>\/<br>—a<br>Performance Visibility Security Evidence<br>&Alerts<br>r J<br>Nga<br>Wazuh<br>Detection & Correlation<br>Threat Detection<br>& Security Alerts<br>Fail2Ban<br>‘Automated Response<br>IP Blocked via Firewall<br><!-- End of picture text -->

# **2. Part 1 – Security Visibility With Netdata** 

Netdata is not primarily a security tool. However, its system and application metrics provide valuable security visibility by revealing abnormal behaviour. It is particularly useful for answering: 

- _“Is something unusual happening on my server?”_ 

It is generally not designed to answer: 

- _“Who exactly performed the action?”_ 

That deeper forensic information is provided by tools such as auditd and Wazuh. 



<!-- Start of picture text -->
Server Signals<br>cpu Memory Network Disk uo Processes<br>Netdata<br>Collection & Analysis<br>Metics & Alerts<br>‘Something is abnormal”<br>Security(ausitdInvestigation -Wazuh)<br><!-- End of picture text -->

## **2.1 CPU Monitoring** 

### **What it monitors** 

- User CPU usage 

- System / kernel CPU usage 

- I/O wait 

- Interrupt and soft-interrupt processing 

- Steal time 

- Idle time 

### **Security relevance** 

|**Indicator**|**Normal**|**Abnormal**|
|---|---|---|
|**CPU utilisation**|20% → 25% → 22% → 24%|25% → 40% → 75% → 98%|



**Possible causes of abnormal behaviour:** 

- Runaway process 

- Unexpected workload 

- Cryptomining activity 

- Resource exhaustion 

- Application malfunction 

- Potential compromise 

### **Benefit** 

CPU monitoring helps identify unexpected computational activity, sustained resource consumption and processes behaving differently from normal. 

**Note:** CPU usage alone does not prove that an attack occurred. It is an indicator that may require further investigation. 

## **2.2 Memory Monitoring** 

### **What it monitors** 

- RAM utilisation 

- Available and cached memory 

- Swap usage 

- Memory pressure 

- Per-process memory consumption 

### **Security relevance** 

|**Indicator**|**Normal**|**Abnormal**|
|---|---|---|
|**RAM usage**|45%|95%|
|**Swap usage**|0%|80%|



**Possible causes of abnormal behaviour:** 

- Memory leak 

- Runaway process 

- Unexpected workload 

- Denial-of-service condition 

- Malicious process consuming resources 

### **Benefit** 

Memory monitoring helps detect resource exhaustion and unusual system behaviour. 

## **2.3 Process Monitoring** 

### **What it monitors** 

- CPU consumption per process 

- Memory consumption per process 

- Process count 

- Process states 

- Process activity 

### **Security relevance** 

|**Category**|**Examples**|
|---|---|
|**Expected processes**|node, nginx, mongod, redis|
|**Unexpected processes**|unknown_binary, suspicious_process, xmrig|



An existing process that suddenly consumes excessive CPU can also become an investigation target. 

### **Benefit** 

Process monitoring can help identify unexpected processes, resource-heavy processes, process explosions, runaway applications and potential cryptomining activity. 

**Note:** Netdata can show that a process is consuming resources, but it is not primarily responsible for establishing who launched that process. 

## **2.4 Network Monitoring** 

### **What it monitors** 

- Incoming and outgoing traffic 

- Packets 

- Network errors 

- Network interfaces 

- Connections 

- Bandwidth utilisation 

### **Security relevance** 

|**Indicator**|**Normal**|**Abnormal**|
|---|---|---|
|**Outbound traffic**|2 MB/s|150 MB/s|



**Possible causes of abnormal behaviour:** 

- Legitimate large data transfer 

- Backup operation 

- Data synchronisation 

- DDoS-related traffic 

- Unexpected data transmission 

- Potential data exfiltration 

### **Benefit** 

Network monitoring identifies traffic anomalies and provides an early indication that network behaviour has changed. 

## **2.5 Disk I/O Monitoring** 

### **What it monitors** 

- Disk reads and writes 

- I/O throughput 

- I/O latency 

- Disk utilisation 

- I/O queues 

### **Security relevance** 

|**Indicator**|**Normal**|**Abnormal**|
|---|---|---|
|**Disk writes**|Low|Extremely high|



**Possible causes of abnormal behaviour:** 

- Database workload 

- Log generation 

- Backup operation 

- Large file processing 

- Application malfunction 

- Malware activity 

### **Benefit** 

Disk monitoring can identify unusual storage activity. 

**Note:** Netdata does not inherently provide forensic detail such as “User X deleted /var/log/application.log at 02:31.” That requires detailed auditing with tools such as auditd. 

## **2.6 Filesystem Monitoring** 

### **What it monitors** 

- Filesystem utilisation 

- Disk usage 

- I/O activity 

- Filesystem errors 

- Mount availability 

- Storage changes 

### **Security relevance** 

|**Indicator**|**Normal**|**Abnormal**|
|---|---|---|
|**Disk usage trend**|40 GB (stable)|40 → 60 → 80 → 95 GB|



### **Possible causes of abnormal behaviour:** 

- Application logs 

- Database growth 

- Temporary files 

- Backups 

- Unexpected file creation 

- Malware-generated files 

### **Benefit** 

Filesystem metrics provide early warning of abnormal storage behaviour. For exact file-level activity, dedicated security tools are more appropriate. 

## **2.7 System Load Monitoring** 

### **What it monitors** 

- System load average 

- Resource pressure 

### **Security relevance** 

|**Indicator**|**Normal**|**Abnormal**|
|---|---|---|
|**Load average**|1.2|20|
|**CPU utilisation**|25%|95%|



**Possible causes of abnormal behaviour:** 

- High application workload 

- CPU-intensive process 

- I/O bottleneck 

- Large number of processes 

- Resource exhaustion 

- Potential denial-of-service activity 

### **Benefit** 

System load provides a high-level indication that the server is under unusual pressure. 

## **2.8 Authentication-Related Visibility** 

Netdata is not an authentication security system. However, authentication-related metrics and log monitoring can contribute to detecting abnormal behaviour, such as a sudden increase in failed SSH logins. 



<!-- Start of picture text -->
Normal SSH Sudden Rise in Potential<br>Activity Failed Logins Brute-Force Activity<br>Security auditd / Wazuh Fail2Ban<br>Investigation Analysis. Blocks Source<br><!-- End of picture text -->

Detailed authentication analysis is better handled by Linux authentication logs, auditd, Wazuh and Fail2Ban. 

### **Benefit** 

Netdata contributes operational awareness, while dedicated security tools perform detailed authentication analysis and response. 

## **2.9 Service Monitoring** 

### **What it monitors** 

- Application and service health checks 

- Supported services such as nginx, MongoDB, Redis, Node.js and Docker 

### **Security relevance** 

|**Indicator**|**Normal**|**Abnormal**|
|---|---|---|
|**Service state**|Running and healthy|Crash or unexpected restart|
|**Resources**|Normal CPU and memory|High resource usage|
|**Connections**|Normal|Connection explosion|
|**Performance**|Stable|Degradation|



**Possible causes of abnormal behaviour:** 

- Operational faults 

- Security-related problems 

### **Benefit** 

Service monitoring identifies unexpected application behaviour that can originate from either operational or security problems. 

## **2.10 Metrics and Alerts** 

Netdata establishes visibility into normal system behaviour and generates alerts when monitored conditions become abnormal. Typical alert conditions include: 

- High CPU, memory or disk usage 

- High network traffic or disk I/O 

- Service failure 

- Resource exhaustion 

### **Benefit** 

Netdata acts as an early-warning layer. It can tell an administrator that the server behaviour has changed. Security tools then investigate why it changed, who caused it and whether it was malicious. 

**Summary:** Netdata is the observability layer rather than the complete security layer. It answers “something is abnormal”; the tools in Part 2 answer the questions that follow. 

# **3. Part 2 – Security Visibility Without Netdata** 

Netdata can identify abnormal system behaviour, but dedicated security tools are required for deeper security visibility. These tools answer questions such as: 

- Who performed an action? 

- Which process modified a file? 

- Was a file malicious? 

- Is the server vulnerable? 

- Is there a rootkit? 

- Is an IP address repeatedly attacking the server? 

- Should suspicious traffic be blocked? 

The major open-source tools covered are auditd, Wazuh, ClamAV and Fail2Ban. 

## **3.1 auditd – Linux System Activity Auditing** 

auditd is the Linux system auditing framework. It records security-relevant activity performed on the operating system. 

### **What it monitors** 

- File creation, modification, deletion and access 

- User login and logout 

- Permission changes and privilege escalation 

- System calls and command execution 

- Changes to security-critical files 

- User and group modifications 

A security-critical file such as /etc/passwd can be audited. Each record can include the timestamp, user ID, process ID, executable, action, file involved and result 



<!-- Start of picture text -->
Ava Femework<br>Aut Logs Executable - Acton<br>oe TimestampFile -- UserResult ID - PID<br><!-- End of picture text -->

### **Benefit** 

The main benefit is accountability and forensic visibility. It helps answer who modified a file, which process deleted it, which user executed a command and when. Netdata may show that unusual disk or file activity occurred; auditd provides the audit trail needed to investigate it. 

## **3.2 Wazuh – Host Security Monitoring and SIEM** 

Wazuh is an open-source security monitoring and threat detection platform. It collects and analyses security information from servers and endpoints. 

### **Capabilities** 

|**Capability**|**Capability**|
|---|---|
|Log analysis|Vulnerability detection|
|File Integrity Monitoring|Security configuration assessment|
|Rootkit detection|Threat detection|
|Malware detection|Authentication monitoring|
|Compliance monitoring|Active response|



### **File Integrity Monitoring** 

Wazuh maintains a baseline of important files, for example /etc/passwd, /etc/shadow, /etc/ssh/sshd_config and application configuration files. When a monitored file changes, Wazuh raises a security event. 



<!-- Start of picture text -->
File Baseline<br>Jetcipasswdsshd_config -- /etc/shadow app configs SSFile Modified. DetectsWazuh Change<br>Security Event Alert / Dashboard<br><!-- End of picture text -->

### **Benefit** 

Wazuh provides a centralised security monitoring layer. Instead of manually checking logs on individual servers, administrators can analyse security events from one platform. It is useful for server security monitoring, intrusion detection, file integrity monitoring, vulnerability monitoring, security investigation and compliance monitoring. 

## **3.3 ClamAV – Malware Detection** 

ClamAV is an open-source antivirus and malware detection engine. It analyses files for known malicious content using malware signatures and detection rules. 

### **Detects** 

- Viruses, trojans and worms 

- Malware and malicious scripts 

- Potentially unwanted files 



<!-- Start of picture text -->
Clean_——»|__ Storage<br>}—<br>User Upload File: ClamAVScan<br>™<br>Malicious Reject / Quarantine Alert<br><!-- End of picture text -->

### **Benefit** 

ClamAV provides file-level malware detection. It is particularly useful for uploaded files, server directories, shared storage, suspicious files and application file-upload systems. 

## **3.4 Fail2Ban – Automated Brute-Force Protection** 

Fail2Ban monitors logs for repeated suspicious authentication activity, such as repeated SSH login failures, and automatically blocks the offending source. It can be used with SSH, web servers, mail servers, authentication systems and other applications with recognisable security logs. 



<!-- Start of picture text -->
Attacker Failed logins SSH Server Authenticationuinenticali<br>Logs<br>,<br>Fail2Ban Threshold exceeded Firewall Rule >| IP Address<br>Pattern Detection Blocked<br><!-- End of picture text -->

Technically, it combines log monitoring, pattern detection, a failure threshold (for example five failed attempts) and a firewall action. 

### **Benefit** 

Fail2Ban provides automated protection against repeated attacks, reducing exposure to SSH bruteforce attacks, password guessing, repeated authentication attempts and automated login attacks. Unlike a tool that only reports an event, it also performs an automated defensive action. 

# **4. Tool Comparison** 

Each tool answers a different question. They complement one another rather than replace one another. 

|**Tool**|**Primary Purpose**|**Main Question**|
|---|---|---|
|**Netdata**|System monitoring|What is happening to my server?|
|**auditd**|System auditing|Who performed this system action?|
|**Wazuh**|Security monitoring|Does this activity indicate a security threat?|
|**ClamAV**|Malware detection|Is this file malicious?|
|**Fail2Ban**|Attack prevention|Should this suspicious source be blocked?|



## **4.1 Security Coverage Matrix** 

|**Requirement**|**Netdata**|**auditd**|**Wazuh**|**ClamAV**|**Fail2Ban**|
|---|---|---|---|---|---|
|**CPU anomaly detection**|✓|||||
|**Memory anomaly detection**|✓|||||
|**Network anomaly detection**|✓|||||
|**Disk I/O monitoring**|✓|||||
|**Service monitoring**|✓|||||
|**System resource monitoring**|✓|||||
|**File modification auditing**|Limited|✓|✓|||
|**File deletion auditing**|Limited|✓|✓|||
|**Command execution auditing**|Limited|✓|✓|||
|**User activity auditing**|Limited|✓|✓|||
|**File integrity monitoring**|Limited||✓|||
|**Vulnerability detection**|||✓|||
|**Rootkit detection**|||✓|||
|**Security event correlation**|Limited||✓|||
|**Malware scanning**|||✓*|✓||
|**Brute-force detection**|Limited||✓||✓|
|**Automated IP blocking**|||✓*||✓|
|**Security dashboard**|✓||✓|||
|**Automated security response**|||✓*||✓|



_* Depends on the specific configuration and integrations._ 

# **5. Core Concept and Summary** 

The security architecture can be understood as five sequential functions: observe, audit, detect, scan and respond. 



<!-- Start of picture text -->
OBSERVE- Netdata<br>What is happening on the<br>server?<br>AUDIT- auditd<br>Who performed the action?<br>DETECT - Wazuh<br>Does it indicate a threat?<br>SCAN- ClamAV<br>Is the file malicious?<br>RESPOND- Fail2Ban<br>Should the source be<br>blocked?<br><!-- End of picture text -->

|**Function**|**Tool**|**Role**|**Key Question**|
|---|---|---|---|
|**Observe**|Netdata|Monitors infrastructure and<br>application behaviour|What is happening on the server?|
|**Audit**|auditd|Records security-sensitive operating-<br>system activity|Who performed this action?|
|**Detect**|Wazuh|Correlates security information and<br>detects potential threats|Does this indicate a security<br>problem?|
|**Scan**|ClamAV|Analyses files for known malicious<br>content|Is this file malicious?|
|**Respond**|Fail2Ban|Automatically reacts to repeated<br>suspicious activity|Should this source be blocked?|



**Conclusion:** Netdata provides operational visibility, while dedicated security tools provide auditing, threat detection, malware analysis and automated response. Together they create a layered approach to monitoring and securing a Linux server. 

