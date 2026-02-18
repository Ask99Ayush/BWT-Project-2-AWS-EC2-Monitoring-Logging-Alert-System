# AWS EC2 Monitoring, Logging & Alert System

---

## Table of Contents
- [1. Project Overview](#1-project-overview)
- [2. Why Monitoring Is Critical in Production](#2-why-monitoring-is-critical-in-production)
- [3. Monitoring vs Logging vs Alerting](#3-monitoring-vs-logging-vs-alerting)
- [4. High-Level Architecture](#4-high-level-architecture)
- [5. AWS Services Used](#5-aws-services-used)
- [6. System Architecture & Data Flow](#6-system-architecture--data-flow)
- [7. EC2 Configuration Overview](#7-ec2-configuration-overview)
- [8. CloudWatch Metrics & Alarms](#8-cloudwatch-metrics--alarms)
- [9. SNS Alerting System](#9-sns-alerting-system)
- [10. Logging Architecture (Conceptual)](#10-logging-architecture-conceptual)
- [11. Log Archival to S3 Using Firehose](#11-log-archival-to-s3-using-firehose)
- [12. End-to-End Testing & Validation](#12-end-to-end-testing--validation)
- [13. Security & Best Practices](#13-security--best-practices)
- [14. Production Relevance](#14-production-relevance)
- [15. Interview Explanation](#15-interview-explanation)
- [16. Resume-Ready Project Description](#16-resume-ready-project-description)

---

## 1. Project Overview

### What This Project Is
This project implements a production-grade monitoring, logging, and alerting system for an Amazon EC2 instance using AWS-native managed services. It continuously observes infrastructure health, detects abnormal behavior, and notifies administrators in real time, while also preparing a scalable architecture for centralized log storage and long-term retention.

### Why This Project Exists (Problem Statement)
In real-world systems, infrastructure failures rarely occur as sudden crashes. Instead, systems degrade gradually—CPU saturation, increased latency, or silent request drops. Without monitoring and alerting, teams learn about issues only after users are impacted. This project addresses the core operational question:

**How do we know something is wrong with our server before users do?**

### Real-World Relevance
This architecture mirrors how baseline observability is introduced in production environments:
- Metrics first
- Alerts second
- Logs as supporting evidence

It avoids unnecessary complexity while remaining extensible.

### Production Mindset
- Uses managed AWS services only
- No agents or custom scripts
- No hardcoded credentials
- Designed to scale and evolve

---

## 2. Why Monitoring Is Critical in Production

Without monitoring:
- There is no visibility into system health
- Incidents are discovered reactively
- Root cause analysis becomes guesswork
- Downtime and business impact increase

Monitoring enables:
- Proactive detection of issues
- Faster incident response
- Capacity planning
- Reliability and uptime guarantees

Alerting ensures that monitoring data leads to action. Metrics without alerts require humans to watch dashboards continuously, which is not realistic in production environments.

---

## 3. Monitoring vs Logging vs Alerting

### Monitoring
- Tracks system health using numeric metrics
- Example: CPUUtilization = 78%
- Answers: *Is something wrong right now?*

### Logging
- Records detailed event data
- Example: error messages, access logs
- Answers: *What exactly happened and why?*

### Alerting
- Notifies humans or systems
- Example: email when CPU > 70% for 10 minutes
- Answers: *Who needs to act now?*

### How They Work Together
Monitoring detects symptoms, alerting triggers response, and logging supports investigation. This project prioritizes metrics and alerts first, treating logging as an extension rather than a prerequisite.

---

## 4. High-Level Architecture

### Text-Based Architecture Diagram

```

EC2 Instance
|
v
CloudWatch Metrics
|
v
CloudWatch Alarm
|
v
SNS Topic
|
v
Email Notification

```

### Monitoring-First Approach
Metrics are collected automatically and evaluated continuously. Alerts are triggered based on sustained abnormal behavior, not transient spikes. Logs are optional and used for deeper analysis.

---

## 5. AWS Services Used

### Amazon EC2
Provides the compute layer and represents the workload being monitored. The instance itself is not application-heavy; its purpose is to generate infrastructure metrics.

### Amazon CloudWatch (Metrics & Alarms)
Acts as the core observability service:
- Collects EC2 metrics automatically
- Evaluates thresholds using alarms
- Triggers actions when alarms change state

### Amazon SNS
Delivers alerts reliably using a publish/subscribe model. It decouples alarm logic from notification delivery and supports multiple subscribers and protocols.

### Amazon CloudWatch Logs
Serves as the centralized log storage and search layer for AWS-managed logs. It is used conceptually in this project without installing agents.

### Amazon S3
Provides durable, low-cost, long-term storage for logs, suitable for compliance and audit requirements.

### Amazon Kinesis Data Firehose
Acts as the managed delivery pipeline that streams logs from CloudWatch Logs into S3, handling buffering, retries, compression, and scaling.

### Why CloudWatch Agent Is NOT Used
- Default EC2 metrics are already available
- Agents add operational overhead and maintenance
- This project focuses on baseline, agentless monitoring
- Many production systems start without agents and add them only when needed

---

## 6. System Architecture & Data Flow

### Step-by-Step Flow
1. EC2 instance runs in AWS.
2. AWS hypervisor collects default metrics.
3. CloudWatch stores metric data points.
4. CloudWatch Alarm evaluates metrics over time.
5. Alarm transitions state when thresholds are breached.
6. SNS publishes notifications to subscribers.
7. Administrators receive email alerts.
8. AWS-managed logs are stored in CloudWatch Logs.
9. Logs are streamed via Firehose into S3 for long-term retention.

No agents, no applications, and no custom scripts are required.

---

## 7. EC2 Configuration Overview

### Instance Purpose
Represents a production server whose health must be monitored.

### Instance Type
- Recommended: t2.micro or t3.micro
- Free-tier eligible
- Sufficient for alarm testing

### Security Group Design
- Inbound: SSH (port 22) restricted to a single IP
- Outbound: Allow all, required for AWS API access

### IAM Role Attachment
- Uses role-based access
- No access keys stored on the instance
- Follows least-privilege principles

### Region Consistency
All services must be deployed in the same region to ensure proper metric visibility and log streaming.

---

## 8. CloudWatch Metrics & Alarms

### CPUUtilization Deep Dive
CPUUtilization represents the percentage of CPU capacity used over a given period. Sustained high CPU usage is a common indicator of system stress and performance degradation.

### Threshold Logic
Example configuration:
- Threshold: CPU > 70%
- Period: 5 minutes
- Evaluation periods: 2

This means CPU must remain above 70% for 10 continuous minutes before triggering an alarm, reducing false positives from short spikes.

### Alarm States
- OK: Metric within normal range
- ALARM: Threshold breached
- INSUFFICIENT_DATA: Not enough data yet

---

## 9. SNS Alerting System

### Why SNS
SNS provides reliable, scalable, and decoupled alert delivery. CloudWatch alarms publish events to SNS, and SNS handles distribution.

### Alarm to Email Flow
```

CloudWatch Alarm
|
v
SNS Topic
|
v
Email Subscription

```

### Subscription Confirmation
Email subscriptions must be confirmed before alerts are delivered. This prevents accidental or unauthorized notifications.

---

## 10. Logging Architecture (Conceptual)

### CloudWatch Logs
CloudWatch Logs stores log events generated by AWS-managed services such as CloudTrail, VPC Flow Logs, and load balancers.

### Why EC2 Does Not Generate Logs by Default
EC2 is a virtual machine without a built-in logging mechanism. Without a log agent or AWS-managed log source, no OS or application logs are sent to CloudWatch Logs.

### Retention
Logs should have defined retention periods to balance troubleshooting needs and cost.

---

## 11. Log Archival to S3 Using Firehose

### Why Firehose Is Required
CloudWatch Logs cannot stream logs directly to S3 in real time. Kinesis Data Firehose is the AWS-supported managed service for this purpose.

### Final Architecture
```

CloudWatch Logs
|
v
Subscription Filter
|
v
Kinesis Data Firehose
|
v
Amazon S3 (Compressed, Partitioned)

```

### Benefits
- Fully managed
- Near real-time delivery
- Built-in buffering and retries
- Cost-efficient long-term storage

---

## 12. End-to-End Testing & Validation

### CPU Alarm Testing
- Generate CPU load using a simple command
- Verify metric increase and alarm transition

### SNS Email Verification
- Confirm email subscription
- Trigger alarm and verify email delivery

### Log Delivery Verification
- Check S3 bucket for compressed log files
- Validate folder structure and timestamps

### Success Criteria
All components function together without manual intervention.

---

## 13. Security & Best Practices

- IAM roles used everywhere
- No hardcoded credentials
- Least-privilege access policies
- Private, encrypted S3 buckets
- Defined log retention and lifecycle policies
- Alert thresholds tuned to avoid noise

---

## 14. Production Relevance

This architecture is widely used in real organizations as baseline observability:
- Scales automatically with AWS-managed services
- Suitable for single-account and multi-account environments
- Forms the foundation for more advanced monitoring and logging systems

---

## 15. Interview Explanation

### Two-Minute Explanation
This project implements a production-grade EC2 monitoring, alerting, and logging system using AWS-native services. EC2 metrics are collected by CloudWatch, evaluated by alarms, and alerts are delivered via SNS. Logs from AWS-managed sources are centralized in CloudWatch Logs and archived to S3 using Kinesis Data Firehose. The system avoids agents, uses IAM roles, and follows least-privilege and cost-optimized retention practices.

### Common Interview Questions
- Why no CloudWatch Agent?  
  To reduce operational overhead and rely on managed metrics.
- Why Firehose?  
  CloudWatch Logs cannot stream directly to S3.
- How does this scale?  
  All services are fully managed and scale automatically.

---

## 16. Resume-Ready Project Description

Designed and implemented an AWS-based EC2 monitoring, logging, and alerting system using CloudWatch, SNS, S3, and Kinesis Data Firehose. Built a fully managed observability pipeline with CPU-based alarms, automated email alerts, centralized log ingestion, and long-term log archival without agents or applications. Applied IAM least-privilege principles, secure storage, and cost-optimized retention strategies.

---
