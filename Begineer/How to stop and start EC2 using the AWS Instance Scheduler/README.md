- Create EC2

![alt text](image.png)

![alt text](image-1.png)

![alt text](image-2.png)

- I left VPC, Subnet, Security Group information in here. Please set up with yours.

![alt text](image-3.png)

- To use the scheduler CLI, you must have credentials for the AWS CLI.

Your credentials must have the following permissions:

`lambda:InvokeFunction` – To invoke the InstanceSchedulerMain function in the scheduler stack, and to update the schedule and period information in the scheduler configuration database from the command line
    
`cloudformation:DescribeStackResource` – To retrieve the physical resource ID of the AWS Lambda function from the stack to handle the CLI request

- The CLI in this solution requires `Python 3.8+`

- Go To `IAM` from AWS Console

![alt text](image-4.png)

- Click `Roles`

![alt text](image-7.png)

- Click `Create role`

![alt text](image-8.png)

- Select `EC2` and Click `Next`

![alt text](image-9.png)

- Choose `AWSLambdaInvocation-DynamoDB`

![alt text](image-10.png)

- Choose `AWSCloudFormationReadOnlyAccess`

![alt text](image-11.png)

- Write `EC2-Instance-Scheduler-Role`

![alt text](image-12.png)

- Click `Create role`

![alt text](image-13.png)

- Attach IAM role to EC2

![alt text](image-14.png)

- Select `EC2-Instance-Scheduler-Role` and Click `Update IAM role`

![alt text](image-15.png)

![alt text](image-16.png)

- Select EC2 and Click `Connect`

![alt text](image-17.png)

- Click `Connect`

![alt text](image-18.png)

- Type 
```
python3 --version
```
and click enter

![alt text](image-19.png)

- Type
```
sudo yum install python3.x86_64 python3-pip.noarch
```

![alt text](image-20.png)

- Type 
```
python3 --version
```

![alt text](image-21.png)

- Type
```
wget https://s3.amazonaws.com/solutions-reference/instance-scheduler-on-aws/latest/instance_scheduler_cli-1.5.3-py3-none-any.whl
```

![alt text](image-22.png)

- Type
```
pip install instance_scheduler_cli-1.5.3-py3-none-any.whl
```

![alt text](image-24.png)

- Type
```
scheduler-cli --version
```
![alt text](image-25.png)

- Type
```
scheduler-cli -h
```
![alt text](image-26.png)

- Download the CloudFormation for InstanceScheduler , Click [Here](https://static.us-east-1.prod.workshops.aws/public/8c252e50-e251-4940-a888-37b7b32a816d/assets/aws-instance-scheduler-sup304v3.template)

- or you can directly go by Click [Here](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?templateURL=https://s3.amazonaws.com/solutions-reference/instance-scheduler-on-aws/latest/instance-scheduler-on-aws.template)
.This will go to CloudFormation Page directly.

- Go to `CloudFormation`

![alt text](image-27.png)

- Click `With new resources (standard)`

![alt text](image-28.png)

- Click `Choose file`

![alt text](image-29.png)

- Click `Next`

![alt text](image-30.png)

Enter `InstanceScheduler` as a stack name

![alt text](image-32.png)

Scroll down and click on `next`

![alt text](image-31.png)

Scroll down and click on `next`

![alt text](image-33.png)

Scroll down and click on `submit`

![alt text](image-34.png)

- Wait the CloudFormation stack is deployed successfully.

![alt text](image-35.png)

![alt text](image-36.png)

- Type
```
scheduler-cli describe-schedules --region ap-southeast-1 --stack InstanceScheduler
```

- You will see the out like this but timezone is different with our local time

```
{
   "Schedules": [
      {
         "Description": "Instances running",
         "Name": "running",
         "UseMetrics": false,
         "Type": "schedule",
         "OverrideStatus": "running"
      },
      {
         "Timezone": "UTC",
         "Description": "Vertical scaling on weekdays, based on UTC time",
         "Periods": [
            "working-days@t2.micro",
            "weekends@t2.nano"
         ],
         "Name": "scale-up-down",
         "Type": "schedule"
      },
      {
         "Timezone": "US/Pacific",
         "Description": "Office hours in Seattle (Pacific)",
         "Periods": [
            "office-hours"
         ],
         "Name": "seattle-office-hours",
         "Type": "schedule"
      },
      {
         "Description": "Instances stopped",
         "Name": "stopped",
         "UseMetrics": false,
         "Type": "schedule",
         "OverrideStatus": "stopped"
      },
      {
         "Timezone": "Europe/London",
         "Description": "Office hours in UK",
         "Periods": [
            "office-hours"
         ],
         "Name": "uk-office-hours",
         "Type": "schedule"
      }
   ]
}
```

- Type
```
scheduler-cli describe-periods --region ap-southeast-1 --stack InstanceScheduler
```
- You will see the out like this but timezone is different with our local time


```
{
   "Periods": [
      {
         "Months": [
            "jan/3"
         ],
         "Description": "Every first monday of each quarter",
         "Weekdays": [
            "mon#1"
         ],
         "Name": "first-monday-in-quarter",
         "Type": "period"
      },
      {
         "Begintime": "09:00",
         "Description": "Office hours",
         "Endtime": "17:00",
         "Weekdays": [
            "mon-fri"
         ],
         "Name": "office-hours",
         "Type": "period"
      },
      {
         "Description": "Days in weekend",
         "Weekdays": [
            "sat-sun"
         ],
         "Name": "weekends",
         "Type": "period"
      },
      {
         "Description": "Working days",
         "Weekdays": [
            "mon-fri"
         ],
         "Name": "working-days",
         "Type": "period"
      }
   ]
}
```

- We have to change information at `DynamoDB` so go to `DynamoDB` from AWS Console

- Click `Explore Items`

![alt text](image-38.png)

- Select `InstanceScheduler-ConfigTable-EIC9ZOW4EYKI` and `Edit Item`

![alt text](image-39.png)

- Change `name` and `timezone` like this and click `Recreate Item`

![alt text](image-37.png)

- Type
```
scheduler-cli describe-schedules --region ap-southeast-1 --stack InstanceScheduler
```

- You will see the out like this and timezone is changed with our local time

```
{
   "Schedules": [
      {
         "Timezone": "Asia/Yangon",
         "Description": "Office hours in Seattle (Pacific)",
         "Periods": [
            "office-hours"
         ],
         "Name": "myanmar-office-hours",
         "Type": "schedule"
      },
      {
         "Description": "Instances running",
         "Name": "running",
         "UseMetrics": false,
         "Type": "schedule",
         "OverrideStatus": "running"
      },
      {
         "Timezone": "UTC",
         "Description": "Vertical scaling on weekdays, based on UTC time",
         "Periods": [
            "working-days@t2.micro",
            "weekends@t2.nano"
         ],
         "Name": "scale-up-down",
         "Type": "schedule"
      },
      {
         "Description": "Instances stopped",
         "Name": "stopped",
         "UseMetrics": false,
         "Type": "schedule",
         "OverrideStatus": "stopped"
      },
      {
         "Timezone": "Europe/London",
         "Description": "Office hours in UK",
         "Periods": [
            "office-hours"
         ],
         "Name": "uk-office-hours",
         "Type": "schedule"
      }
   ]
}
```

- Type
```
scheduler-cli describe-periods --region ap-southeast-1 --stack InstanceScheduler
```
- You will see the out like this and timezone is changed with our local time


```
{
   "Periods": [
      {
         "Months": [
            "jan/3"
         ],
         "Description": "Every first monday of each quarter",
         "Weekdays": [
            "mon#1"
         ],
         "Name": "first-monday-in-quarter",
         "Type": "period"
      },
      {
         "Begintime": "09:00",
         "Description": "Office hours",
         "Endtime": "17:00",
         "Weekdays": [
            "mon-fri"
         ],
         "Name": "office-hours",
         "Type": "period"
      },
      {
         "Description": "Days in weekend",
         "Weekdays": [
            "sat-sun"
         ],
         "Name": "weekends",
         "Type": "period"
      },
      {
         "Description": "Working days",
         "Weekdays": [
            "mon-fri"
         ],
         "Name": "working-days",
         "Type": "period"
      }
   ]
}
```

Go to `Tag Editor` from AWS Console

![alt text](image-40.png)

- Regions - `ap-southeast-1`

- Resource types - `All supported resources types`

- Tag key - `Environment`

- Optional tag value - `Dev`

![alt text](image-41.png)

- Click `Search resources` and Click `Manage tags of selected resources`

![alt text](image-42.png)

- Click `Add`

- Tag key - `Schedule`

- Optional tag value - `myanmar-office-hours`

- Click `Review and apply tag changes`

![alt text](image-43.png)

- Click `Apply changes to all selected`

![alt text](image-44.png)

---

## **Resources & Next Steps**

*   **📦 Full Code Repository:** [AWS Learning Labs](https://github.com/thukhakyawe/aws-learning-labs) - Get the complete, working code from this post
*   **📖 More Deep Dives:** [Whispering Cloud Insights](https://thukhakyawe.hashnode.dev/) - Read other technical articles
*   **💬 Join Discussion:** [DEV Community](https://dev.to/thukhakyawe_cloud) - Share your thoughts and questions
*   **💼 Let's Connect:** [Linkedin](www.linkedin.com/in/thukhakyawe/) - I'd love to connect with you

---