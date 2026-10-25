### Step 1 - Create `EC2` and attach `CloudWatchAgentAdminPolicy` role to EC2 ###

![alt text](image.png)

![alt text](image-1.png)

![alt text](image-2.png)

- Choose `CloudWatchAgentAdminPolicy` role which  already created by using AWS Managed Rule `CloudWatchAgentAdminPolicy`

![alt text](image-3.png)


### Step 2 - Install CloudWatch Agent on EC2 ###

- For installation CloudWatch agent we need to execute the following command:

```
sudo yum install amazon-cloudwatch-agent -y 
```

- After running this command you will see that installation is complete.

![alt text](image-4.png)

- Configure the CloudWatch agent with the wizard and fill in data about our log file.To create a configuration file execute the following command:

```
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

- After running this command we need to answer the following questions (I will provide answers for my configuration):

![alt text](image-5.png)

1. On which OS are you planning to use the agent? (Linux)

![alt text](image-6.png)

2. Are you using EC2 or On-Premises hosts? (EC2)

![alt text](image-7.png)

3. Which user are you planning to run the agent? (cwagent)

![alt text](image-8.png)

4. Do you want to turn on the StatsD daemon? (yes)

![alt text](image-9.png)

5. Which port do you want the StatsD daemon to listen to? (8125)

![alt text](image-10.png)


6. What is the collection interval for the StatsD daemon? (10s)

![alt text](image-11.png)


7. What is the aggregation interval for metrics collected by 
StatsD daemon?(the 60s)

![alt text](image-12.png)


8. Do you want to monitor metrics from CollectD?(No)

![alt text](image-13.png)


9. Do you want to monitor any host metrics? e.g. CPU, memory, etc. (yes)

![alt text](image-14.png)


10. Do you want to monitor CPU metrics per core? (yes)

![alt text](image-15.png)

11. Do you want to add ec2 dimensions (ImageId, InstanceId, InstanceType, AutoScalingGroupName) into all of your metrics if the info is available? (yes)

![alt text](image-16.png)


12. Do you want to aggregate ec2 dimensions (InstanceId)? (yes)

![alt text](image-17.png)


13. Would you like to collect your metrics at high resolution (sub-minute resolution)? This enables sub-minute resolution for all metrics, but you can customize for specific metrics in the output JSON file. (60s)

![alt text](image-18.png)


14. Which default metrics config do you want? (Standard) 
    On this question, you can choose the answer between Basic, Standard, Advanced, and None. (Detailed description below)

<table>

<tr>
<td width="33%"">
Detail level - Basic
</td>
<td width="33%">
Metrics included
</td>
</tr>

<tr>
<td width="33%"">
Mem	
</td>
<td width="33%">
mem_used_percent
</td>
</tr>

<tr>
<td width="33%"">
Disk	
</td>
<td width="33%">
disk_used_percent
</td>
</tr>

</table>

------

<table>

<tr>
<td width="33%"">
Detail level - Standard
</td>
<td width="33%">
Metrics included
</td>
</tr>

<tr>
<td width="33%"">
CPU	
</td>
<td width="33%">
cpu_usage_idle, cpu_usage_iowait, cpu_usage_user, cpu_usage_system
</td>
</tr>

<tr>
<td width="33%"">
Disk	
</td>
<td width="33%">
disk_used_percent, disk_inodes_free
</td>
</tr>

<tr>
<td width="33%"">
Diskio	
</td>
<td width="33%">
diskio_io_time
</td>
</tr>

<tr>
<td width="33%"">
Mem	
</td>
<td width="33%">
mem_used_percent
</td>
</tr>

<tr>
<td width="33%"">
Swap	
</td>
<td width="33%">
swap_used_percent
</td>
</tr>

</table>

-----

<table>

<tr>
<td width="33%"">
Detail level - Advanced
</td>
<td width="33%">
Metrics included
</td>
</tr>

<tr>
<td width="33%"">
CPU	
</td>
<td width="33%">
cpu_usage_idle, cpu_usage_iowait, cpu_usage_user, cpu_usage_system
</td>
</tr>

<tr>
<td width="33%"">
Disk	
</td>
<td width="33%">
disk_used_percent, disk_inodes_free
</td>
</tr>

<tr>
<td width="33%"">
Diskio	
</td>
<td width="33%">
diskio_io_time, diskio_write_bytes, diskio_read_bytes, diskio_writes, diskio_reads
</td>
</tr>

<tr>
<td width="33%"">
Mem	
</td>
<td width="33%">
mem_used_percent
</td>
</tr>

<tr>
<td width="33%"">
Swap	
</td>
<td width="33%">
swap_used_percent
</td>
</tr>

<tr>
<td width="33%"">
Netstat	
</td>
<td width="33%">
netstat_tcp_established, netstat_tcp_time_wait
</td>
</tr>

</table>

-----

15. Are you satisfied with the above config? (yes)

![alt text](image-19.png)
    
16. Do you have any existing CloudWatch Log Agent? (no)

![alt text](image-20.png)

17.Do you want to monitor any log files? (yes)

![alt text](image-21.png)
    
18. Log file path: (/var/log/httpd/error_log)

![alt text](image-22.png)

19. Log group name:

![alt text](image-23.png)

20. Log group class: (Standard)

![alt text](image-24.png)

21. Log stream name: [{instance_id}]

![alt text](image-25.png)

22. Log Group Retention in days (7)

![alt text](image-26.png)

23. Do you want to specify any additional log files to monitor? (no)

![alt text](image-27.png)

24. Do you want the CloudWatch agent to also retrieve X-ray traces?

![alt text](image-28.png)

25. Do you want to store the config in the SSM parameter store? (no)

![alt text](image-29.png)

- The configuration file will store in the “bin” folder:

```
ls /opt/aws/amazon-cloudwatch-agent/bin/
```

- Name - `config.json`

## Step 3 - Start the CloudWatch agent with our configuration file ##

- To launch the CloudWatch agent we need to execute the following command:

```
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json -s
```

- After running this command we see that execution is successfully finished.

![alt text](image-30.png)

- Check CloudWatch Agent Status

```
sudo systemctl status amazon-cloudwatch-agent
```

![alt text](image-31.png)




---

### **Resources & Next Steps**

*   **📦 Full Code Repository:** [AWS Learning Labs](https://github.com/thukhakyawe/aws-learning-labs) - Get the complete, working code from this post
*   **📖 More Deep Dives:** [Whispering Cloud Insights](https://thukhakyawe.hashnode.dev/) - Read other technical articles
*   **💬 Join Discussion:** [DEV Community](https://dev.to/thukhakyawe_cloud) - Share your thoughts and questions
*   **💼 Let's Connect:** [Linkedin](www.linkedin.com/in/thukhakyawe/) - I'd love to connect with you

---
