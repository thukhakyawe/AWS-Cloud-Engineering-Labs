## 1. Create VPC, Subnets, Route tables ##

- Create VPC

![alt text](image.png)

- Create Two EC2

![alt text](image-1.png)

![alt text](image-2.png)

![alt text](image-3.png)

- Use your VPC, Public Subnet, Security Group and add User-Data by using User-Data's information

![alt text](image-4.png)

- Click `Launch Instances`

- Create for Second Lab Server too

![alt text](image-5.png)

## 2. Create an application load balancer for EC2 ##

- Click `Load Balancers`

![alt text](image-6.png)

- Click `Create load balancer`

![alt text](image-7.png)

- Click `Create` of Application Load Balancer 

![alt text](image-8.png)

- Write ALB Name

![alt text](image-9.png)

- Select your vpc and choose public subnets and security group

- Click `Create target group`

![alt text](image-10.png)

- Write Name and click `Next`

![alt text](image-11.png)

- Click both EC2 and  `Include as pending below`

![alt text](image-12.png)

- Click `Create target group`

![alt text](image-13.png)

- Back to previous tab and choose your target group

![alt text](image-14.png)

- Click `Create load balancer`

![alt text](image-15.png)


## 3. Implement the Web Application Firewall (WAF) ##

- Go to `WAF` from AWS Management Console

![alt text](image-16.png)

- Click `Create web ACL`

![alt text](image-17.png)

- Choose `Singapore` at Region and write name

![alt text](image-18.png)

- Click `Add AWS resources`

![alt text](image-19.png)

- Select `Application Load Balancer` and `LabLB` then Click `Add`

![alt text](image-20.png)

- Click `Next`

![alt text](image-21.png)

- Click `Add rules` and `Add managed rule groups`

![alt text](image-22.png)

- Click `AWS managed rule groups`

- Select `Core rule set` and `SQL database` at `Free rule groups`

- Click `Add rules`

![alt text](image-23.png)

- Click `Next`

![alt text](image-21-1.png)

- Click `Next`

![alt text](image-24.png)

- Click `Next`

![alt text](image-25.png)

- Click `Create web ACL`

![alt text](image-26.png)

![alt text](image-27.png)

---

**Status:** Complete ✅

Congratulations on successfully completing this.

---

### **Related Resources**

*   **Lab Repository:** [AWS Learning Labs](https://github.com/thukhakyawe/aws-learning-labs)
*   **Technical Blog:** [Whispering Cloud Insights](https://thukhakyawe.hashnode.dev/)
*   **DEV Community:** [Thu Kha Kyawe](https://dev.to/thukhakyawe_cloud)

---