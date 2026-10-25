# AWS Stress Testing and Observability Lab

## Overview

This hands-on lab explores how to deploy a simple distributed application on AWS and evaluate its behavior under load. The environment includes a frontend service, a backend service, and an Amazon RDS database. Amazon CloudWatch is used to collect metrics and logs, while AWS X-Ray provides distributed tracing for application requests.

The goal is to understand how workload affects compute resources, how to monitor application and database performance, and how observability tools help identify bottlenecks.

> **Lab title note:** The existing folder is named `Using Chef Solo on Linux for AWS`, but the steps documented here describe an AWS stress-testing and observability lab rather than a Chef Solo exercise. Consider renaming the folder to `AWS Stress Testing and Observability` so its name matches the content.

## Objectives

By completing this lab, you will practice how to:

- Set up frontend and backend application components on Amazon EC2.
- Connect to EC2 instances using AWS Systems Manager Session Manager.
- Configure the Amazon CloudWatch agent to collect logs and system metrics.
- Instrument and explore distributed requests with AWS X-Ray.
- Build a CloudWatch dashboard for EC2, custom agent, and RDS metrics.
- Create a CloudWatch alarm with an Amazon SNS email notification.
- Generate CPU and HTTP request load using `stress`, ApacheBench, and Taurus/JMeter.
- Use metrics, logs, tracing, and load-test results to reason about service performance and bottlenecks.

## Architecture

The lab uses three main application tiers:

- **Frontend:** A Node.js and Express application that accepts requests and calls the backend.
- **Backend:** A Java application that exposes CPU-work and database endpoints.
- **Database:** An Amazon RDS PostgreSQL instance queried by the backend.

Monitoring and testing components:

- **Amazon CloudWatch:** Metrics, log groups, dashboards, and alarms.
- **Amazon SNS:** Email notification for an alarm.
- **AWS X-Ray:** Tracing across application services.
- **AWS Systems Manager Session Manager:** Administrative access to EC2 without requiring direct inbound SSH access.
- **Stress / ApacheBench / Taurus with JMeter:** Load generation and performance testing.

### High-level request flow

```text
Load Generator / Client
          |
          v
+--------------------+
| Frontend EC2       |
| Node.js / Express  |
+--------------------+
          |
          v
+--------------------+
| Backend EC2        |
| Java Service       |
+--------------------+
          |
          v
+--------------------+
| Amazon RDS          |
| PostgreSQL          |
+--------------------+

Observability:
  EC2 + applications ---> CloudWatch Agent ---> CloudWatch Logs / Metrics
  Frontend / Backend  ---> AWS X-Ray ---------> Trace Map
  CloudWatch Alarm    ---> Amazon SNS --------> Email notification
```

This diagram illustrates the intended flow of the lab. The exact resources and network configuration depend on the environment provisioned for the exercise.

## Services and Tools

| Service or Tool | Purpose |
|---|---|
| Amazon EC2 | Hosts the frontend and backend services |
| Amazon RDS for PostgreSQL | Provides the relational database |
| AWS Systems Manager Session Manager | Provides managed shell access to instances |
| Amazon CloudWatch Agent | Collects configured host metrics and log files |
| Amazon CloudWatch | Stores metrics and logs, displays dashboards, and evaluates alarms |
| Amazon SNS | Sends alarm notifications to email subscribers |
| AWS X-Ray | Helps trace requests through the distributed application |
| Java 17 and Maven | Build and run the backend service |
| Node.js and Express | Implement the frontend service |
| ApacheBench (`ab`) | Generates HTTP benchmark requests |
| Taurus (`bzt`) and JMeter | Runs load tests from YAML configuration |
| `stress` | Generates CPU load for controlled experiments |

## Prerequisites

Before starting, ensure that you have:

- Access to an AWS account and the lab environment.
- EC2 instances and an RDS database provisioned for the lab, or permission to create them.
- Appropriate IAM permissions for Systems Manager, CloudWatch, X-Ray, and SNS.
- Working Systems Manager Session Manager access to the target instances.
- Network connectivity from the frontend to the backend, and from the backend to RDS.
- Basic familiarity with Linux, AWS networking, Java, Node.js, and command-line tools.

Use the operating-system package manager and installation instructions appropriate for the distribution and software versions provided by your environment.

## Implementation

### 1. Inspect the lab resources

Open the AWS CloudFormation console and inspect the lab stack outputs. Record the resource details you need, such as the RDS endpoint and the relevant EC2 instance names or private IP addresses.

Open the EC2 console and identify the frontend and backend instances. Use Systems Manager Session Manager to connect where it is configured.

**Do not commit environment-specific instance IDs, public IP addresses, RDS endpoints, credentials, or other sensitive values to this repository.** Use placeholders in documentation and supply actual values through your lab environment.

### 2. Configure CloudWatch Agent

On each target EC2 instance, install the CloudWatch agent using the instructions for its operating system. Create an agent configuration that collects the appropriate application log files and host metrics.

The original lab configuration collects logs from paths such as:

- `/home/ssm-user/backend/backend.log`
- `/home/ssm-user/frontend/frontend.log`

It also configures metrics such as memory utilization, swap utilization, and disk utilization. Update file paths and permissions to match the applications and operating system in use.

Start the agent with the appropriate configuration file and verify its status. Confirm that log groups and metrics appear in CloudWatch before proceeding.

### 3. Configure AWS X-Ray

Install and configure the X-Ray daemon or the supported tracing components for the lab environment. The application must be instrumented with a compatible X-Ray SDK and configured with the permissions and network access needed to publish traces.

The objective is to trace requests across the frontend and backend and inspect service activity in the X-Ray trace map or the relevant CloudWatch application monitoring views.

### 4. Build and run the backend service

The backend sample application is a Java service that exposes endpoints for CPU work and a database query:

- `/cpu` — performs a short CPU-intensive operation.
- `/db` — connects to PostgreSQL and queries the current database time.

The application is built with Maven and uses environment variables for database configuration. Supply these values securely in the lab environment rather than committing credentials to source control.

Example placeholder configuration:

```bash
export DB_HOST="<your-rds-endpoint>"
export DB_NAME="<your-database-name>"
export DB_USER="<your-database-user>"
export DB_PASS="<your-database-password>"
```

Build and run the application using the project's Maven configuration and the appropriate Java command. Verify each endpoint locally before moving on.

Do not use the example placeholders literally. Confirm that the database hostname, database name, username, and password match your authorized lab environment.

### 5. Build and run the frontend service

The frontend sample uses Node.js and Express to provide endpoints that simulate CPU work and call the backend database endpoint.

Configure the backend's private address and port through environment variables:

```bash
export BACKEND_HOST="<your-backend-private-ip>"
export BACKEND_PORT="8080"
```

Install the Node.js dependencies, start the frontend application, and test the endpoints locally. Confirm that the frontend can reach the backend over the configured network path.

### 6. Create a CloudWatch dashboard

Create a CloudWatch dashboard that brings together the important metrics from the lab. Useful views include:

- EC2 CPU utilization for the frontend and backend.
- CloudWatch Agent metrics for memory, swap, and disk utilization.
- EC2 CPU credit metrics where supported by the selected instance type.
- RDS CPU utilization, burst balance where applicable, read/write latency, and other relevant database metrics.

Check that the graphs reference the correct current resources. Select a period appropriate to the available metric resolution and the monitoring objective. Detailed monitoring and high-resolution metrics may have additional charges or service-specific limitations.

Save the dashboard and use it throughout the load tests.

### 7. Create a CloudWatch alarm

Create an alarm for a metric and threshold appropriate to the experiment. For example, the lab may use an EC2 CPU utilization threshold to demonstrate alarm evaluation.

Configure an SNS topic and an email subscription, then confirm the subscription using the message sent to the chosen address. Verify the alarm state and notification path before relying on it.

The threshold, evaluation periods, and metric period should be selected for the actual test rather than treated as production recommendations.

### 8. Generate CPU load

Use a controlled CPU workload on the target instance. For example, a short `stress` run can help demonstrate how CPU utilization changes while the dashboard is open.

Start with a modest number of workers and a short duration, then inspect the results before increasing the workload. Observe CPU utilization and any applicable CPU credit metrics.

### 9. Generate HTTP request load

Use ApacheBench or the Taurus/JMeter configuration to send test requests to the application endpoints.

The lab includes separate test scenarios for the backend's CPU and database endpoints. Taurus configuration files can define the target endpoint, concurrency, ramp-up time, and test duration.

Review the test summary and monitoring data together. Relevant measurements include:

- Request throughput.
- Response time and latency percentiles.
- Error counts and HTTP status codes.
- Frontend and backend CPU utilization.
- Database load and query latency.
- Any resource saturation or throttling signals.

Avoid launching high-concurrency tests until the effects of smaller runs are understood.

### 10. Explore distributed tracing

Use the available X-Ray and CloudWatch application monitoring views to inspect traces generated by the frontend and backend. Compare trace timing with CloudWatch metrics and application logs to identify which service or dependency contributes to increased latency or errors.

A useful investigation correlates a slow request with its trace, the corresponding service logs, and the resource metrics over the same time window.

## Observations and Analysis

After each test, record the workload parameters and observed results. Do not assume the outcome in advance; results depend on instance types, CPU credits, concurrency, network conditions, database performance, and the application implementation.

Suggested experiment log:

| Experiment | Workload parameters | Metrics to inspect | Findings |
|---|---|---|---|
| Backend CPU test | Workers and duration used | Backend CPU utilization, CPU credits where applicable | Record the observed effect |
| Frontend CPU test | Workers and duration used | Frontend CPU utilization, CPU credits where applicable | Record the observed effect |
| Database endpoint test | Concurrent users and duration | RDS CPU, read/write latency, backend response times | Record the observed effect |
| HTTP benchmark | Request count and concurrency | Throughput, latency percentiles, errors | Record the observed effect |
| End-to-end trace review | Endpoint and test window | Trace segments, service latency, logs, metrics | Record identified bottlenecks |

### Questions to Consider

- Which tier becomes the bottleneck first under the tested workload?
- Does increased frontend CPU utilization correspond to increased backend traffic?
- How do database query latency and RDS metrics change during the database test?
- Do traces show that time is spent in the frontend, backend, or database call?
- Do alarm thresholds and evaluation periods provide useful notifications without excessive noise?
- Which additional metrics or logs would make the cause of a failure easier to identify?

## Troubleshooting

| Symptom | Things to Check |
|---|---|
| Session Manager connection fails | Instance registration, SSM Agent health, IAM instance profile permissions, and network access to Systems Manager endpoints |
| CloudWatch logs do not appear | Agent status and configuration, file paths, file permissions, log group permissions, and network connectivity |
| Custom metrics are missing | Agent configuration, namespace, metric dimensions, and collection interval |
| Backend cannot connect to RDS | RDS endpoint, database credentials, security group rules, network routes, and database availability |
| Frontend cannot call backend | Backend process status, configured private IP and port, security group rules, and network reachability |
| X-Ray traces are missing | SDK and daemon configuration, sampling, IAM permissions, and application instrumentation |
| Alarm email is not received | SNS subscription confirmation, alarm configuration, and alarm state |
| Load test returns errors | Application logs, target endpoint, process health, connection settings, and resource capacity |
| Taurus command is not found | Python installation method and whether the executable directory is on `PATH` |

## Security and Cost Considerations

- Use least-privilege IAM policies for EC2, Systems Manager, CloudWatch, X-Ray, and SNS.
- Prefer Systems Manager Session Manager instead of opening SSH to broad IP ranges.
- Do not commit passwords, access keys, session tokens, private keys, or real environment-specific credentials to Git.
- Store secrets in an appropriate secret-management mechanism or pass them securely at runtime.
- Restrict security group ingress to the minimum access required for the experiment.
- Load-test only resources you own or are explicitly authorized to test.
- Use short test durations and increase concurrency gradually. High load can degrade the lab and consume resources.
- CloudWatch metrics, logs, detailed monitoring, data transfer, RDS, and other AWS resources may incur charges.
- Stop or delete lab resources when they are no longer needed, after preserving any required results.

## Cleanup

When you finish:

1. Stop load generators and background application processes.
2. Remove temporary files and test artifacts that are no longer required.
3. Review CloudWatch dashboards, alarms, log groups, and SNS topics created for the lab.
4. Stop or terminate EC2 resources and delete other lab resources according to the lab's cleanup instructions.
5. Confirm that no unneeded chargeable resources remain.

Follow the lab provider's instructions before deleting shared or managed resources.

## Key Takeaways

This lab demonstrates how application performance testing and observability complement one another:

- **Metrics** show resource utilization and performance trends.
- **Logs** provide event details and error context.
- **Distributed traces** help locate latency across service boundaries.
- **Load tests** provide a repeatable way to observe system behavior under demand.
- **Alarms** can notify operators when a monitored condition crosses a configured threshold.

The most useful conclusions come from correlating these signals rather than interpreting any single metric in isolation.

## References

- [Amazon EC2 documentation](https://docs.aws.amazon.com/ec2/)
- [AWS Systems Manager Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)
- [Amazon CloudWatch documentation](https://docs.aws.amazon.com/cloudwatch/)
- [CloudWatch Agent documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html)
- [AWS X-Ray documentation](https://docs.aws.amazon.com/xray/)
- [Amazon SNS documentation](https://docs.aws.amazon.com/sns/)
- [Apache Maven](https://maven.apache.org/)
- [ApacheBench documentation](https://httpd.apache.org/docs/2.4/programs/ab.html)
- [Taurus documentation](https://gettaurus.org/)

## Disclaimer

This repository documents hands-on learning and lab experimentation. Validate commands and configurations against your own environment and the current official documentation before using them elsewhere. Resource names, software versions, performance results, and costs vary by environment.
