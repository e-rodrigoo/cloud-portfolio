# ☁️ AWS Infrastructure as Code Automation with CloudFormation

## 📌 Project Overview

This project demonstrates automated cloud infrastructure provisioning and lifecycle management using **Infrastructure as Code (IaC)** with **AWS CloudFormation**. Rather than manually configuring resources through the management console, a declarative **YAML** template was engineered and iteratively updated to deploy a complete network and compute environment in a consistent, repeatable manner.

The initial stack deployment provisioned a custom **Amazon VPC** (`10.0.0.0/16`), Public Subnet, Internet Gateway, Public Route Table, and stateful **Security Group** (`AppSecurityGroup`). Through incremental stack updates and Change Set validations, the template was expanded to provision an **Amazon S3** bucket and an **Amazon EC2** instance (`t3.micro` named `App Server`) dynamically referencing the latest **Amazon Linux 2 AMI** via **AWS Systems Manager Parameter Store** and intrinsic functions (`!Ref`). Finally, automated stack teardown was executed to verify clean termination of all dependent resources.

## 🏗️ Architecture Overview

```text
+-------------------------------------------------------------------------+
|                           AWS Cloud Environment                         |
|                                                                         |
|  +-------------------------------------------------------------------+  |
|  |                  AWS CloudFormation (YAML Template)               |  |
|  | - Stack Name: Lab                                                 |  |
|  | - Intrinsic Functions (!Ref) & SSM Parameter AMI Lookup           |  |
|  +---------+------------------------+------------------------+-------+  |
|            | (Provisions)           | (Updates Stack)        |          |
|            v                        v                        |          |
|  +---------+-------------+  +-------+--------------+         |          |
|  |      Amazon S3        |  |   AWS SSM Parameter  |         |          |
|  | - Resource: S3Bucket  |  | - Latest AL2 AMI ID  |         |          |
|  +-----------------------+  +-------+--------------+         |          |
|                                     | (Resolves ImageId)     |          |
|                                     v                        v          |
|  +-------------------------------------------------------------------+  |
|  |                        Custom VPC (10.0.0.0/16)                   |  |
|  |                                                                   |  |
|  |  +--------------------------+      +---------------------------+  |  |
|  |  |     Internet Gateway     |<---->|     Public Route Table    |  |  |
|  |  +--------------------------+      +-------------+-------------+  |  |
|  |                                                  |                |  |
|  |  +-----------------------------------------------+-------------+  |  |
|  |  |                         Public Subnet                       |  |  |
|  |  |                                                             |  |  |
|  |  |  +-------------------------------------------------------+  |  |  |
|  |  |  |           Security Group (AppSecurityGroup)           |  |  |  |
|  |  |  |                                                       |  |  |  |
|  |  |  |  +-------------------------------------------------+  |  |  |  |
|  |  |  |  |          Amazon EC2 Instance (t3.micro)         |  |  |  |  |
|  |  |  |  | - Tag: Name = App Server                        |  |  |  |  |
|  |  |  |  | - AMI: Dynamic SSM Parameter Reference          |  |  |  |  |
|  |  |  |  +-------------------------------------------------+  |  |  |  |
|  |  |  +-------------------------------------------------------+  |  |  |
|  |  +-------------------------------------------------------------+  |  |
|  +-------------------------------------------------------------------+  |
+-------------------------------------------------------------------------+
```

## 🛠️ Technologies & AWS Services Used

* **AWS CloudFormation**: Orchestrates the creation, incremental updating (Change Sets), dependency resolution, and automated teardown of all infrastructure resources defined in YAML templates.
* **Amazon VPC**: Provides an isolated virtual network (`10.0.0.0/16`) complete with a Public Subnet, Internet Gateway, and Public Route Table provisioned via template definitions.
* **Amazon EC2 & Security Groups**: Deploys a `t3.micro` compute instance (`App Server`) associated with the `AppSecurityGroup` and `PublicSubnet` using CloudFormation `!Ref` intrinsic functions.
* **Amazon S3**: Provisions an object storage bucket (`AWS::S3::Bucket`) added seamlessly through an in-place CloudFormation stack update without disrupting existing network resources.
* **AWS Systems Manager (SSM) Parameter Store**: Dynamically resolves the latest region-specific Amazon Linux 2 AMI ID (`/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2`) using the `AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>` parameter type.

## 🚀 Implementation Steps

1. **Initial Network Stack Deployment**: Analyzed the base YAML template (`Parameters`, `Resources`, and `Outputs` sections) and deployed the `Lab` stack in AWS CloudFormation to provision the custom VPC, public subnet, Internet Gateway, route table, and security group (`CREATE_COMPLETE`).
2. **In-Place Stack Update with Amazon S3**: Extended the YAML template under the `Resources` block by defining an `AWS::S3::Bucket` resource. Generated a Change Set preview to confirm zero impact on running network components and executed the stack update (`UPDATE_COMPLETE`).
3. **Dynamic AMI Lookup & EC2 Provisioning**: Added the `AmazonLinuxAMIID` parameter referencing `/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2` in SSM Parameter Store. Defined an `AWS::EC2::Instance` resource (`t3.micro`, tagged `App Server`) using `!Ref` to dynamically link `AmazonLinuxAMIID`, `AppSecurityGroup`, and `PublicSubnet`.
4. **Automated Infrastructure Teardown**: Initiated a full stack deletion in CloudFormation (`DELETE_IN_PROGRESS` to completion) and verified in the EC2 and VPC consoles that the `App Server` instance (`Terminated`), S3 bucket, and custom networking components were cleanly deprovisioned in reverse dependency order.

## 📸 Project Verification & Screenshots

Detailed visual evidence of the initial stack deployment, automated resource cleanup upon stack deletion, and EC2 instance termination verification is available in the `screenshots` directory:

👉 **[View Implementation Screenshots](./screenshots/README.md)**

## 🧠 Key Takeaways & Best Practices

* **Declarative Dependency Management**: CloudFormation automatically determines the optimal provisioning and teardown sequence (for example, creating the VPC before the Subnet and Route Table, and terminating the EC2 instance before deleting its Subnet and VPC).
* **Non-Destructive Incremental Updates**: Updating an existing stack with Change Sets allows engineers to preview and deploy new resources (such as an `AWS::S3::Bucket` or `AWS::EC2::Instance`) without redeploying or disrupting unchanged infrastructure.
* **Region-Agnostic Templates via SSM Parameters**: Querying `/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2` via `AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>` eliminates hardcoded AMI IDs, making templates portable across AWS Regions and ensuring instances launch with patched images.
* **Zero-Orphan Resource Cleanup**: Managing infrastructure lifecycles through a single CloudFormation stack guarantees that deleting the stack removes all associated compute, storage, and networking resources, preventing configuration drift and unexpected cloud costs.
