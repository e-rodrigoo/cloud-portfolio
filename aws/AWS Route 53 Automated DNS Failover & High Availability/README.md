# ☁️ AWS Route 53 Automated DNS Failover & High Availability

## 📌 Project Overview

This project demonstrates the implementation of an automated **Active-Passive Disaster Recovery (DR)** and **High Availability** architecture on **Amazon Web Services (AWS)** using **Amazon Route 53**.

Instead of relying on a single point of failure, a LAMP-stack web application ("The Café") is deployed across two distinct **Availability Zones (`us-west-2a` and `us-west-2b`)**. By leveraging Route 53 Health Checks, CloudWatch Alarms, and Failover Routing Policies, the infrastructure continuously monitors endpoint health and automatically redirects user traffic to a standby instance during an outage, while triggering real-time email alerts via **Amazon SNS**.

## 🏗️ Architecture Overview

```text
                                  +------------------+
                                  |   End Users      |
                                  +--------+---------+
                                           |
                                    (DNS Query: www)
                                           v
                    +----------------------------------------------+
                    |               Amazon Route 53                |
                    |          (Hosted Zone / Failover)            |
                    +----------------------+-----------------------+
                                           |
                  +------------------------+------------------------+
                  |                                                 |
       [PRIMARY RECORD - Active]                        [SECONDARY RECORD - Passive]
       Routing Policy: Failover                         Routing Policy: Failover
       Health Check: Associated                         Health Check: None
                  |                                                 |
                  | (Healthy: Normal Traffic)                       | (Unhealthy: Failover Traffic)
                  v                                                 v
+-----------------------------------+             +-----------------------------------+
| VPC                               |             | VPC                               |
|  +-----------------------------+  |             |  +-----------------------------+  |
|  | Availability Zone A         |  |             |  | Availability Zone B         |  |
|  | (us-west-2a)                |  |             |  | (us-west-2b)                |  |
|  |                             |  |             |  |                             |  |
|  |  +-----------------------+  |  |             |  |  +-----------------------+  |  |
|  |  | Public Subnet 1       |  |  |             |  |  | Public Subnet 2       |  |  |
|  |  |                       |  |  |             |  |  |                       |  |  |
|  |  |  +-----------------+  |  |  |             |  |  |  +-----------------+  |  |  |
|  |  |  | CafeInstance1   |  |  |  |             |  |  |  | CafeInstance2   |  |  |  |
|  |  |  | (LAMP Web App)  |  |  |  |             |  |  |  | (LAMP Web App)  |  |  |  |
|  |  |  +--------+--------+  |  |  |             |  |  |  +-----------------+  |  |  |
|  |  +-----------|-----------+  |  |             |  |  +-----------------------+  |  |
|  +--------------|--------------+  |             +-----------------------------------+
+-----------------|-----------------+
                  |
                  | (HTTP Health Check: Every 10s on /cafe)
                  v
    +---------------------------+          +----------------------------------+
    |   Route 53 Health Check   | -------> | CloudWatch Alarm & Amazon SNS    |
    |  (Threshold: 2 Failures)  | (Alert)  | (Email Notification to Admin)    |
    +---------------------------+          +----------------------------------+
```

## 🛠️ AWS Services & Technologies Used

* **Amazon Route 53:** Managed DNS service configured with Public Hosted Zones, Health Checks, and Failover Routing Policies (`A` Records).
* **Amazon EC2:** Dual Linux instances running a full **LAMP Stack** (Linux, Apache, MySQL, PHP) across multi-AZ public subnets.
* **Amazon CloudWatch:** Monitors the Route 53 health check metric and triggers an alarm state upon consecutive failures.
* **Amazon SNS (Simple Notification Service):** Dispatches automated email alerts to administrators when the primary endpoint becomes unhealthy.
* **AWS CloudFormation:** Used to bootstrap the initial VPC, Subnets, and EC2 base environment.

## ⚙️ Configuration Breakdown

### 1. Route 53 Health Check Configuration
To ensure rapid detection of an outage in the primary Availability Zone, the health check was tuned with fast intervals:

| Parameter | Configuration | Purpose |
| :--- | :--- | :--- |
| **Name** | `Primary-Website-Health` | Identifies the primary instance monitor |
| **What to Monitor** | Endpoint (by IPv4 Address) | Targets `CafeInstance1` Public IP directly |
| **Protocol & Path** | `HTTP` / `/cafe` | Verifies application-level availability, not just server ping |
| **Request Interval** | Fast (`10 seconds`) | Minimizes detection time during an outage |
| **Failure Threshold** | `2` consecutive failures | Triggers unhealthy state within ~20–30 seconds |
| **SNS Notification** | Enabled (`Primary-Website-Health`) | Sends immediate email alert upon failure |

### 2. DNS Failover Records (Hosted Zone)
Two `A` records sharing the same subdomain (`www`) were created to govern traffic routing:

| Record Attribute | Primary Record (`FailoverPrimary`) | Secondary Record (`FailoverSecondary`) |
| :--- | :--- | :--- |
| **Record Name** | `www.<domain>` | `www.<domain>` |
| **Record Type** | `A` (IPv4) | `A` (IPv4) |
| **Value (Target)** | `CafeInstance1` Public IP (`us-west-2a`) | `CafeInstance2` Public IP (`us-west-2b`) |
| **TTL (Seconds)** | `15` | `15` |
| **Routing Policy** | Failover | Failover |
| **Failover Record Type** | **Primary** | **Secondary** |
| **Health Check** | `Primary-Website-Health` | *None (Standby)* |

> **💡 Why a 15-second TTL?** A low Time-to-Live (TTL) ensures that DNS resolvers do not cache the unhealthy IP address for long, allowing client browsers to resolve the secondary standby server almost immediately after a failover event.

## 🚀 Implementation & Failover Testing Steps

### Phase 1: Baseline Verification
1. Inspected the two pre-provisioned EC2 instances (`CafeInstance1` in `us-west-2a` and `CafeInstance2` in `us-west-2b`).
2. Accessed both endpoints directly via HTTP and verified the **Server Information** panel, confirming both instances were actively serving the Café web application and processing orders independently.

### Phase 2: Health Check & Alerting Setup
1. Provisioned the `Primary-Website-Health` check targeting the `/cafe` URI on `CafeInstance1`.
2. Configured a new Amazon SNS topic and subscribed an administrative email address to receive CloudWatch alarm notifications.
3. Confirmed the SNS subscription and verified that the Health Check status transitioned to **`Healthy`**.

### Phase 3: Failover Routing Configuration
1. Navigated to the Route 53 Public Hosted Zone containing the authoritative `NS` and `SOA` records.
2. Created the **Primary** and **Secondary** failover `A` records pointing to their respective EC2 instance IPs with a 15-second TTL.
3. Tested initial DNS resolution via `http://www.<domain>/cafe` and confirmed traffic routed exclusively to the primary instance (`us-west-2a`).

### Phase 4: Outage Simulation (Failover Test)
1. **Simulated an AZ Failure:** Manually stopped `CafeInstance1` via the EC2 Management Console.
2. **Health Check Detection:** Observed the Route 53 health check fail after 2 consecutive 10-second intervals, transitioning the status to **`Unhealthy`**.
3. **Automated Failover:** Refreshed the application URL (`http://www.<domain>/cafe`). Without changing the URL, the application seamlessly reloaded from `CafeInstance2` in **`us-west-2b`**.
4. **Incident Notification:** Received the automated `ALARM: "Primary-Website-Health..."` email via Amazon SNS confirming the primary endpoint failure.

## 🧠 Key Takeaways & Best Practices

* **Active-Passive Resilience:** Route 53 Failover routing provides a cost-effective and straightforward mechanism to maintain business continuity across Availability Zones or Regions.
* **Application-Layer Probing:** Checking a specific path (`/cafe`) rather than a simple ICMP ping ensures that web server crashes (e.g., Apache/PHP failure) are detected even if the underlying OS is still running.
* **DNS Caching Awareness:** Tuning the DNS `TTL` down to `15 seconds` is critical for failover architectures; default TTLs (such as 300s or 3600s) would leave users stranded on an unreachable IP during an outage.
