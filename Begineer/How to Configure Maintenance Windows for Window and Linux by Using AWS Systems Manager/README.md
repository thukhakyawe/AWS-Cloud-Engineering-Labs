![alt text](Untitled.jpg)


## 1. Create EC2 for Window and Linux ##

- Use `Microsoft Windows Server 2022` for Window

![alt text](image.png)

- Use `Ubuntu Server 22.04 LTS` for Ubuntu

![alt text](image-1.png)

- It will take some minutes to set up

![alt text](image-2.png)

- Modify IAM role for `TKK-Window`

![alt text](image-10.png)

- Choose `SSM Role` and Click `Update IAM role`

![alt text](image-11.png)

- Modify IAM role and Update IAM role for `TKK-Linux` too

![alt text](image-12.png)

## 2. Create a maintenance window at Systems Manager ##

- In the navigation bar, type `Systems Manager` into the search box, and then select the `Systems Manager`.

![alt text](image-3.png)

- In the navigation pane, in Change Management, `select Maintenance Windows`.

![alt text](image-4.png)

- Select `Create Maintenance Window`.

![alt text](image-5.png)

- In Name, write `Patching-Maintenance-Window`
- In Description, enter `Patching Maintenance Window - weekly every Sunday at 1 AM`.
- In Schedule, select `Rate schedule builder`.
- In Window starts, enter `7`, and then select `Day(s)`.
- In Duration, enter `6` hours.
- In Stop initiating tasks, enter `2` hour before the window closes.
- In Window start date, select the calendar, and then select the first Sunday after the current date.
- In Time, enter 01:00:00, and then in timezone, select your current timezone.
- In Schedule time zone, select your current timezone.
- Select Create maintenance window.

![alt text](image-6.png)

- Select `Create Maintenance Window`.

![alt text](image-7.png)

- In Name, write `SSMAgentupdate-Maintenance-Window`
- In Description, enter `SSMAgentupdate-Maintenance-Window - every 1 hour`.
- In Schedule, select `Rate schedule builder`.
- In Window starts, enter `1`, and then select `Hour(s)`.
- In Duration, enter `6` hours.
- In Stop initiating tasks, enter `0` hour before the window closes.

![alt text](image-8.png)

## 3.Register targets to the maintenance windows ##

- In Maintenance windows, select `Patching-Maintenance-Window`, and then on the Actions menu, select `Register targets`.

![alt text](image-9.png)

- In Target name, enter `PatchTargets`
- In Target selection, select `Choose instances manually`
- Select `Linux and Windows`, and then select `Register target`

![alt text](image-13.png)

- In Maintenance windows, select `SSMAgentupdate-Maintenance-Window`, and then on the Actions menu, select `Register targets`.

![alt text](image-14.png)

- In Target name, enter `SSMAgentTarget`
- In Target selection, select `Choose instances manually`
- Select `Linux and Windows`, and then select `Register target`

![alt text](image-15.png)

## 4. Add a task to a maintenance window ##

- On the Maintenance windows page, select 	`Patching-Maintenance-Window`
- On the Actions menu, select `Register Run command task`

![alt text](image-16.png)

- In Name, enter `RunPatchBaseline-task`
- In Command document, search for and select 	`AWS-RunPatchBaseline`
- In Targets, ensure that `Selecting registered target groups is selected`, and then select the `PatchTarget` check box.
- In Rate control, in Concurrency, in targets, enter `2`, and then in Error threshold, in errors, enter `2`
- In Parameters, In Operation, select `Install`
- Review the remaining default settings, and then select `Register Run command task`

![alt text](image-17.png)

![alt text](image-18.png)

![alt text](image-19.png)

![alt text](image-20.png)

![alt text](image-21.png)

![alt text](image-22.png)

- On the Maintenance windows page, select 	`SSMAgentupdate-Maintenance-Window`
- On the Actions menu, select `Register Run command task`

![alt text](image-23.png)

- In Name, enter `UpdateSSMAgent-task`
- In Command document, search for and select 	`AWS-UpdateSSMAgent`
- In Targets, ensure that `Selecting registered target groups is selected`, and then select the `SSMAgentTarget` check box.
- In Rate control, in Concurrency, in targets, enter `2`, and then in Error threshold, in errors, enter `2`
- Review the remaining default settings, and then select `Register Run command task`

![alt text](image-24.png)

![alt text](image-25.png)

![alt text](image-26.png)

![alt text](image-27.png)

![alt text](image-28.png)

## 5. Verify that the Run Command task completes during the maintenance window ##

- On the Maintenance windows page, select 	`SSMAgentupdate-Maintenance-Window`
- On the History tab, verify that there are executions every 1 hour.
- Verify that the AWS-UpdateSSMAgent Run Command task has a status of Success.

![alt text](image-29.png)

------------------------------------------
**That it is. Congratulations, you have completed *How to Configure Maintenance Windows for Window and Linux by Using AWS Systems Manager***