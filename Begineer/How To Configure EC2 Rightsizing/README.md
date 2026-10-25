

- Go to `AWS Compute Optimizer` from AWS Console

![alt text](image.png)

- Select `EC2 instances`

![alt text](image-1.png)

- You will see a list of EC2 instances like this, 

![alt text](image-2.png)

- Click on the `filter by one or more properties searchbox`, and select `"Findings"` to select `"Over-provisioned"`

![alt text](image-3.png)

- You will see a list of instances with CPU over-provisioned like this

![alt text](image-4.png)

- Select EC2 and click `View Details`

![alt text](image-5.png)

- You will see three Options to choose and Select the option which has most saving on hourly price.

![alt text](image-6.png)

- Go to `EC2` from AWS Management Console select the instance and ensure it is the one you were targeting in Compute Optimizer

![alt text](image-7.png)

- You will see it here too. It is `Over-provisioned`

![alt text](image-8.png)

- Click on `Instance state` and select `Stop instance`

![alt text](image-9.png)

- Click `Stop`

![alt text](image-10.png)

- Change the instance type clicking on `Actions`, `Instance settings`and `Change instance type`


![alt text](image-12.png)

- Select new instance which is recommended by `Compute Optimizer` and click `Apply`

![alt text](image-13.png)

- Click on `Instance state` and select `Start instance`

![alt text](image-11.png)

---

> [!CAUTION]
> AWS recommends to take backups of EC2 before changing instance types and make sure to check with vendor to ensure compatibility and supportability before migrating your instances.

> [!NOTE]
> You can always manually change the instance back to its original type. 


> [!IMPORTANT]
> Check cost by using Cost Explorer or the Cost and Usage Report (CUR). Cost Explorer and CUR billing data are delayed by ~48hours, after ~2days you will be able to visualize the savings. 

---

## **Resources & Next Steps**

*   **📦 Full Code Repository:** [AWS Learning Labs](https://github.com/thukhakyawe/aws-learning-labs) - Get the complete, working code from this post
*   **📖 More Deep Dives:** [Whispering Cloud Insights](https://thukhakyawe.hashnode.dev/) - Read other technical articles
*   **💬 Join Discussion:** [DEV Community](https://dev.to/thukhakyawe_cloud) - Share your thoughts and questions
*   **💼 Let's Connect:** [Linkedin](www.linkedin.com/in/thukhakyawe/) - I'd love to connect with you

---