# Lab-How-To-Configure-Cross-Region-Replication-for-an-S3-Bucket

![alt text](Untitled-1.jpg)

### 1. Create an S3 bucket ###

- Type `S3` in AWS Console and click

![alt text](image.png)

- Set Primary Bucket's Region as `Singapore`

![alt text](image-1.png)

- select `Create bucket`.

![alt text](image-3.png)

- Write bucket name as Unique Name

![alt text](image-2.png)

- Click `enable`

![alt text](image-4.png)

- Click `Create bucket`

![alt text](image-5.png)

- Change Region for Backup Bucket 

![alt text](image-6.png)

- select `Create bucket`.

![alt text](image-3.png)

- Write bucket name as Unique Name

![alt text](image-7.png)

- Click `enable`

![alt text](image-4.png)

- Click `Create bucket`

![alt text](image-5.png)

### 2. Configure Cross-Region Replication ###

- Click Primary Bucket

![alt text](image-8.png)

- Click `Managment`

![alt text](image-9.png)

- Click `Create replication rule`

![alt text](image-10.png)

- Write rule name 

![alt text](image-11.png)

- Select `Apply to all objects in the bucket`

![alt text](image-27.png)

- Choose `backup-bucket`

![alt text](image-12.png)

- Choose `Create new role`

![alt text](image-33.png)

> [!NOTE]
> You can create policy and rule by yourself too. If you want, please do the following until created policy and rule:

### 3. Create Policy and Rule at IAM ###

- Type `IAM` in AWS Console and open as new tab

![alt text](image-13.png)

- Click `Policies`

![alt text](image-15.png)

- Click `Create policy`

![alt text](image-16.png)

- Click `json` and delete default policy

![alt text](image-17.png)

- Copy and paste the following policy, edit with `your bucket arn` and click `Next`

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "s3:ListBucket",
                "s3:GetReplicationConfiguration",
                "s3:GetObjectVersionForReplication",
                "s3:GetObjectVersionAcl",
                "s3:GetObjectVersionTagging",
                "s3:GetObjectRetention",
                "s3:GetObjectLegalHold"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::primary-bucket-23042024",
                "arn:aws:s3:::primary-bucket-23042024/*",
                "arn:aws:s3:::backup-bucket-23042024",
                "arn:aws:s3:::backup-bucket-23042024/*"
            ]
        },
        {
            "Action": [
                "s3:ReplicateObject",
                "s3:ReplicateDelete",
                "s3:ReplicateTags",
                "s3:ObjectOwnerOverrideToBucketOwner"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::primary-bucket-23042024/*",
                "arn:aws:s3:::backup-bucket-23042024/*"
            ]
        }
    ]
}

```

- Write policy name and Click `Create Policy`

![alt text](image-18.png)

- Click `Roles`

![alt text](image-14.png)

- Click `Create role`

![alt text](image-19.png)

- Choose S3 and Click `Next`

![alt text](image-20.png)

- Choose `TKK-S3-Cross-Region-Replication-Rule` and Click `Next`

![alt text](image-21.png)

- Write `TKK-S3-Cross-Region-Replication-Rule` and Click `Create role`

![alt text](image-22.png)

![alt text](image-23.png)

- Go back to previous tab and choose `TKK-S3-Cross-Region-Replication-Rule`

![alt text](image-24.png)

- Select `Change the storage class for the replicated objects`  and then in Storage class, select `Standard-IA`.

![alt text](image-25.png)

- Click `Save`

![alt text](image-26.png)

- Click `No, do not replicate existing objects.` and then Click `Submit`

![alt text](image-28.png)

- Select `primary-bucket-23042024`, and then select `Upload`.

![alt text](image-29.png)

- Click `Add files`

![alt text](image-30.png)

- select any file on your local computer, and click `Open` and then click `Upload`

![alt text](image-31.png)

- Click `Close`

![alt text](image-32.png)

- On the Buckets page, select `backup-bucket-23042024` to view the file you just uploaded.

![alt text](image-34.png)


>  Congratulations, you have completed  *Lab-How To Configure Cross-Region Replication for an S3 Bucket* 

***