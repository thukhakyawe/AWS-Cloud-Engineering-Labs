# AWS Stress Testing and Observability Lab

## Overview

This hands-on lab deploys a small distributed application on AWS and observes it under CPU and HTTP load. The environment uses a Node.js frontend, a Java backend, and an Amazon RDS for PostgreSQL database. CloudWatch collects host metrics and logs, X-Ray helps visualize distributed requests, and load generators create controlled workloads.

> **Scope:** This is a stress-testing and observability lab, not a Chef Solo exercise.

## Objectives

- Connect to EC2 with AWS Systems Manager Session Manager.
- Configure CloudWatch Agent for logs and Linux metrics.
- Deploy Java and Node.js services and connect the backend to PostgreSQL.
- Build CloudWatch dashboards and alarms with SNS notifications.
- Generate CPU and HTTP workloads with stress, ApacheBench, and Taurus/JMeter.
- Correlate metrics, logs, traces, and test results to investigate bottlenecks.

## Architecture

~~~text
Frontend EC2 (Node.js / Express :3000)
             |
             v
Backend EC2 (Java / Spark :8080)
             |
             v
Amazon RDS for PostgreSQL

CloudWatch Agent --> CloudWatch Logs and Metrics
X-Ray instrumentation --> Trace Map
CloudWatch Alarm --> SNS --> Email
stress / ab / Taurus-JMeter --> Application endpoints
Session Manager --> EC2 shell access
~~~

## Prerequisites

- Access to the AWS account and lab environment.
- Frontend and backend EC2 instances and an RDS PostgreSQL database.
- Working Session Manager access.
- Appropriate IAM permissions for Systems Manager, CloudWatch, and tracing.
- Network access from frontend to backend and backend to RDS.
- Basic Linux, Java, Node.js, and AWS networking knowledge.

**Security:** Never commit actual instance IDs, IP addresses, RDS endpoints, passwords, private keys, access keys, or tokens to a public repository. If a real credential was committed publicly, rotate it; removing it from the latest file does not remove it from Git history.

**Cost:** Load testing, CloudWatch metrics, log ingestion, traces, detailed monitoring, EC2, and RDS may incur charges. Use an authorized test environment and clean up resources after the lab.

## Step 1: Inspect the Lab Resources

1. Open CloudFormation in the AWS console and select the lab stack.
2. Open the Outputs tab and record the resource information you need in private notes.
3. Open EC2 → Instances and identify the frontend and backend instances by their Name tags.
4. Record the private IP addresses for application configuration.
5. Use Session Manager for shell access when available rather than opening SSH to the internet.

Use placeholders in public documentation:

| Value | Placeholder |
|---|---|
| Frontend private IP | <frontend-private-ip> |
| Backend private IP | <backend-private-ip> |
| RDS endpoint | <your-rds-endpoint> |
| Database name | <your-database-name> |
| Database user | <your-database-username> |

## Step 2: Configure CloudWatch Agent

Perform these steps on both frontend and backend instances.

### 2.1 Connect and install

In EC2, select the instance, choose Connect, open Session Manager, and select Connect. Then check the OS and install the agent:

~~~bash
cat /etc/os-release
whoami
df -h
sudo dnf install -y amazon-cloudwatch-agent
~~~

The command uses dnf for an RPM-based lab AMI. Use the package manager that matches your operating system.

### 2.2 Create the configuration

~~~bash
cat <<'EOF' > /home/ssm-user/cwagent-config.json
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/home/ssm-user/backend/backend.log",
            "log_group_name": "/sre/backend",
            "log_stream_name": "{instance_id}"
          },
          {
            "file_path": "/home/ssm-user/frontend/frontend.log",
            "log_group_name": "/sre/frontend",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  },
  "metrics": {
    "append_dimensions": {
      "InstanceId": "INSTANCE_DIMENSION_TOKEN"
    },
    "metrics_collected": {
      "mem": {
        "measurement": ["mem_used_percent"],
        "metrics_collection_interval": 60
      },
      "swap": {
        "measurement": ["swap_used_percent"],
        "metrics_collection_interval": 60
      },
      "disk": {
        "measurement": ["used_percent"],
        "metrics_collection_interval": 60
      }
    }
  }
}
EOF
~~~

Replace the placeholder with the CloudWatch Agent EC2 instance dimension token:

~~~bash
python3 - <<'PY'
from pathlib import Path
p = Path("/home/ssm-user/cwagent-config.json")
p.write_text(p.read_text().replace("INSTANCE_DIMENSION_TOKEN", chr(36) + "{aws:InstanceId}"))
PY
~~~

Validate the JSON:

~~~bash
python3 -m json.tool /home/ssm-user/cwagent-config.json
~~~

The configuration collects memory, swap, and disk utilization every 60 seconds and defines log groups for the frontend and backend. A log path must exist and be readable before its contents can be collected.

### 2.3 Start and verify the agent

~~~bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -c file:/home/ssm-user/cwagent-config.json \
  -s

sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
ps aux | grep '[a]mazon-cloudwatch-agent'
sudo tail -n 100 /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
~~~

The instance profile must have permissions to publish metrics and logs. After the application log files exist and the agent has had time to publish data, check CloudWatch → Log groups for /sre/backend and /sre/frontend and Metrics → CWAgent for the instance metrics.

### 2.4 Install X-Ray daemon

On a compatible RPM-based instance, select the ARM64 package when required:

~~~bash
ARCH=""
if [[ "$(uname -m)" == "aarch64" ]]; then
  ARCH="-arm64"
fi

sudo dnf install -y "https://s3.us-west-2.amazonaws.com/aws-xray-assets.us-west-2/xray-daemon/aws-xray-daemon$ARCH-3.x.rpm"
sudo systemctl enable --now xray
sudo systemctl status xray
~~~

If the package URL or service name no longer works for the AMI, consult the current AWS package instructions rather than disabling package integrity checks.

> **Lifecycle note:** AWS X-Ray SDKs and the daemon entered maintenance mode on February 25, 2026, and support is scheduled to end on February 25, 2027. These steps remain for compatibility with this exercise. For new applications, follow AWS's OpenTelemetry migration guidance.

