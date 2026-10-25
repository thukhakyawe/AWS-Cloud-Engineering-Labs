![alt text](image.png)

## 1. Create an S3 bucket ##

- Click `Create bucket`

![alt text](image-1.png)

- Set Bucket Name as Unique Name

![alt text](image-2.png)

- Click `Create bucket`

![alt text](image-3.png)

- Click `Upload`

![alt text](image-4.png)

- Click `Add files`

![alt text](image-5.png)

- Choose `Your File` and Click `Upload`

![alt text](image-6.png)

- Click `Test.txt`

![alt text](image-7.png)

- Check at `Storage class`

![alt text](image-8.png)

- Click `Your bucket name`

![alt text](image-9.png)

- Click `Managment` and `Create lifecycle rule`

![alt text](image-10.png)

- Write `lifecycyle-rule` at Lifecycle rule name
- Select `Apply to all objects in the bucket`
- Click `I acknowledge that this rule will apply to all objects in the bucket.`

![alt text](image-11.png)

- Select `Lifecycle rule actions`like this and choose `Choose storage class transitions`and write `Days after object creation` at `Transition current versions of objects between storage classes`

![alt text](image-12.png)

- Write `days` at `Days after object creation` and `Permanently delete noncurrent versions of objects `

![alt text](image-13.png)

- Click `Create rule`

![alt text](image-14.png)

-----------------
***That it is. Congratulations, you have completed Lab-How To Create an Object Lifecycle Policy for Amazon S3***

-----------------
