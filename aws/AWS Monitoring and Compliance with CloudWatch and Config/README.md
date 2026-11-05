# ☁️ AWS Infrastructure Monitoring and Compliance with CloudWatch and Config

## 📌 Project Overview

This project implements a full-stack observability, real-time alerting, and continuous compliance auditing architecture on **Amazon Web Services (AWS)**. Rather than relying solely on default hypervisor-level telemetry, this setup deploys the **Amazon CloudWatch Agent** via **AWS Systems Manager (SSM)** to extract OS-level metrics (memory, disk, swap, CPU) and stream Apache HTTP Server (`httpd`) logs directly from **Amazon EC2** to **Amazon CloudWatch Logs**.

In addition to proactive log metric filtering and automated alerting via **Amazon SNS**, the architecture uses **Amazon CloudWatch Events (EventBridge)** to detect instance state transitions in real time and **AWS Config** to enforce organizational governance rules across compute and storage resources.

## 🏗️ Architecture Overview

```text
+-------------------------------------------------------------------------+
|                           AWS Cloud Environment                         |
|                                                                         |
|  +---------------------------+        +------------------------------+  |
|  |   AWS Systems Manager     |        |       Amazon EC2 Instance    |  |
|  |                           |        |         (Web Server)         |  |
|  | - Run Command (Deploy)    +------->+ - Apache HTTP Server (httpd) |  |
|  | - Parameter Store (JSON)  |        | - CloudWatch Agent           |  |
|  +---------------------------+        +--------------+---------------+  |
|                                                      |                  |
|         +--------------------------------------------+                  |
|         | (Streams Logs & OS Metrics)                | (State Changes)  |
|         v                                            v                  |
|  +------+--------------------+        +--------------+---------------+  |
|  |     Amazon CloudWatch     |        |   CloudWatch Events / Rules  |  |
|  |                           |        |                              |  |
|  | - Log Groups (Access/Err) |        | - EC2 State-Change Rule      |  |
|  | - Metric Filter (HTTP 404)|        |   (Stopped / Terminated)     |  |
|  | - CWAgent Custom Metrics  |        +--------------+---------------+  |
|  | - CloudWatch Alarm (>= 5) |                       |                  |
|  +------+--------------------+                       |                  |
|         | (Triggers Alarm)                           | (Routes Event)   |
|         +--------------------+-----------------------+                  |
|                              |                                          |
|                              v                                          |
|               +--------------+---------------+                          |
|               |          Amazon SNS          |                          |
|               | - Email Notification Topic   |                          |
|               +------------------------------+                          |
|                                                                         |
|  +-------------------------------------------------------------------+  |
|  |                            AWS Config                             |  |
|  | - Managed Rule: required-tags (Key: project)                      |  |
|  | - Managed Rule: ec2-volume-inuse-check (Unattached EBS Detection) |  |
|  +-------------------------------------------------------------------+  |
+-------------------------------------------------------------------------+
```

## 🛠️ Technologies & AWS Services Used

* **Amazon EC2**: Hosts the Linux web server running Apache (`httpd`) and the `AmazonCloudWatchAgent` service.
* **AWS Systems Manager (SSM)**:
  * **Run Command**: Remotely installs and configures the CloudWatch Agent (`AWS-ConfigureAWSPackage` and `AmazonCloudWatch-ManageAgent`) without requiring direct SSH access.
  * **Parameter Store**: Centralizes the JSON agent configuration (`Monitor-Web-Server`) defining log paths and metric collection intervals.
* **Amazon CloudWatch Logs**: Ingests and stores `/var/log/httpd/access_log` (`HttpAccessLog`) and `/var/log/httpd/error_log` (`HttpErrorLog`) streams per instance ID.
* **Amazon CloudWatch Metrics & Alarms**: Collects custom OS-level telemetry under the `CWAgent` namespace and evaluates metric filters (`LogMetrics/404Errors`) to trigger high-severity notifications.
* **Amazon CloudWatch Events (EventBridge)**: Monitors EC2 lifecycle events and triggers real-time notifications when instances transition to `stopped` or `terminated` states.
* **Amazon Simple Notification Service (SNS)**: Delivers real-time email alerts for both application-layer anomalies (HTTP 404 spikes) and infrastructure state changes.
* **AWS Config**: Continuously audits resource configurations against managed compliance rules (`required-tags` and `ec2-volume-inuse-check`).

## 🚀 Implementation Steps

1. **Agent Deployment via Systems Manager**: Executed the `AWS-ConfigureAWSPackage` SSM document through Run Command to remotely install the latest `AmazonCloudWatchAgent` package on the target EC2 Web Server.
2. **Centralized Configuration in Parameter Store**: Created the `Monitor-Web-Server` parameter containing the JSON specification to collect Apache access/error logs and OS-level metrics (`cpu`, `disk`, `diskio`, `mem`, `swap`) at 10-second intervals. Applied the configuration using the `AmazonCloudWatch-ManageAgent` Run Command document.
3. **Application Log Ingestion & Metric Filtering**: Verified real-time log shipping into `HttpAccessLog` and `HttpErrorLog` log groups. Configured the `404Errors` metric filter using the pattern `[ip, id, user, timestamp, request, status_code=404, size]` to extract HTTP 404 responses into the `LogMetrics` namespace.
4. **Proactive Threshold Alerting**: Created a CloudWatch Alarm (`404 Errors`) monitoring the custom metric over a 1-minute period, triggering an Amazon SNS email notification whenever five or more 404 errors occur.
5. **OS-Level Metric Inspection**: Validated internal host metrics inside the `CWAgent` namespace, enabling visibility into memory utilization (`mem_used_percent`) and filesystem capacity (`used_percent`, `inodes_free`).
6. **Event-Driven State Notifications**: Built an EventBridge / CloudWatch Events rule (`Instance_Stopped_Terminated`) matching EC2 state-change notifications for `stopped` and `terminated` states, routing alerts directly to an SNS topic.
7. **Continuous Compliance Auditing with AWS Config**: Enabled AWS Config recording and deployed two managed rules: `required-tags` (enforcing the `project` tag key across resources) and `ec2-volume-inuse-check` (identifying unattached EBS volumes to prevent orphaned storage costs).

## 📸 Project Verification & Screenshots

Detailed visual evidence of each configuration step, log stream verification, alarm state transition, and compliance audit is available in the `screenshots` directory:

👉 **[View Implementation Screenshots](./screenshots/README.md)**

## 🧠 Key Takeaways & Best Practices

* **Zero-Touch Fleet Management**: Using AWS Systems Manager Run Command alongside Parameter Store eliminates the need to open SSH ports or manage bastion hosts when deploying and configuring monitoring agents across large EC2 fleets.
* **Bridging the Hypervisor Visibility Gap**: Standard EC2 metrics only report hypervisor-visible stats (CPU, network, raw disk I/O). Deploying the CloudWatch Agent is essential for monitoring guest OS memory pressure, swap usage, and partition-level disk exhaustion.
* **Log-Driven Observability**: Extracting metrics directly from existing Apache log files allows teams to detect application-level issues (such as broken links or directory enumeration scans generating HTTP 404 spikes) without modifying application source code.
* **Automated FinOps & Governance**: Implementing AWS Config rules like `ec2-volume-inuse-check` and `required-tags` automates cost-allocation auditing and immediately surfaces unattached EBS volumes that continue accruing charges.
