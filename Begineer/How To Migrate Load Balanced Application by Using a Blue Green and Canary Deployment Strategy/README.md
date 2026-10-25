## Step 1. Create resources by using cloudformation template ##

![alt text](image-2.png)

- Click `Next`

![alt text](image-1.png)

![alt text](image.png)

![alt text](image-3.png)

- Click `Next`

![alt text](image-4.png)

- Click `Submit`

![alt text](image-5.png)

- Wait until `Create_Complete`

![alt text](image-6.png)


## Step 2. Configure the blue deployment

- Click `Target Groups`

![alt text](image-7.png)

- Click `Blue`

![alt text](image-8.png)

- Click `Register targets`

![alt text](image-9.png)

- Select `Blue EC2` and Click `Include as pending below`

![alt text](image-11.png)

- Click `Register pending targets`

![alt text](image-12.png)

- Wait `Health status` as Healthy

![alt text](image-10.png)

- Click `Load Balancer`

![alt text](image-13.png)

- Click `BlueGreenALB`

![alt text](image-14.png)

- Click `1 rule`

![alt text](image-15.png)

- Check that the Forward to value is Blue: 100 (100%). 

![alt text](image-16.png)

- Click `BlueGreenALB`

![alt text](image-17.png)

- Copy DNS and paste in new tab

![alt text](image-18.png)

![alt text](image-19.png)

## Step 3. Configure the green deployment ##

- Click `Target Groups`

![alt text](image-7.png)

- Click `Green`

![alt text](image-20.png)

- Click `Register targets`

![alt text](image-21.png)

- Choose `Green EC2`

![alt text](image-22.png)

- Click `Register pending targets`

![alt text](image-23.png)

![alt text](image-24.png)

- Click `Load Balancer`

![alt text](image-13.png)

- Click `BlueGreenALB`

![alt text](image-14.png)

- Select `HTTP:80` and Click `Manage rules` and `Edit rules`

![alt text](image-25.png)

- Select `Default` and Click `Actions` and `Edit rule`

![alt text](image-26.png)

- Click `Add target group`

![alt text](image-27.png)

- Choose `Green` and Update `Weight` and Click `Save change`

![alt text](image-28.png)

- Click `Refresh Icon` 

![alt text](image-29.png)

- Click `BlueGreenALB`

![alt text](image-17.png)

- Copy DNS and paste in new tab

![alt text](image-18.png)

![alt text](image-19.png)

![alt text](image-30.png)

- If stable, change weight to `50%` and test DNS again 

![alt text](image-31.png)

## Step 4. Change route all traffic to the Green target group by using a CloudFormation change set

- In the AWS Management console, on the navigation bar, type and select the `CloudFormation` service.

![alt text](image-32.png)

- Select `Blue-Green-and-Canary-Deployment` and click `Stack actions` and `Create change set for current stack`

![alt text](image-33.png)

- Choose `Edit in Application Composer` and Click `Edit in Application Composer`

![alt text](image-34.png)

- Click `Go to Desinger`

![alt text](image-35.png)

- Select and delete the `BlueWebServer1` and `BlueWebServer2` resources.

![alt text](image-36.png)

- In the `BlueGreenALBListener` resource, in `TargetGroupArn`, change the value to `!Ref GreenTargetGroup`.

![alt text](image-37.png)

![alt text](image-38.png)

- Delete `BlueTargetGroup`

![alt text](image-39.png)

- Click `Refresh`, `Validate` and `Save` Icon

![alt text](image-40.png)

- Select `Replace existing template` and Click `Next`

![alt text](image-41.png)

- Write `RemoveBlueEC2andTargetGroup` and Click `Next`

![alt text](image-42.png)

- Click `Next`

![alt text](image-43.png)

- Click `Submit`

![alt text](image-44.png)

- Check In the `Changes` section,the change will remove the `BlueTargetGroup, BlueWebServer1, and BlueWebServer2` resources, and modify the `BlueGreenALBListener`.

![alt text](image-45.png)

![alt text](image-46.png)

- Click `Execute change set`

![alt text](image-47.png)

- Click `Execute change set` and Wait until `Update_Complete`

![alt text](image-48.png)

![alt text](image-49.png)

- Click `Load Balancer`

![alt text](image-13.png)

- Click `BlueGreenALB`

![alt text](image-14.png)

- Copy DNS and paste in new tab

![alt text](image-18.png)

![alt text](image-50.png)

![alt text](image-30.png)

---

## **Resources & Next Steps**

*   **📦 Full Code Repository:** [AWS Learning Labs](https://github.com/thukhakyawe/aws-learning-labs) - Get the complete, working code from this post
*   **📖 More Deep Dives:** [Whispering Cloud Insights](https://thukhakyawe.hashnode.dev/) - Read other technical articles
*   **💬 Join Discussion:** [DEV Community](https://dev.to/thukhakyawe_cloud) - Share your thoughts and questions
*   **💼 Let's Connect:** [Linkedin](www.linkedin.com/in/thukhakyawe/) - I'd love to connect with you

---