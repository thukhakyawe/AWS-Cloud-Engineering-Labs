
- Open the `EC2` console page from AWS Console

![alt text](image.png)

- Click on `Lifecycle Manager` on the left side menu and there is no lifecycle policies defined on the welcome screen.

![alt text](image-1.png)

- Click on `Snapshots` on the left side menu

![alt text](image-2.png)

- Filter the snapshots using the "environment" tag key and you will see there is no snapshot

![alt text](image-3.png)

- Click on `Lifecycle Manager` on the left side menu and click `Next step`

![alt text](image-1.png)

- Click on `Target resource tags` and select `environment` and `prod` as the tag key 

![alt text](image-4.png)

- Click `Add`

![alt text](image-8.png)

- Write prod-snapshot-policy at `Policy description`

![alt text](image-6.png)

- Select `Choose another role` and select `AWSDataLifecycleManagerDefaultRole`

![alt text](image-5.png)

- Click `Next`

![alt text](image-7.png)

- Fill the information:

- `Schedule name`- `Schedule 1`
- `Frequency` - `Daily`
- `Every` - `1 hour`
- `Starting at` - `09:00`
- `Retention type` - `Age`
- `Expire from standard tier` - `1`
- `after creation` - `days`

and Click checkbox of `Copy tags from source`

![alt text](image-9.png)

- Click `Review policy`

![alt text](image-10.png)

- Click Create policy`

![alt text](image-11.png)

- Now The policy should be created like this

![alt text](image-12.png)

- Let create another policy for `environment:dev`

- Select `Custom policy`

- Select `EBS snapshot policy`

- Click `Next`

![alt text](image-13.png)

- - Click on `Target resource tags` and select `environment` and `dev` as the tag key and click `Add`

![alt text](image-14.png)

- Write dev-snapshot-policy at `Policy description`

![alt text](image-15.png)

- Select `Choose another role` and select `AWSDataLifecycleManagerDefaultRole`

![alt text](image-5.png)

- Click `Next`

![alt text](image-7.png)

- Fill the information:

- `Schedule name`- `Schedule 1`
- `Frequency` - `Daily`
- `Every` - `1 hour`
- `Starting at` - `09:00`
- `Retention type` - `Age`
- `Expire from standard tier` - `1`
- `after creation` - `days`

and Click checkbox of `Copy tags from source`

![alt text](image-9.png)

- Click `Review policy`

![alt text](image-10.png)

- Click Create policy`

![alt text](image-11.png)

- Now The policy should be created like this

![alt text](image-16.png)

- After a couple of hours of the prod and dev snapshot policies being created, you should be able to see the first snapshots created by Lifecycle Manager. The snapshots will have a tag key dlm:managed set to a value of "true".

---

## **Resources & Next Steps**

*   **📦 Full Code Repository:** [AWS Learning Labs](https://github.com/thukhakyawe/aws-learning-labs) - Get the complete, working code from this post
*   **📖 More Deep Dives:** [Whispering Cloud Insights](https://thukhakyawe.hashnode.dev/) - Read other technical articles
*   **💬 Join Discussion:** [DEV Community](https://dev.to/thukhakyawe_cloud) - Share your thoughts and questions
*   **💼 Let's Connect:** [Linkedin](www.linkedin.com/in/thukhakyawe/) - I'd love to connect with you

---