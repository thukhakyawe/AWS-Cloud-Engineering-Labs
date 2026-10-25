![alt text](Untitled.jpg)


### 1. Create IAM Role for System Manager Permissions ###

- Choose `IAM` from AWS Console

![alt text](image.png)

- Click `Create Role`

![alt text](image-1.png)

- Choose `EC2`

![alt text](image-2.png)

- Choose `EC2 Role for AWS Systems Manager`
- Click `Next`

![alt text](image-3.png)

- Click `Next`

![alt text](image-5.png)

- Set Role Name `TKK-AmazonSSMManagedInstanceCore`

![alt text](image-6.png)

- Click `Create role`

![alt text](image-7.png)

- Role is created.

![alt text](image-8.png)


### 2. Create Linux Instance ###

- Create Linux Instance by using Ubuntu 22.04 and wait for finished creation.

### 3. Create Window Instance ###

- Create Window Instance by using Window Server 2022 and wait for finished creation.

![alt text](image-9.png)

### 4. Attach IAM roles to Instances ###

- Select `Instance` 
    - Click `Actions`
        - Click `Security`
            - Click `Modify IAM role`

![alt text](image-11.png)

- Select `TKK-AmazonSSMManagedInstanceCore`
    - Click `Update IAM role`

![alt text](image-12.png)

- Click `Stop Instance`

![alt text](image-13.png)

- Click `Start Instance`

![alt text](image-14.png)

- Follow these steps to `Linux Instance` too.

### 5. Check Your Instances are appeared at System Manager

- Go to `Systems Manager` 

![alt text](image-15.png)

- Go to `Fleet Manager`

![alt text](image-16.png)

![alt text](image-17.png)

- Done, You can proceed next steps.

### 6. Create new patch baselines ###

- In the navigation bar, type `Systems Manager` into the search box, and then select `Systems Manager`
- In the navigation pane, select `Patch Manager`

![alt text](image-18.png)


- In the navigation pane, select `Patch Manager`, and then select `Patch baselines`
- In Patch baselines, select `Create patch baseline`

![alt text](image-19.png)


- Name - `Linux-Ubuntu-custombaseline-TKK`
- Description - `Custom patch baseline for Ubuntu`
- Operating System - Select `Ubuntu`
- Products - Select `All`
- Compliance reporting - Select `Critical`
- Section - Select `All`
- Priority - Select `Important`
- Click `Create patch baseline`

![alt text](image-21.png)

- In Patch baselines, select `Create patch baseline`
- Name - `Win2022-DefenderAV-custombaseline-TKK`
- Description - `Custom patch baseline for Window Server 2022`
- Operating System - Select `Windows`
- Products - Select `All`
- Approve patches after a specified number of days - `5`
- Classification - Select `CriticalUpdates`, `DefinitionUpdates` and `SecurityUpdates`
- Compliance reporting - Select `Critical`
- Severity - Select `Critical`, `Important`
- Click `Create patch baseline`

![alt text](image-22.png)

- Check that you have created custombaseline for both instances.

![alt text](image-23.png)


### 7. Enable amazon EC2 OpsData source in Explorer and set up recording in AWS Config ###

![alt text](image-24.png)

- Click `Get started`

![alt text](image-25.png)


- Click `Enable Explorer`

![alt text](image-26.png)


### 8. Enable AWS Config

- Click `Get started`

![alt text](image-27.png)

- Use Default Setting and Click `Next`

![alt text](image-28.png)


-Choose `EC2` and Click checkbox to select all rules

![alt text](image-29.png)

- Click `Next`

- Click `Confirm`

![alt text](image-30.png)


![alt text](image-31.png)

- Click `Create`

![alt text](image-32.png)

### 9. Add a patch group to a patch baseline ###

- In Patch baselines, search for and select `Linux-Ubuntu-custombaseline-TKK`, and then on the Actions menu, select Modify patch groups.



------------------------------------

# How do you turn off AWS config #


- 1. Turn off Recording for that region using the console

- 2. Delete the Rule by going to actions, delete rule

- 3. Use the AWS CLI and delete the default recording by

`aws configservice delete-configuration-recorder --configuration-recorder-name default --region <region-name>`

- 4. Delete the service linked role created for AWS Config
