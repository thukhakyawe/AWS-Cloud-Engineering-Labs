![alt text](<Adobe Express - file.png>)


## 1. Create EC2 for Amazon Linux

![alt text](image.png)

## 2. Create Cloudwatch Alarm

- In the AWS Management Console, type `cloudwatch` into the search box, and then select the `CloudWatch` service.

![alt text](image-1.png)

- Click `All alarms`

![alt text](image-2.png)

- Click `Create alarm`

![alt text](image-3.png)

- Click `Select metric` 

![alt text](image-4.png)

- Click `EC2`

![alt text](image-5.png)

- Click `Per-Instance Metrics`

![alt text](image-6.png)

- Select `Cloudwatch Lab` and click `Create alarm`

![alt text](image-7.png)

- Set Period based on your need

![alt text](image-8.png)

- Select `Greater/Equal` and Write `80` and then Click `Next`

![alt text](image-9.png)

- Select `Create new topic` , Write topic name and your email then click `Next`

![alt text](image-11.png)

- Write `CPU High Usage Alert` and click `Next` then click `Create alarm`

![alt text](image-12.png)

## 3. Install stress application at EC2

- In the AWS Management Console, type `EC2` into the search box, and then select the `EC2` service.
•	On the Instances page, select the `Cloudwatch lab` check box, and then select `Connect`.

![alt text](image-13.png)

- To install the Stress Application

``` sudo yum install stress -y ```

``` sudo stress --cpu 4 ```

## 4. Monitor the CPU High Usage Alert alarm for the WebServer EC2 instance ##

- In the AWS Management Console, type `EC2` into the search box, and then select the `EC2` service.

![alt text](image-14.png)

- Select `Cloudwatch lab` and `Monitoring`

![alt text](image-15.png)

## 5. Check CPU High Usage Alert ##

- In the AWS Management Console, type `Cloudwatch` into the search box, and then select the `CloudWatch` service.

![alt text](image-16.png)

- Click `All alarms`

![alt text](image-17.png)

- You will see alarm is `in alarm` state

![alt text](image-18.png)

--------------------------
***That it is. Congratulations, you have completed Lab-How To Create a CloudWatch Alarm for CPU Utilization***

--------------------------