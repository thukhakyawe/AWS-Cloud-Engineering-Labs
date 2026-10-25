
## Step 1. Create an ECR repository

- In the AWS navigation bar, type `ECR` into the search box, and then select `Elastic Container Registry service`

![alt text](image.png)

- Click `Repositories`

![alt text](image-1.png)

- Click `Create repository`

![alt text](image-2.png)

- Write Repository name 

![alt text](image-3.png)

- Click `Create repository`

![alt text](image-4.png)

- In the AWS navigation bar, type `cloudshell` into the search box, and then click

![alt text](image-5.png)

- Click `Close`

![alt text](image-6.png)

- Create new folder by typing the following command

```
mkdir ~/labmicroservice
```

![alt text](image-7.png)

- type the following command

```
cd ~/labmicroservice
```

![alt text](image-8.png)

- Create `Dockerfile` by using VIM Editor

```
vim Dockerfile
```

- Copy and paste the following command in `Dockerfile` and save

```
FROM public.ecr.aws/amazonlinux/amazonlinux:latest

# Install dependencies
RUN yum update -y && \
yum install -y httpd

# Install the cowsay binary
RUN dnf install --assumeyes cowsay

# Install apache and write a web page
RUN echo "<html><pre style=\"line-height:100%\">" > /var/www/html/index.html
RUN cowsay 'We have a LOT of milk in stock!' >> /var/www/html/index.html
RUN echo "</pre></html>" >> /var/www/html/index.html

# Configure apache
RUN echo 'mkdir -p /var/run/httpd' >> /root/run_apache.sh && \
echo 'mkdir -p /var/lock/httpd' >> /root/run_apache.sh && \
echo '/usr/sbin/httpd -D FOREGROUND' >> /root/run_apache.sh && \
chmod 755 /root/run_apache.sh

EXPOSE 80

CMD /root/run_apache.sh
```

![alt text](image-10.png)

- Run the following command

```
docker build --tag labmicroservice .
```

![alt text](image-9.png)

## Step 2. Deploy a Docker container to ECR

- Type the following command and press `Enter`

```
REPO=labmicroservice

ACCOUNT=$(aws sts get-caller-identity --query "Account" --output text)

ECR=${ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com
aws ecr get-login-password | docker login --username AWS --password-stdin ${ECR}

docker tag labmicroservice ${ECR}/${REPO}

docker push ${ECR}/${REPO}

```

![alt text](image-11.png)

- Type the following command

```
aws ecr list-images --repository-name labmicroservice --region ${AWS_REGION}
```

![alt text](image-12.png)

- Click `Repositories`

![alt text](image-1.png)

- Click `labmicroservice`

![alt text](image-13.png)

- Click `latest`

![alt text](image-14.png)

- Copy this URL

![alt text](image-15.png)

## Step 3. Create an ECS cluster using Amazon Fargate

- Type `Elastic Container Service` in Services and click

![alt text](image-16.png)

- Click `Create cluster`

![alt text](image-17.png)

- Type Cluster name and Choose `AWS Fargate (serverless)` then Click `Create`

![alt text](image-18.png)

![alt text](image-19.png)

![alt text](image-20.png)

- Click `Task definitions`

![alt text](image-21.png)

- Click `Create new task definition` and choose `Create new task definition`   

![alt text](image-22.png)

![alt text](image-23.png)

- Type name and paste ULR you copied earlier

![alt text](image-24.png)

- Clear `Use log collection`

![alt text](image-25.png)

- Click `Create`

![alt text](image-26.png)

- Select `Create service`

![alt text](image-27.png)

- Select `Launch type`

![alt text](image-28.png)

- Type `Service name`

![alt text](image-29.png)

- Select `VPC` and Choose `All subnets` and `Security Group`

- Click `Create`

![alt text](image-30.png)

![alt text](image-31.png)

## Step 4. Test the container application ##

- Click `Task` and Click on task name

![alt text](image-32.png)

- Copy `Public ip` and open in new browser

![alt text](image-33.png)

![alt text](image-34.png)


---

## **Resources & Next Steps**

*   **📦 Full Code Repository:** [AWS Learning Labs](https://github.com/thukhakyawe/aws-learning-labs) - Get the complete, working code from this post
*   **📖 More Deep Dives:** [Whispering Cloud Insights](https://thukhakyawe.hashnode.dev/) - Read other technical articles
*   **💬 Join Discussion:** [DEV Community](https://dev.to/thukhakyawe_cloud) - Share your thoughts and questions
*   **💼 Let's Connect:** [Linkedin](www.linkedin.com/in/thukhakyawe/) - I'd love to connect with you

---