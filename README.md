# NexusCart CloudOps Infrastructure Monitoring & Alerting

## Overview
This project establishes a comprehensive monitoring, telemetry, and automated alerting solution for heavily utilized application servers within the NexusCart infrastructure. The objective is to achieve deep situational awareness of system health and application uptime by deploying custom telemetry agents, building centralized dashboards, and configuring automated remediation pipelines.

___
## Problem Statement

As application traffic scales, relying solely on default host-level metrics provides an incomplete picture of system health. Cloud operations teams lack visibility into sub-resource telemetry (like per-CPU core or specific memory utilization) and cannot proactively respond to application failures because there is no automated notification system tied to specific performance thresholds.

___
## Problem Reframing & Requirements

To achieve operational maturity and reduce Mean Time To Resolution (MTTR), the infrastructure requires:
* Automated deployment of telemetry agents across the Amazon EC2 fleet without manual SSH access.
* Customized dashboards that filter out noise and display only mission-critical metrics.
* Automated alerting mechanisms that notify the operations team immediately when system reachability fails or metric thresholds are breached.
* A synthetic monitoring solution (Canary) to continuously verify that web endpoints are returning the correct, expected content.

___
## Solution & Tool Tradeoffs

The architecture utilizes Amazon CloudWatch as the centralized observability platform, coupled with AWS Systems Manager for fleet configuration.

**Tradeoffs Considered:**
* **AWS Systems Manager (SSM) vs. Manual Configuration:** Utilizing SSM Command Documents and Parameter Store allows the CloudWatch Agent to be installed and configured consistently across the entire fleet at scale, eliminating configuration drift and manual provisioning errors.
* **Lambda Canary vs. External Uptime Monitors:** Building a custom AWS Lambda function triggered by Amazon EventBridge keeps synthetic testing native to the AWS ecosystem, allowing seamless integration with CloudWatch Alarms and Amazon SNS without relying on third-party SaaS monitoring tools.

___
## Architecture

* Compute: Amazon EC2 (AppServer) running the CloudWatch agent for deep telemetry collection.
* Configuration Management: AWS Systems Manager (Run Command, Session Manager) and Parameter Store for agent configuration state.
* Observability: Amazon CloudWatch (Dashboards, Metrics, and Alarms).
* Notifications: Amazon Simple Notification Service (Amazon SNS) routing alerts to subscribed CloudOps engineers.
* Synthetic Testing: AWS Lambda (Canary function) triggered by Amazon EventBridge scheduled events.

___
## Data Assets & Metrics

The monitoring solution tracks specific telemetry data from the application servers:
* **Memory Utilization:** `mem_used_percent` metric from the custom `CWAgent` namespace to track application memory consumption.
* **Network Activity:** `NetworkIn` and `NetworkOut` metrics from the `AWS/EC2` namespace to monitor traffic loads.
* **System Health:** `StatusCheckFailed_System` to monitor underlying host reachability.
* **Application Uptime:** Custom Lambda error metrics measuring failed HTTP content validations.

___
## Pipeline Execution Flow
<img width="1920" height="1080" alt="myDashboard" src="https://github.com/user-attachments/assets/fbeaf88f-a5a9-4ba9-97d0-5195db76d038" />

1. **Agent Deployment:** Updated the SSM Agent and utilized Systems Manager Run Command (`AWS-ConfigureAWSPackage`) to install the CloudWatch Agent onto the EC2 instances.
2. **Configuration Management:** Started the CloudWatch Agent using a standardized JSON configuration document securely pulled from AWS Systems Manager Parameter Store.
3. **Dashboard Creation:** Built a customized CloudWatch Dashboard (`myDashboard`) containing Line and Stacked Area widgets to visualize memory utilization and network activity side-by-side.
4. **Infrastructure Alarming:** Configured a CloudWatch Alarm (`AppServerSystemsCheckAlarm`) tied to an Amazon SNS topic (`SysOpsTeamPager`) to automatically email engineers if the EC2 instance fails its system reachability check.
5. **Synthetic Canary Setup:** Deployed an AWS Lambda function triggered every 1 minute by EventBridge to scrape a target URL and verify the presence of a specific expected string.
6. **Application Alarming:** Created a secondary CloudWatch Alarm (`lambda-canary-alarm`) to monitor the Lambda Canary's error metrics, successfully testing it by intentionally misconfiguring the expected environment variable to trigger a simulated outage.

___
## Security & Roles

* IAM Roles: EC2 instances were provisioned with specific IAM profiles allowing them to securely interact with CloudWatch and Systems Manager without hardcoded credentials.
* Secure Access: Utilized AWS Systems Manager Session Manager to securely access the EC2 command line for alarm testing, completely bypassing the need for open inbound SSH (Port 22) rules.

___
## Business Outcomes & Analytical Outputs

* Proactive Incident Response: Successfully transformed a reactive infrastructure into a proactive one by routing critical threshold breaches directly to the CloudOps team via Amazon SNS.
* Synthetic Uptime Verification: Ensured the business application is not just online, but actively returning the correct content to users via continuous Lambda Canary testing.
* Standardized Fleet Management: Established a scalable pattern for deploying and configuring telemetry agents across hundreds of instances simultaneously using Systems Manager.
