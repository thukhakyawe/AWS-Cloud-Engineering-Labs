Description

Introduction to Stress Testing

In this lab you will use AWS infrastructure — Web, Application, and Database, to help you explore stress testing tools. You will:

    Configure the infrastructure
    Run a series of tests to assess their affects.
    Configure Observability tools for
        monitoring
        logging
        tracing

Step 2: Noting the Variables

1. In the AWS console, search for "CloudFormation" and open.

2. From the list of stacks, click cloudacademylabs:

3. Find the Outputs tab that looks similar to this

4. Copy these values from the CloudFormation Outputs tab into a notepad 

5. Open the AWS console and search for and navigate to the EC2 console.

6. Navigate to Instances.

7. Select the Backend instance and note its Public and Private IP addresses.

8. Select the SRELab-Frontend instance and note its Public and Private IP addresses. Then repeat for the SRELab-Backend instance.

9. Note the Instance ID, Name, Public, and Private IP

Frontend Instance
i-0d95dbb4b1e2a1a94
54.191.50.242
10.98.1.250

Backend Instance
i-00a0da376c9dc1ff9
34.212.201.150
10.98.1.232

Step 3: Setting up the Instances [Do for Both Servers]

1. From the EC2 dashboard, select the appropriate Frontend/Backend EC2.

2. Click the Connect button.

3. Ensure the Session manager tab is selected and click Connect.

4. Type the following (or copy/paste) into the terminal window and run one code block at a time:

5. Install & configure the Cloudwatch Agent:
cd ~
pwd
sudo dnf install -y amazon-cloudwatch-agent

6. Create the configuration for the CloudWatch agent
vi ~/cwagent-config.json

7. Press i for Insert mode (and paste the following):

```
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
    "append_dimensions": { "InstanceId": "${aws:InstanceId}" },
    "metrics_collected": {
      "mem": { "measurement": ["mem_used_percent"], "metrics_collection_interval": 60 },
      "swap": { "measurement": ["swap_used_percent"], "metrics_collection_interval": 60 },
      "disk": { "measurement": ["used_percent"], "metrics_collection_interval": 60 }
    }
  }
}
```

8. Press the Esc key to exit insert mode and type :wq to save & quit.

9. Start the agent:

```
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
-a start -m ec2 -c file:/home/ssm-user/cwagent-config.json -s
```

10. Now verify:

```
ps aux | grep cloudwatch
```

11. Install the X-ray daemon and check status:

```
if [[ "$(uname -m)" == "aarch64" ]]; then
    arch="-arm64"
fi
sudo dnf install -y  https://s3.us-west-2.amazonaws.com/aws-xray-assets.us-west-2/xray-daemon/aws-xray-daemon${arch}-3.x.rpm
sudo systemctl enable --now xray
sudo systemctl status xray
```

Step 4: Deploy the Java app to the Backend Instance

1. Return to the SSM terminal for the Backend Instance.

2. Enter the following code and run it one code block at a time:

```
sudo dnf install -y java-17-amazon-corretto maven git
mkdir -p backend/src/main/java/com/example
vi ~/backend/src/main/java/com/example/BackendService.java
```

```
package com.example;
import com.amazonaws.xray.AWSXRay;
import com.amazonaws.xray.AWSXRayRecorderBuilder;
import com.amazonaws.xray.entities.Subsegment;
import com.amazonaws.xray.strategy.ContextMissingStrategy;
import spark.Spark;
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
        // Initialize X-Ray and set a no-op ContextMissingStrategy for local use
        AWSXRayRecorderBuilder.defaultRecorder();
        AWSXRay.getGlobalRecorder().setContextMissingStrategy(
            new ContextMissingStrategy() {
                @Override
                public void contextMissing(String message, Class<? extends RuntimeException> exceptionClass) {
                    // No operation: ignore context missing for local/testing
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
            Subsegment subsegment = AWSXRay.beginSubsegment("ComputeTask");
            try {
                long now = System.currentTimeMillis();
                while (System.currentTimeMillis() - now < 2000) {
                    Math.sqrt(new Random().nextDouble());
                }
                return "CPU load completed";
            } finally {
                AWSXRay.endSubsegment();
            }
        });
        get("/db", (req, res) -> {
            Subsegment subsegment = AWSXRay.beginSubsegment("PostgresQuery");
            Connection conn = null;
            String result = "";
            try {
                String url = "jdbc:postgresql://" + DB_HOST + ":5432/" + DB_NAME;
                conn = DriverManager.getConnection(url, DB_USER, DB_PASS);
                PreparedStatement stmt = conn.prepareStatement("SELECT now()");
                ResultSet rs = stmt.executeQuery();
                if (rs.next()) {
                    result = rs.getString(1);
                }
                return "DB time: " + result;
            } catch (Exception e) {
                subsegment.addException(e);
                res.status(500);
                return "DB error: " + e.getMessage();
            } finally {
                if (conn != null) try { conn.close(); } catch (Exception ignored) {}
                AWSXRay.endSubsegment();
            }
        });
    }
}
```

```
vi ~/backend/pom.xml
```

```
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
            <version>2.15.0</version>
        </dependency>
        <dependency>
            <groupId>com.sparkjava</groupId>
            <artifactId>spark-core</artifactId>
            <version>2.9.4</version>
        </dependency>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <version>42.6.0</version>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <!-- Compiler plugin -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.1</version>
                <configuration>
                    <source>17</source>
                    <target>17</target>
                </configuration>
            </plugin>
            <!-- Shade plugin for fat jar -->
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
```

7.  Now build the app.

```
cd ~/backend
mvn clean package
```

Explanation: 

    Runs Apache Maven, the Java build and dependency management tool.
    clean: Deletes the target/ directory (where compiled classes, JARs, WARs, etc. are stored). This ensures a fresh build with no leftovers.
    package: Compiles the source code, runs tests, and then packages the compiled code into a distributable format (like a JAR or WAR), as defined in your pom.xml

8.  Ignore any warnings but fix any errors.

9.  Run the following commands in the terminal to define the variables from the outputs recorded earlier. Ensure the RDS endpoint is replaced, including the chevrons, but the quotes remain.

export DB_HOST="<RDSEndpoint>"
export DB_NAME="labdb"
export DB_USER="labuser"
export DB_PASS="LabPass1234"

10. Now test the running of the lab by running the following command

```
nohup java -jar target/backendservice-1.0-SNAPSHOT.jar > backend.log 2>&1 &
```

Explanation:
•    nohup: Runs the command even after the session ends.
•    java -jar: Executes the JAR file.
•    > backend.log 2>&1: Redirects both stdout and stderr to backend.log.
•    &: Runs the process in the background.

11. View and test the application running.

```
ps aux | grep backendservice
curl http://localhost:8080/db
```

Expected responses:
•    /cpu: Should return a message indicating CPU load completion.
•    /db: Should return the current timestamp from the database.

You have now successfully setup a java application on the backend for testing

Step 5: Deploy the Node.js app to the frontend instance

1. Per the previous step instructions, ensure you have a SSM session manager terminal open to the frontend instance.

2. Enter the following commands.

```
cd ~
pwd
mkdir ~/frontend && cd ~/frontend
```

3. Create the JSON.

```
vi package.json
```
4. Press i to enter Insert mode.

5. Copy/ paste the following code:

```
{
  "name": "frontend-service",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "axios": "^1.3.6",
    "express": "^4.18.2"
  }
}
```


6. Press the Esc key then :wq to save and quit.

7. Install node, npm, and the stress tool we'll initially use.

```
sudo dnf install -y nodejs npm git stress
npm install
```

Explanation: The stress tool installed is a workload generator for testing system performance. It allows you to put a load on CPU, memory, I/O, or disk to simulate heavy usage

8. Now create the index file.

```
vi index.js
```

9. Press i to enter insert mode and paste the following:

```
const AWSXRay = require('aws-xray-sdk');
const express = require('express');
const axios = require('axios');
// Capture HTTP/S (this will allow AWS X-Ray to trace HTTP requests made by axios)
AWSXRay.captureHTTPsGlobal(require('http'));
AWSXRay.captureHTTPsGlobal(require('https'));
const app = express();
// Instrument Express with X-Ray
app.use(AWSXRay.express.openSegment('FrontendService'));
const BACKEND_HOST = process.env.BACKEND_HOST || "localhost";
const BACKEND_PORT = process.env.BACKEND_PORT || "8080";
app.get('/work/cpu', (req, res) => {
    let start = Date.now();
    while (Date.now() - start < 2000) { Math.sqrt(Math.random()); }
    res.send('Frontend CPU load complete');
});
app.get('/work/db', async (req, res) => {
    try {
        const response = await axios.get(`http://${BACKEND_HOST}:${BACKEND_PORT}/db`);
        res.send(`Frontend DB result: ${response.data}`);
    } catch (err) {
        res.status(500).send('Backend DB error: ' + err.message);
    }
});
// End the X-Ray segment for each response
app.use(AWSXRay.express.closeSegment());
app.listen(3000, () => console.log('Frontend listening on port 3000'));
```

10. Press the Esc key then :wq to save and quit.

Explanation: The code defines a Node.js Express app that uses AWS X-Ray to trace requests and interactions with a backend service. This frontend service simulates work or queries a backend, while AWS X-Ray traces both the incoming requests and outgoing backend calls for observability.

11. Run the following to install the X-ray SDK (for tracing):

```
npm install aws-xray-sdk
```

12. Set up the environment by pasting the following in the command line, replacing the Private IP of your backend instance (refer to the notepad you created earlier):

export BACKEND_HOST="<backend-private-ip>"
export BACKEND_PORT=8080


13. Run the application from the terminal command line

```
nohup node index.js > frontend.log 2>&1 &
```

14. Test the application from the terminal command line

```
curl http://localhost:3000/work/cpu
```

Step 6: Connecting to a configured Instance via SSM 

1. With the back-end terminal open, ensure you are in the correct directory.

```
cd ~
cd backend
```

2. Reapply any variables that will be lost when the terminal closes.


export DB_HOST="cloudacademylabs-rdsinstance-eorstszx5cc3.c29xocnzzj9t.us-west-2.rds.amazonaws.com"
export DB_NAME="labdb"
export DB_USER="labuser"
export DB_PASS="LabPass1234"

3. Re-run the application.

```
nohup java -jar target/backendservice-1.0-SNAPSHOT.jar > backend.log 2>&1 &
```

4. Test that the application is running.



ps aux | grep backendservice
curl http://localhost:8080/cpu
curl http://localhost:8080/db


At Frontend EC2

5.  With the front-end terminal open, ensure you are in the correct directory.


```
cd ~
pwd
cd ~/frontend
```

6. Configure the variables for the backend service.


export BACKEND_HOST="<Replace with your backend PRIVATE IP"
export BACKEND_PORT=8080


7. Start the application.

```
nohup node index.js > frontend.log 2>&1 &
```

8. Test the application.

```
curl http://localhost:3000/work/cpu
curl http://localhost:3000/work/db
```

Step 7: Setup the Cloudwatch Dashboard

1. Navigate to the CloudWatch dashboard.


2. To ensure the custom metrics have been created by the CloudWatch agent, navigate to: CloudWatch> Metrics> CWAgent in the console 

3. Create a dashboard to provide a single view of all our metrics. Navigate to the Dashboards section and click Dashboards.

4. Click Create dashboard.

5. Give the dashboard a name and click Create dashboard.

6. In the Add Widgets window, keep the defaults and click Next.

7. In the Add Metric graph, navigate to Custom namespaces and click CWAgent.

8. Select the tile with 2 entries.

9. Select both instances and click Create Widget. (Order the columns by name, then scroll to find the appropriate metrics)

10. Add CPU stats to view side by side.

Click the 3 dots in the top right-hand corner of the widget and click Edit.

11. Select the Browse tab.

12. Click EC2 under AWS namespaces.

13. Select per-instance Metrics.

14. Apply a CPU filter (to narrow the choice of metrics).

15. Select CPUUtilization for both instances and click Update Widget

Note: Ensure you select the correct instances; <?>-Frontend & <?>-Backend (older terminated instances may be cached)

16. Click the + button on the far right-hand side to add another widget.

17. Accept the defaults and click Next.

18. Select the CW Agent under custom namespaces. Click the tile with 14 entries.

19. Select ALL metrics and click Create widget.

20. Resize the widgets, and your StressTest Dashboard should be similar to this:

21. Click on the CPUUtilization Widget in the dashboard and select the 3 dots in the top right-hand corner, and select Edit.

22. Notice the statistic is set to Average over a Period of 5 minutes. We may change this to other values, for example, Maximum over 10 seconds. This controls the sampling and granularity. Make any changes you wish and click Update Widget.

23. We will add two more widgets. Click the + button to add a widget.

24. Accept the defaults and click Next.

25. Select EC2, then per instance metrics.

26. Apply a filter for credit.

27. For the Frontend instance, select the 3 CPU credit options.

28. Click Create Widget.

29.  Click the + button again to Add Widget.

30. Keep the defaults and click next.

31. Under the Metrics, select RDS.

32. Select DBinstanceIdentifier.

33. Select the following Metrics:
CPUsurplusCreditBalance
CPUCreditBalance
CPUUtilization
CPUCreditUsage
BurstBalance
ReadLatency
WriteLatency

34. Click Create widget.

35. Re-adjust the dashboard to see all widgets as 4 equal tiles.

36. For the lab, on BOTH new widgets, select the 3 dots and choose edit (complete one widget, then the other).

37.  Change the period on both widgets to 1 second (leave Statistic as average).

38. Click Update widget.

39. Remember....Save your Dashboard too!

40. Open the EC2 Dashboard in another tab.

41. Select one of the EC2s then Actions> Monitor and troubleshoot > Manage detailed monitoring.

42. Ensure Detailed monitoring is 'Enable' and click Confirm.

Step 8: Stress Testing the Backend Instance

```
cd ~/backend
sudo yum install -y stress
stress --cpu 2 --timeout 60
```

Explanation:
•    --cpu 2: Spawns 2 CPU workers
•    --timeout 60: Runs the test for 60 seconds

4. Return to your CloudWatch Dashboard to observe the effect.

5. Hover the mouse over the appropriate point in the graph, and the pop-up tile will give more detail, including the exact metric. Notice there is also a Bell icon to create an alert.

6.  We can now configure the metrics for an Alert. We will duplicate the environment we set for the widget just for the lab (set as appropriate in the real world).
For example: Lab: Maximum CPU over 10 Seconds with a threshold of Greater than 50.

7. Click the Bell icon and set it as below:

Maxium CPU Utilization > 50 for 10 seconds

8. Click Next.

9. Configure a Notification (see other actions that may also be available). Select the radio button for Create a new topic. (This creates a new SNS topic to send a notification to a specific subscriber)

Configure as below:

Give the topic a name: SREtest
Set an email that you can check & receive emails while performing the lab

10. Click Create Topic.

11. Click Next.

12. Name the Alert, for example, HighBackendCPU. 

13. Click Next.

14. Review and click Create Alarm.

15. Return to your Dashboard.

16. Before we continue, check your email. You’ll need to confirm subscription to the SNS Topic we have just created. Click to confirm.

17. From the SSM terminal window, run the stress test again (adjust the settings if desired):

stress --cpu 20 --timeout 120

18. Monitor from your Dashboard.

Step 9:Stress Testing the Frontend instance

cd ~/frontend
curl http://localhost:3000/work/db
stress --cpu 200 --timeout 1200

5. Return to your dashboard and monitor credit usage.

Explanation: As the workload increases, credits should decrease. If we run out of credits, the CPU will settle at its 20-30% baseline (the amount depends on the T series and size).
Note: If you see no changes in values, check that SSM terminals are still active. Ultimately, how long it takes for the credits to exhaust will depend on the number of vCPUs and your current balance — 1 credit = 100% vCPU / 1 min. (Follow the previous step instructions if you need to restart the Backend, then Frontend.)

6. On the Backend instance, run the following command:

stress --cpu 20 --timeout 900

7. On the Frontend instance, open a second SSM Terminal window and type the following:


cd ~
pwd
cd frontend
stress --cpu 20 --timeout 900

8. On the originalFrontend SSM terminal run:
sudo dnf install -y httpd-tools

9. Now run the following to simulate workload and benchmarking:
ab -n 100000 -c 10 http://localhost:3000/work/db

10. Return to the CloudWatch dashboard.

Consider: As you view metrics for the Frontend, Backend, and Database server, how would these be useful to identify where the workload originates and where any bottlenecks would occur?

These tests should run for approximately 15-20 minutes. You may re-run any of these tests and adjust their parameters. Open a number of terminals to increase the number of concurrent requests. Monitor the dashboards and also the benchmarking feedback from the ApacheBench tool. This will help to understand how multiple concurrent requests can be handled for any given percentile.
Consider where there may be issues as we observe the flow of activity end-to-end.

Step 10: End to End Tracing

During this step, we will use a tracing tool called X-ray. This will allow observability end-to-end across a distributed application/microservice setup.

1. From the Cloudwatch console, navigate to Application Signals (APM) and Trace Map.

2. If we select one of the nodes, we get detailed information about it (use this together with the metrics we discovered in our dashboard to help understand and ‘observe’.

Step 12:  Configure Jmeter (via Taurus) and run test

During this option lab step, you will enhance our current setup and perform tests with another tool — Junit. This may be run locally or called on with tools such as BlazeMeter.

Ensure you have SSM terminal sessions open to both the front-end and back-end instances.

On the Backend instance:

1. Confirm the correct tools are installed, run:

```
sudo dnf install -y java-17-amazon-corretto maven git python3-pip
pip3 install --upgrade pip
pip3 install bzt
```

Explanation:

What Taurus (bzt) is:

    An open-source test automation framework maintained by BlazeMeter.
    It allows you to define performance, functional, and load tests in a simple YAML/JSON configuration instead of writing raw JMeter/Gatling code.
    Wraps and integrates with tools like JMeter, Locust, Gatling, Selenium, etc
    Makes it easy to run tests locally or push them to BlazeMeter’s cloud.

Taurua hides the complexity of performance and functional tests with an automation-friendly convenience wrapper. Taurus relies on JMeter, Gatling, Locust.io, and Selenium WebDriver as its underlying tools. This can be installed by: pip install bzt

2. Export to the path environmental variable (this will only remain in the active session — re-run if necessary).

```
export PATH=$HOME/.local/bin:$PATH
```

Alternatively:

export PATH=$HOME/.local/bin:$PATH
echo 'export PATH=$HOME/.local/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

3. Create a directory for Taurus.

```
mkdir -p ~/taurus
cd ~/taurus
```

4. Define a configuration file for our load.

```
vi cpu_load.yml
```

5. Press i to enter insert mode and paste the code below:

```
# cpu_load.yml
execution:
  - executor: "jmeter"         # Use Taurus’s JMeter executor
    concurrency: 10            # Simulate 10 virtual users
    ramp-up: 10s               # Ramp all 10 users over 10 seconds
    hold-for: 1m               # Maintain 10 users for 1 minute
    scenario: "cpu_scenario"   # Reference the scenario below
scenarios:
  cpu_scenario:
    default-address: "http://localhost:8080"
    requests:
      - "/cpu"
```

6. Press the Esc key and type :wq to save and exit (from this point foward we will not repeat this detail of instruction).

Explanation:

    execution
        executor: "jmeter" Taurus will run JMeter under the hood to simulate the load.
        concurrency: 10    Simulates 10 virtual users in parallel.
        ramp-up: 10s    Gradually adds users over 10 seconds (so not all 10 start at once).
        hold-for: 1m    After ramp-up, keeps all 10 users active for 1 minute.
        scenario: "cpu_scenario"    Ties this execution to the defined cpu_scenario below.
    scenarios
        default-address: "http://localhost:8080"   Base URL for requests.
        requests:
            "/cpu"  Each virtual user repeatedly calls the /cpu endpoint.

7. Create a file called db_load.yml using vi and the steps above.

```
# db_load.yml
execution:
  - executor: "jmeter"         # Use the JMeter executor
    concurrency: 5             # Simulate 5 virtual users (DB is heavier)
    ramp-up: 5s                # Ramp all 5 users over 5 seconds
    hold-for: 2m               # Maintain 5 users for 2 minutes
    scenario: "db_scenario"    # Reference the scenario below
scenarios:
  db_scenario:
    default-address: "http://localhost:8080"
    requests:
      - "/db"
```

Explanation:

    execution
        executor: "jmeter"  Taurus will run the test using JMeter.
        concurrency: 5  Simulates 5 virtual users at once.   ( Lower than CPU test because hitting the DB is resource-intensive.)
        ramp-up: 5s  Gradually adds those 5 users over 5 seconds.
        hold-for: 2m  After ramp-up, keeps all 5 users active for 2 minutes.
        scenario: "db_scenario"  Links to the scenario definition below.
    scenarios
        default-address: "http://localhost:8080"  Base URL for requests.
        requests:
            "/db"  Each user repeatedly calls the /db endpoint.

What this test does: Starts with 0 users and scales up to 5 users within 5 seconds.  For 2 minutes, those 5 users continuously call http://localhost:8080/db. This generates a steady database load, testing latency, throughput, and error handling for DB queries.

8. Run the CPU test.

```
bzt cpu_load.yml
```

9. View the response (remember to check your CloudWatch dashboard).

View the performance graph, but also the response time for a given percentile.

10. Now run db load.

```
bzt db_load.yml
```

11. View the response (remember to check your CloudWatch dashboard).

Explanation:

Taurus launches JMeter to run 5 concurrent users calling /db for a total of 2 minutes (5s ramp-up + 115s hold).

You see similar metrics: throughput, latency, and error count.

Under light to moderate load, /db latency is typically 100–200 ms (network + Postgres).

After 2 minutes, the summary displays total requests (≈5 users × 120s = 600 calls) and average latency.

12. Consider how these metrics could be used to understand your system's performance capabilities and identify any issues.

Congratulations! You have completed a series of tests using JMeter (Taurus from Blazemeter).