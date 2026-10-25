![alt text](<Adobe Express - file(1).png>)

## 1. Create two EC2 by using User-Data-Amazon-Linux Script

- Launch Two EC2 with Two Availability Zone

![alt text](image.png)

- I will left information about vpc,subnet[public],security. Please configure with yours.

- Attach IAM Instance role by using `CloudWatchAgentServerPolicy`

![alt text](image-1.png)


- Add User Data and Click `Launch Instance`
- Wait until `2/2 checks passed`

![alt text](image-2.png)

## 2. Create LoadBalancer ##

- Click `Load Balancer`

![alt text](image-3.png)


- Click `Create load balancers`

![alt text](image-5.png)

- Click `Create` of Application Load Balance

![alt text](image-4.png)

- Write `Load balancer name`

![alt text](image-12.png)

- Choose your vpc, select public subnet which route to IGW,  security group

- Click `Create target group`

![alt text](image-7.png)

- Write `Target group name` and Click `Next`

![alt text](image-8.png)

- Choose both EC2 and Click `Include as pending below`

![alt text](image-9.png)

- Click `Create Target Group`

![alt text](image-10.png)

- Choose `LabTG`

![alt text](image-11.png)

- Click `Create load balancer`

![alt text](image-6-1.png)

## 2. Open Running Public IPv4 DNS in browser ##

- Reload these links in browser a few times

![alt text](image-13.png)

![alt text](image-6.png)

- Add /admin.php at the end of public ipv4 dns and reload in browser a few times

![alt text](image-14.png)

## 3. Create a metric filter for 404 errors ##

- In the AWS Management Console, type `CloudWatch` and click this.

![alt text](image-15.png)

- In the navigation pane, in Logs, select `Log groups`

![alt text](image-16.png)

- Click `access_log`

![alt text](image-17.png)

- In the Log streams tab, select the `i-02a1487b82ba62ad7`

![alt text](image-18.png)

•	In Filter events, enter `404`, and then press Enter.

![alt text](image-19.png)

- Click `Create metric filter`

![alt text](image-20.png)

- Filter name - `LabServer1-Error404`

- Metric namespace - `CloudwatchLab`

- Metric name - `LabServer1-Error404`

- Metric value - `1`

- Default value - `0`
 
- Unit - `Count`

- Click `Create`

![alt text](image-21.png)

- Configure like this for `i-0ab69fd72c3a5498c` too but changes as `2` for Filter name and Metric name.

## 4. Create a custom CloudWatch dashboard ##

- Click `Dashboards`

![alt text](image-22.png)

- Click `Create dashboard`

![alt text](image-23.png)

- Write name and Click `Create dashboard`

![alt text](image-25.png)

- Click `Next`

![alt text](image-24-1.png)

- Select Untitled graph, and then enter `404-Error-log`

![alt text](image-24.png)

- Click `CloudwatchLab`

![alt text](image-26.png)

- Click `Metrics with no dimensions`

- Select both EC2 and Click `Create widget`

![alt text](image-27.png)

- Click `+`

![alt text](image-28.png)

- Click `Next`

![alt text](image-29.png)

- Select Untitled graph, and then enter `Alb-Traffic-Log`

![alt text](image-30.png)

- Click `ApplicationELB`

![alt text](image-31.png)

- Click `Per AppELB Metrics`

![alt text](image-32.png)

- Choose the metric you want to monitor and Click `Create widget`

> [!NOTE]
> Choose your alb carefully if you have a lof of alb

![alt text](image-33.png)

- Click `+`

- Click `Next`

- Select Untitled graph, and then enter `Memory Usage`

- Type `mem_used_percent` and enter

- Click `ImageId, InstanceId, InstanceType`

- Select both EC2 and Click `Create widget`

![alt text](image-34.png)

- Click `+`

- Click `Next`

- Select Untitled graph, and then enter `CPU Usage`

- Type `CPUUtilization` and enter

- Click `Per-Instance Metrics`

- Select both EC2 and Click `Create widget`

![alt text](image-35.png)

- Click `Save`

![alt text](image-36.png)

-----
***That it is.Congratulations, you have completed Lab-How To Create a CloudWatch Dashboard for EC2 and ELB.***

-----