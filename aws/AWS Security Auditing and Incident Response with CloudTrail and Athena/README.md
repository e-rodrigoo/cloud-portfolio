# ☁️ AWS Security Auditing and Incident Response with CloudTrail and Athena

## 📌 Project Overview

This project demonstrates end-to-end cloud security auditing, forensic log analysis, and incident response on **Amazon Web Services (AWS)**. An **AWS CloudTrail** trail encrypted with **AWS KMS** was configured to record all API activity across the account and deliver compressed JSON event logs to an **Amazon S3** bucket. Following a simulated security breach and website defacement on an **Amazon EC2** Linux web server, multi-stage forensic analysis was conducted using Linux CLI utilities (`grep`, `python -m json.tool`), the **AWS CLI** (`lookup-events`), and serverless SQL queries in **Amazon Athena**.

After identifying the compromised **AWS IAM** identity (`chaos`), source IP address, and unauthorized `AuthorizeSecurityGroupIngress` API call that exposed SSH (`0.0.0.0/0`), full containment and remediation were executed across both the Linux operating system and the AWS control plane. This included terminating active rogue user sessions (`chaos-user`), hardening the OpenSSH daemon (`sshd_config`), locking down EC2 Security Group rules, restoring the web application assets, and revoking the compromised IAM credentials.

## 🏗️ Architecture Overview

```text
+-------------------------------------------------------------------------+
|                           AWS Cloud Environment                         |
|                                                                         |
|  +---------------------------+        +------------------------------+  |
|  |      AWS Control Plane    |        |      Amazon EC2 Instance     |  |
|  |                           |        |      (Linux Web Server)      |  |
|  | - AWS IAM (User: chaos)   |        | - Security Group (Ports 80,  |  |
|  | - EC2 Security Group API  |        |   22 Restricted to /32 IP)   |  |
|  +-------------+-------------+        | - Hardened sshd_config       |  |
|                |                      | - Auditd (aureport / who)    |  |
|                | (API Activity)       +------------------------------+  |
|                v                                                        |
|  +-------------+-------------+        +------------------------------+  |
|  |       AWS CloudTrail      |        |           AWS KMS            |  |
|  | - Trail: monitor          +------->+ - SSE-KMS Log Encryption     |  |
|  +-------------+-------------+        +------------------------------+  |
|                |                                                        |
|                | (Delivers .json.gz Logs)                               |
|                v                                                        |
|  +-------------+-------------+                                          |
|  |         Amazon S3         |                                          |
|  | - Bucket: monitoring####  |                                          |
|  +-------------+-------------+                                          |
|                |                                                        |
|                +-----------------------+                                |
|                |                       |                                |
|                v (CLI / Grep)          v (Serverless SQL)               |
|  +-------------+-------------+  +------+-----------------------------+  |
|  |   CLI Forensic Analysis   |  |            Amazon Athena           |  |
|  | - aws s3 cp & gunzip      |  | - Table: cloudtrail_logs_monitor   |  |
|  | - python -m json.tool     |  | - SQL Filtering by userIdentity,   |  |
|  | - cloudtrail lookup-events|  |   eventSource, and eventName       |  |
|  +---------------------------+  +------------------------------------+  |
+-------------------------------------------------------------------------+
```

## 🛠️ Technologies & AWS Services Used

* **AWS CloudTrail**: Audits and records account-wide API calls and management events, delivering log files to Amazon S3 for forensic analysis.
* **Amazon Athena**: Interactive serverless query service used to query structured CloudTrail JSON logs directly in Amazon S3 using standard SQL.
* **Amazon S3**: Stores compressed CloudTrail log archives (`.json.gz`) and Amazon Athena SQL query result sets.
* **AWS Key Management Service (AWS KMS)**: Encrypts CloudTrail log files at rest within the S3 bucket.
* **Amazon EC2 & Security Groups**: Hosts the Linux web server (`/cafe` web application) and enforces stateful network access control on ports `80` (HTTP) and `22` (SSH).
* **AWS Identity and Access Management (IAM)**: Manages account identities and permissions; used during remediation to permanently delete the compromised `chaos` user.
* **AWS CLI & Linux Security Utilities**: Includes `aws cloudtrail lookup-events`, `grep`, `python -m json.tool`, `aureport --auth`, `who`, `kill`, `userdel`, and `sshd_config` for host-level forensics and OS hardening.

## 🚀 Implementation Steps

1. **Baseline Network Security & Web Verification**: Configured the EC2 Security Group (`WebSecurityGroup`) to allow inbound HTTP (`80`) and restricted SSH (`22`) strictly to the administrator IP address (`/32`), confirming normal operation of the web application.
2. **CloudTrail Auditing Activation**: Created a multi-event CloudTrail trail named `monitor` backed by a dedicated Amazon S3 bucket (`monitoring####`) and encrypted with a custom AWS KMS key alias.
3. **Incident Detection (Defacement & Network Exposure)**: Identified an unauthorized modification to the web application landing page alongside a rogue inbound Security Group rule exposing TCP port `22` (SSH) to `0.0.0.0/0`.
4. **Log Extraction & CLI Triage**: Downloaded the compressed `.json.gz` audit logs from S3 to the EC2 instance using `aws s3 cp --recursive`, extracted them via `gunzip`, and parsed JSON fields (`sourceIPAddress`, `eventName`) using `python -m json.tool`, `grep`, and `aws cloudtrail lookup-events` filtered by `AWS::EC2::SecurityGroup`.
5. **Serverless SQL Forensics with Amazon Athena**: Generated an external Athena table (`cloudtrail_logs_monitoring####`) mapped to the S3 bucket schema and executed SQL queries filtering on `useridentity.userName`, `eventtime`, `eventsource = 'ec2.amazonaws.com'`, and `eventname` to pinpoint the exact user (`chaos`), timestamp, and source IP responsible for the `AuthorizeSecurityGroupIngress` breach.
6. **Linux OS Incident Containment**: Audited OS authentication logs using `sudo aureport --auth` and `who`, discovering an active session by the rogue local user `chaos-user`. Terminated the user's active process tree via `sudo kill -9` and purged the account and home directory with `sudo userdel -r chaos-user`.
7. **SSH Daemon & Security Group Hardening**: Inspected `/etc/ssh/sshd_config`, disabled password-based logins (`PasswordAuthentication no`), restarted the `sshd` daemon, and deleted the unauthorized `0.0.0.0/0` SSH rule from the EC2 Security Group.
8. **Application Recovery & IAM Remediation**: Restored the original web server image asset from backup in `/var/www/html/cafe/images/` and permanently deleted the compromised `chaos` IAM user from the AWS account.

## 📸 Project Verification & Screenshots

Detailed visual evidence of the baseline configuration, attack detection, Athena forensic SQL queries, Linux OS containment, and IAM remediation is available in the `screenshots` directory:

👉 **[View Implementation Screenshots](./screenshots/README.md)**

## 🧠 Key Takeaways & Best Practices

* **Centralized Immutable Audit Trails**: Enabling AWS CloudTrail with KMS encryption and dedicated S3 storage ensures every management API call (`AuthorizeSecurityGroupIngress`, console logins, IAM changes) is recorded for post-incident forensics.
* **Scalable Log Analytics with Amazon Athena**: While CLI tools (`grep`, `lookup-events`) work for quick checks, querying CloudTrail logs in S3 with Amazon Athena SQL drastically reduces Mean Time to Detect (MTTD) across large datasets.
* **Defense-in-Depth for SSH Access**: Restricting Security Group SSH rules to known `/32` CIDR blocks must be paired with OS-level hardening (`PasswordAuthentication no` in `/etc/ssh/sshd_config`) to prevent brute-force or credential-stuffing attacks if a network rule is misconfigured.
* **Thorough Incident Eradication**: Effective cloud incident response requires remediating both the cloud control plane (removing rogue IAM users and Security Group rules) and the compute workload (auditing `/etc/passwd`, killing active intruder processes, and removing unauthorized OS accounts).
