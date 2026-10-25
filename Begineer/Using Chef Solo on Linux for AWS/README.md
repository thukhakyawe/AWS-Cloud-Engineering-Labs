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


## Step 3: Deploy the Java Backend Application

Run these commands on the backend EC2 instance.

### 3.1 Install Java, Maven, and Git

~~~bash
sudo dnf install -y java-17-amazon-corretto maven git
java -version
mvn -version
mkdir -p ~/backend/src/main/java/com/example
cd ~/backend
~~~

Java runs the service, Maven manages dependencies and builds the application, and Git is available for version-control tasks.

### 3.2 Create BackendService.java

~~~bash
cat <<'EOF' > ~/backend/src/main/java/com/example/BackendService.java
package com.example;

import com.amazonaws.xray.AWSXRay;
import com.amazonaws.xray.AWSXRayRecorderBuilder;
import com.amazonaws.xray.strategy.ContextMissingStrategy;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.util.Random;

import static spark.Spark.*;

public class BackendService {
    private static final String DB_HOST = System.getenv("DB_HOST");
    private static final String DB_NAME = System.getenv("DB_NAME");
    private static final String DB_USER = System.getenv("DB_USER");
    private static final String DB_PASS = System.getenv("DB_PASS");

    public static void main(String[] args) {
        AWSXRayRecorderBuilder.defaultRecorder();

        AWSXRay.getGlobalRecorder().setContextMissingStrategy(
            new ContextMissingStrategy() {
                @Override
                public void contextMissing(
                    String message,
                    Class<? extends RuntimeException> exceptionClass
                ) {
                    // Ignore a missing trace context for local lab requests.
                }
            }
        );

        port(8080);

        before((req, res) -> {
            if (AWSXRay.getCurrentSegmentOptional().isEmpty()) {
                AWSXRay.beginSegment("BackendService");
            }
        });

        after((req, res) -> {
            if (AWSXRay.getCurrentSegmentOptional().isPresent()) {
                AWSXRay.endSegment();
            }
        });

        get("/cpu", (req, res) -> {
            AWSXRay.beginSubsegment("ComputeTask");
            try {
                long start = System.currentTimeMillis();
                Random random = new Random();

                while (System.currentTimeMillis() - start < 2000) {
                    Math.sqrt(random.nextDouble());
                }

                return "CPU load completed";
            } finally {
                AWSXRay.endSubsegment();
            }
        });

        get("/db", (req, res) -> {
            AWSXRay.beginSubsegment("PostgresQuery");
            Connection connection = null;

            try {
                String jdbcUrl = "jdbc:postgresql://" + DB_HOST + ":5432/" + DB_NAME;
                connection = DriverManager.getConnection(jdbcUrl, DB_USER, DB_PASS);

                try (PreparedStatement statement =
                         connection.prepareStatement("SELECT now()");
                     ResultSet resultSet = statement.executeQuery()) {
                    if (resultSet.next()) {
                        return "DB time: " + resultSet.getString(1);
                    }
                    return "Database query returned no rows";
                }
            } catch (Exception e) {
                AWSXRay.getCurrentSubsegmentOptional().ifPresent(segment -> segment.addException(e));
                res.status(500);
                return "Database request failed; check the backend log for details";
            } finally {
                if (connection != null) {
                    try {
                        connection.close();
                    } catch (Exception ignored) {
                        // Best-effort cleanup for this lab.
                    }
                }
                AWSXRay.endSubsegment();
            }
        });
    }
}
EOF
~~~

The backend exposes two endpoints:

- /cpu performs a short CPU-intensive task to make CPU utilization easier to observe.
- /db connects to PostgreSQL and executes SELECT now().

The database connection values are read from environment variables rather than hard-coded into the source code. The X-Ray calls illustrate the original lab approach; the SDK and daemon lifecycle note above applies.

### 3.3 Create pom.xml

~~~bash
cat <<'EOF' > ~/backend/pom.xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>backendservice</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>com.amazonaws</groupId>
            <artifactId>aws-xray-recorder-sdk-core</artifactId>
            <version>2.21.1</version>
        </dependency>
        <dependency>
            <groupId>com.sparkjava</groupId>
            <artifactId>spark-core</artifactId>
            <version>2.9.4</version>
        </dependency>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <version>42.7.7</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.13.0</version>
                <configuration>
                    <source>17</source>
                    <target>17</target>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.5.1</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <transformers>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                    <mainClass>com.example.BackendService</mainClass>
                                </transformer>
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
EOF
~~~

The Maven configuration declares the libraries used by the Java application and packages an executable JAR. Verify dependency compatibility and updates before using lab code in any non-lab environment.

### 3.4 Build the backend

~~~bash
cd ~/backend
mvn clean package
ls -lh target/
~~~

What the commands do:

- clean removes the previous build output.
- package compiles the source and packages the application.
- The Shade plugin bundles the application dependencies and configures the main class.

A JAR similar to target/backendservice-1.0-SNAPSHOT.jar should be produced. Do not ignore build errors; inspect the first compiler or dependency error and correct it.

### 3.5 Configure the database environment variables

Replace the placeholders with values from your own CloudFormation outputs and database settings:

~~~bash
export DB_HOST="<your-rds-endpoint>"
export DB_NAME="<your-database-name>"
export DB_USER="<your-database-username>"
read -s -p "Database password: " DB_PASS
echo
export DB_PASS
~~~

The password prompt does not echo the password. These variables are available to processes started from the current shell. If Session Manager disconnects, re-enter the values before restarting the app.

Do not commit real credentials to the README, source code, or shell scripts. For production workloads, use a managed secret store and an appropriate runtime secret-delivery mechanism.

### 3.6 Start and verify the backend

~~~bash
cd ~/backend
nohup java -jar target/backendservice-1.0-SNAPSHOT.jar > backend.log 2>&1 &
~~~

Explanation:

- nohup allows the process to continue after the session disconnects.
- java -jar runs the packaged application.
- The redirection writes standard output and standard error to backend.log.
- The ampersand runs the process in the background.

Check the process and application logs:

~~~bash
pgrep -af 'backendservice-1.0-SNAPSHOT.jar'
tail -n 50 ~/backend/backend.log
~~~

Test both endpoints from the backend instance:

~~~bash
curl -i http://localhost:8080/cpu
curl -i http://localhost:8080/db
~~~

Expected behavior:

- /cpu returns the text CPU load completed.
- /db returns the database timestamp when the endpoint, credentials, network route, security groups, and database state are correct.

If /db fails, check DB_HOST, DB_NAME, DB_USER, DB_PASS, RDS status, the RDS security group, network routing, and the backend log.

---

## Step 4: Deploy the Node.js Frontend Application

Run these commands on the frontend EC2 instance.

### 4.1 Create the project and package.json

~~~bash
mkdir -p ~/frontend
cd ~/frontend

cat <<'EOF' > ~/frontend/package.json
{
  "name": "frontend-service",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "axios": "^1.3.6",
    "express": "^4.18.2"
  }
}
EOF
~~~

### 4.2 Install Node.js and the dependencies

~~~bash
sudo dnf install -y nodejs npm git stress
node --version
npm --version

cd ~/frontend
npm install
npm install aws-xray-sdk
~~~

The npm install command installs the dependencies defined in package.json. The second command installs the X-Ray SDK used by this lab's sample app.

### 4.3 Create index.js

~~~bash
cat <<'EOF' > ~/frontend/index.js
const AWSXRay = require('aws-xray-sdk');
const express = require('express');
const axios = require('axios');

AWSXRay.captureHTTPsGlobal(require('http'));
AWSXRay.captureHTTPsGlobal(require('https'));

const app = express();

app.use(AWSXRay.express.openSegment('FrontendService'));

const BACKEND_HOST = process.env.BACKEND_HOST || 'localhost';
const BACKEND_PORT = process.env.BACKEND_PORT || '8080';

app.get('/work/cpu', (req, res) => {
  const start = Date.now();

  while (Date.now() - start < 2000) {
    Math.sqrt(Math.random());
  }

  res.send('Frontend CPU load complete');
});

app.get('/work/db', async (req, res) => {
  try {
    const url = 'http://' + BACKEND_HOST + ':' + BACKEND_PORT + '/db';
    const response = await axios.get(url);
    res.send('Frontend DB result: ' + response.data);
  } catch (err) {
    console.error('Backend request failed:', err.message);
    res.status(500).send('Backend request failed; check the frontend log');
  }
});

app.use(AWSXRay.express.closeSegment());

app.listen(3000, () => {
  console.log('Frontend listening on port 3000');
});
EOF
~~~

The application runs on port 3000:

- /work/cpu performs a short CPU-intensive operation in the frontend process.
- /work/db calls the backend /db endpoint and returns its response.
- X-Ray middleware and HTTP instrumentation demonstrate tracing for requests, assuming the tracing components and permissions are configured correctly.

### 4.4 Configure the backend private address

Use the private IP of the backend instance in your lab:

~~~bash
export BACKEND_HOST="<backend-private-ip>"
export BACKEND_PORT="8080"
~~~

Test network reachability from the frontend:

~~~bash
curl -i "http://$BACKEND_HOST:$BACKEND_PORT/cpu"
curl -i "http://$BACKEND_HOST:$BACKEND_PORT/db"
~~~

If these requests fail, check the backend process, private IP, port, route configuration, and backend security group. Permit inbound port 8080 only from the intended frontend security group or other authorized source.

### 4.5 Start and test the frontend

~~~bash
cd ~/frontend
nohup node index.js > frontend.log 2>&1 &

pgrep -af 'node index.js'
tail -n 50 ~/frontend/frontend.log
~~~

Test the endpoints:

~~~bash
curl -i http://localhost:3000/work/cpu
curl -i http://localhost:3000/work/db
~~~

Expected behavior:

- /work/cpu returns Frontend CPU load complete.
- /work/db returns a result from the backend database request.

If the frontend cannot call the backend, inspect BACKEND_HOST, BACKEND_PORT, the backend log, and the security group rules.


## Step 5: Restart Applications After Reconnecting

Environment variables do not automatically return in a new Session Manager shell. Re-enter the values before starting the applications again.

### On the backend instance

~~~bash
cd ~/backend
export DB_HOST="<your-rds-endpoint>"
export DB_NAME="<your-database-name>"
export DB_USER="<your-database-username>"
read -s -p "Database password: " DB_PASS
echo
export DB_PASS
pgrep -af 'backendservice-1.0-SNAPSHOT.jar'
~~~

If the backend is not already running, start it:

~~~bash
nohup java -jar target/backendservice-1.0-SNAPSHOT.jar > backend.log 2>&1 &
~~~

Verify:

~~~bash
curl -i http://localhost:8080/cpu
curl -i http://localhost:8080/db
~~~

### On the frontend instance

~~~bash
cd ~/frontend
export BACKEND_HOST="<backend-private-ip>"
export BACKEND_PORT="8080"
pgrep -af 'node index.js'
~~~

If the frontend is not running, start it:

~~~bash
nohup node index.js > frontend.log 2>&1 &
~~~

Verify:

~~~bash
curl -i http://localhost:3000/work/cpu
curl -i http://localhost:3000/work/db
~~~

Check for an existing process before starting another copy; duplicate processes can fail when the port is already in use.

---

## Step 6: Create a CloudWatch Dashboard

### 6.1 Confirm that metrics are arriving

1. Open CloudWatch in the AWS console.
2. Select Metrics and then the CWAgent namespace.
3. Confirm that metrics exist for the current frontend and backend instances.
4. Open Log groups and check for the configured application log groups.
5. If data is missing, confirm that the agent is running, the IAM role allows publishing, and the configured file paths exist.

### 6.2 Create the dashboard

1. Open CloudWatch → Dashboards.
2. Select Create dashboard.
3. Enter a name such as StressTestDashboard.
4. Add metric widgets, then save the dashboard after arranging them.

### 6.3 Add CloudWatch Agent metrics

1. Add a metric widget and select the CWAgent namespace.
2. Select the current frontend and backend instances using the dimensions available in your configuration.
3. Add memory, swap, and disk utilization metrics.
4. Use a descriptive widget title such as Host Memory and Disk Utilization.

Metric names and dimensions depend on the agent configuration and operating system.

### 6.4 Add EC2 CPU utilization

1. Add a widget and choose EC2 → Per-Instance Metrics.
2. Select CPUUtilization for the current frontend and backend instances.
3. Use the same statistic and period when comparing their behavior.
4. Save the widget.

### 6.5 Add CPU credit metrics where applicable

CPU credit metrics are available only for supported burstable EC2 instance families. If your lab uses an eligible T-series instance:

1. Add a widget for the relevant EC2 instance.
2. Select available metrics such as CPUCreditBalance and CPUCreditUsage.
3. Save the widget and compare these metrics during the stress test.

A CPU credit represents one vCPU running at 100% utilization for one minute. Behavior at the baseline and when credits are exhausted depends on the instance family and operating mode. Do not assume every EC2 instance uses CPU credits.

### 6.6 Add RDS metrics

1. Add a metric widget and select the AWS/RDS namespace.
2. Select the database instance identifier for the lab.
3. Add applicable metrics such as CPUUtilization, ReadLatency, WriteLatency, and BurstBalance where available.
4. Add any CPU credit metrics only if supported by the RDS instance class.
5. Save the widget.

Available metrics depend on the RDS engine, storage, instance class, and monitoring configuration.

### 6.7 Choose appropriate metric periods

Standard EC2 and RDS metrics commonly have one-minute resolution depending on monitoring settings. Use a period supported by the selected metric. A one-second dashboard period does not create one-second data if the metric is published only once per minute.

Use Average to understand typical behavior, and Maximum to identify peaks within the chosen aggregation period. High-resolution custom metrics require appropriate collection settings and can add costs.

Enable EC2 detailed monitoring only if the exercise needs it and you understand the additional charge.

**Verification:** Run a short load test and confirm the correct graph changes for the current instance, not a previously terminated instance.

---

## Step 7: Create a CloudWatch Alarm and SNS Email Notification

The threshold below is an example for a lab, not a production alerting recommendation.

### 7.1 Create the alarm

1. Open CloudWatch → Alarms, or select the CPU metric from your dashboard.
2. Choose CPUUtilization for the current backend instance.
3. Configure a threshold, for example CPU utilization greater than 50%.
4. Select the metric period and evaluation periods supported by its resolution.
5. Review the metric graph and alarm conditions.

A 10-second evaluation period may not be appropriate for a standard one-minute metric. Use a period that matches the selected metric's resolution.

### 7.2 Configure SNS

1. Add an alarm notification action.
2. Create or select an SNS topic, for example SREtest.
3. Add an email address you can access.
4. Save the topic and alarm settings.
5. Open your inbox and confirm the SNS subscription.
6. Return to the alarm wizard and complete alarm creation.

### 7.3 Verify the alarm

1. Run the short CPU test in Step 8.
2. Observe the metric and alarm state.
3. Confirm that the alarm changes state only when the configured condition is met.
4. Check the destination inbox if a notification should have been sent.

An alarm in INSUFFICIENT_DATA may need more metric samples. Verify the resource dimension, metric resolution, threshold, evaluation period, and permissions before changing the alarm.

---

## Step 8: Stress Test the Backend Instance

Run these tests only on authorized lab instances. Start small, observe the result, and increase load gradually.

### 8.1 Install the stress utility

On the backend instance:

~~~bash
sudo dnf install -y stress
stress --version
~~~

If the package is unavailable for the provided AMI, use the installation method supported by that operating system.

### 8.2 Run a short CPU test

~~~bash
stress --cpu 2 --timeout 60
~~~

Explanation:

- --cpu 2 starts two CPU workers.
- --timeout 60 stops the test after 60 seconds.

### 8.3 Observe metrics and logs

While the command runs:

1. Watch backend CPUUtilization in the CloudWatch dashboard.
2. Check CPU credit metrics if the instance type supports them.
3. Observe any alarm state changes.
4. Compare the metric before, during, and after the test.
5. Review the application and CloudWatch Agent logs if expected data is absent.

### 8.4 Increase the workload carefully

After reviewing the first test, you may run a slightly larger lab workload:

~~~bash
stress --cpu 4 --timeout 120
~~~

Adjust the worker count to the instance size and lab conditions. The goal is to observe the response, not to exhaust resources.

---

## Step 9: Stress Test the Frontend and Generate HTTP Load

### 9.1 Run a frontend CPU test

On the frontend instance:

~~~bash
stress --cpu 2 --timeout 60
~~~

Observe the frontend CPU metric during the test.

### 9.2 Send a database request through the frontend

~~~bash
curl -i http://localhost:3000/work/db
~~~

The request flows through frontend → backend → RDS. Compare frontend and backend logs, traces, and database metrics for the same time window.

### 9.3 Install ApacheBench and run a small benchmark

~~~bash
sudo dnf install -y httpd-tools
ab -n 1000 -c 10 http://localhost:3000/work/db
~~~

Explanation:

- -n 1000 sends 1,000 total requests.
- -c 10 allows up to 10 concurrent requests.
- The URL targets the frontend endpoint that calls the backend database route.

ApacheBench reports request throughput, timing statistics, and failed requests. Results depend on the instance size, database performance, network, and application implementation.

### 9.4 Increase workload only when safe

The original lab contains longer and much higher-concurrency tests, such as dozens of CPU workers, long durations, and 100,000 HTTP requests. These can saturate instances, overwhelm a database, interrupt other learners, and create charges.

Do not run the largest tests by default. Increase request counts, concurrency, and duration gradually. Stop the test if the service becomes unhealthy or if costs exceed the lab budget.

For a controlled follow-up test, use a moderate workload such as:

~~~bash
stress --cpu 4 --timeout 120
ab -n 5000 -c 10 http://localhost:3000/work/db
~~~

Only run higher values when the lab is isolated, authorized, and designed to handle that level of load.

### 9.5 Analyze CPU credits and bottlenecks

On eligible burstable instances, sustained CPU demand may reduce available credits. The behavior after credits are depleted depends on instance type and operating mode. Correlate credit metrics with CPU utilization, response latency, error counts, and database metrics.

Questions to investigate:

- Does frontend CPU rise during the frontend CPU test?
- Does backend CPU rise when requests call /work/db?
- Does RDS CPU utilization or database latency change?
- Do response times increase as concurrency increases?
- Does the application return errors when a resource is saturated?
- Can logs and traces distinguish frontend processing from backend and database latency?

---

