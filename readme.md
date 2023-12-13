Lab - Using Auto Scaling in AWS (Linux)
# Using Auto Scaling in AWS (Linux)
This lab walks you through using the Elastic Load Balancing (ELB) and Auto Scaling services to load balance and automatically scale your infrastructure.


Scenario

In this lab, you will create the scalable web server system shown in the following diagram:

ScalingArchitecture

## Objectives After completing this lab, you will be able to:

Use Auto Scaling to scale up the number of servers available for a specific task when other servers are experiencing heavy load.
The following components are created for you as a part of the lab environment:

Amazon VPC
Public Subnets
Private Subnets
Amazon EC2 - Command Host (in the public subnet), you will log in to this instance to create a few of your AWS assets.

You will create the following components for this lab:

Amazon EC2 - Web Server
Auto Scaling Launch Template
Auto Scaling Group
Auto Scaling Policies
ALB


Task 1: Configure the CLI and create an EC2 instance
In this task, you will launch a new EC2 instance. You will use the AWS CLI tools on the Command Host to perform all of these operations.

The following instructions vary slightly depending on whether you are using Windows or Mac/Linux.

 WINDOWS USERS: USING SSH TO CONNECT
 These instructions are specifically for Windows users. If you are using macOS or Linux, skip to the next section.

From the Lab Information pane, select the PPK button and save the Ec2KeyPair-PPK.ppk file. Typically your browser will save it to the Downloads directory.

Make a note of the Command Host address.

Download PuTTY to SSH into the Amazon EC2 instance. If you do not have PuTTY installed on your computer, download it here.

Open putty.exe

Configure PuTTY timeout to keep the PuTTY session open for a longer period of time.:

Select Connection
Set Seconds between keepalives to 

30
Configure your PuTTY session:
Select Session
Host Name (or IP address): Paste the Public DNS or IPv4 address of the CommandHost IP.
Back in PuTTY, in the Connection list, expand  SSH
Select Auth (don’t expand it)
Select Browse
Browse to and select the lab#.ppk file that you downloaded
Select Open to select it
Select Open again.
Select Yes, to trust and connect to the host.

When prompted login as, enter: 

ec2-user
 This will connect you to the EC2 instance.

Windows Users: Select here to skip ahead to the next task.

​

MACOS  AND LINUX  USERS
These instructions are specifically for Mac/Linux users.

From the Lab Information pane, select the PEM button and save the Ec2KeyPair-PEM.pem file.

Make a note of the Command Host \ Bastion Host address, if it is displayed.

Open a terminal window, and change directory 

cd
 to the directory where the Ec2KeyPair-PEM.pem file was downloaded. For example, if the Ec2KeyPair-PEM.pem file was saved to your Downloads directory, run this command:


cd ~/Downloads
Change the permissions on the key to be read-only, by running this command:

chmod 400 Ec2KeyPair-PEM.pem
Run the below command (replace <public-ip> with the Command Host \ Bastion Host address you copied earlier). Alternatively, return to the EC2 Console and select Instances. Check the box next to the instance you want to connect to and in the Description tab copy the IPv4 Public IP value.:

ssh -i Ec2KeyPair-PEM.pem ec2-user@<public-ip>
Type 

yes
 when prompted to allow the first connection to this remote SSH server. Because you are using a key pair for authentication, you will not be prompted for a password.​
CONFIGURE THE AWS CLI
Discover the region in which the Command Host instance is running:

curl http://169.254.169.254/latest/dynamic/instance-identity/document | grep region
You will use this region information in the next step.

Update the AWS CLI software with the credentials.

aws configure
At the prompts, enter the following information:
AWS Access Key ID: Press enter.
AWS Secret Access Key: Press enter.
Default region name: Type in the name of the region, which you just discovered a moment ago. For example, 

us-east-1
 or 

eu-west-2
.
Default output format: 

json
Now you are ready to access and run the scripts detailed in the exercises below. To access these scripts you will first need to navigate to their directory by issuing the following command.

cd /home/ec2-user/
CREATE A NEW EC2 INSTANCE
Now that you are logged in to CommandHost, you will use the AWS CLI to create a new instance that hosts a web server.

Inspect the script UserData.txt that was installed for you as part of the CommandHost.

more UserData.txt
This script performs a number of initialization tasks, including updating all installed software on the box and installing a small PHP web application that you can use to simulate a high CPU load on the instance. Near the bottom of the script, you will see the following lines:


find -wholename /root/.*history -wholename /home/*/.*history -exec rm -f {} \;
find / -name 'authorized_keys' -exec rm -f {} \;
rm -rf /var/lib/cloud/data/scripts/*
These lines erase any history or security information that may have accidentally been left on the instance when the image was taken.

Use the KEYNAME, AMIID, SUBNETID and HTTPACCESS values you made a note of in the Accessing the AWS Management Console section and paste it into relevant sections of the below script. Note: You can query these values by viewing the Lab information at the left.

aws ec2 run-instances --key-name KEYNAME --instance-type t3.micro --image-id AMIID --user-data file:///home/ec2-user/UserData.txt --security-group-ids HTTPACCESS --subnet-id SUBNETID --associate-public-ip-address --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=WebServerBaseImage}]' --output text --query 'Instances[*].InstanceId'
The output of this command will provide you with an InstanceId. This value is referred to as new-instance-id in subsequent steps and should be replaced appropriately.

Use the aws ec2 wait instance-running command to monitor this instance’s status. Replace NEW-INSTANCE-ID with the value you copied previously.

aws ec2 wait instance-running --instance-ids NEW-INSTANCE-ID
Wait for the command to return to a prompt, before proceeding to the next step.

Your instance should have started a new web server. To test that the web server was installed properly, obtain the public DNS name. Copy the output of this value (minus quotation marks). This value is referred to as public-dns-address in the next procedure. Replace NEW-INSTANCE-ID with the value you copied previously.

aws ec2 describe-instances --instance-id NEW-INSTANCE-ID --query 'Reservations[0].Instances[0].NetworkInterfaces[0].Association.PublicDnsName'
Use a Web browser to navigate to the address returned by the command above.
It could take a few minutes for the web server to be installed. Please wait for five minutes before trying other steps.

Do not click on Start Stress at this stage. Replace PUBLIC-DNS-ADDRESS with the value you copied in the last step.


http://PUBLIC-DNS-ADDRESS/index.php
If your web server does not appear to be running, check with your instructor.

Task 2: Create an Auto Scaling Environment
In this section, you will create a load balancer that pools a group of EC2 instances under a single DNS address. You will use Auto Scaling to create a dynamically scalable pool of EC2 instances based on the image that you created in the previous section. Finally, you will create a set of alarms that will scale out or scale in the number of instances in your load balancer group whenever the CPU performance of any machine within the group exceeds or falls below a set of specified thresholds.

The following task can be performed using either the AWS CLI or the Management Console. For purposes of simplicity, you will implement this task using the Management Console.

CREATE AN APPLICATION LOAD BALANCER
Load Balancing

On the  Services menu, click EC2.

In the left navigation pane, click Load Balancers (you might need to scroll down to find it).

Click Create load balancer

Under Application Load Balancer, click Create

For Name, enter: 

webserverloadbalancer

Scroll down to the Availability Zones section, for VPC, select: Lab VPC

You will now specify which subnets the Load Balancer should use. It will be an Internet-facing load balancer, so you will select both Public Subnets.

Click the first displayed Availability Zone, then click the Public Subnet 1 displayed underneath.

Click the second displayed Availability Zone, then click the Public Subnet 2 displayed underneath.

You should now have two subnets selected: Public Subnet 1 and Public Subnet 2. (If not, go back and try the configuration again.)

Under Security groups. Select  HTTPAccess and deselect  default.
Routing configures where to send requests that are sent to the load balancer. You will create a Target Group that will be used by Auto Scaling.

Under Listeners and routing choose Create target group.

For Target group name, enter: 

LabGroup

For Health check path, enter: 

/index.php

Expand  Advanced health check settings.

The Application Load Balancer automatically performs Health Checks on all instances to ensure that they are responding to requests. The default settings are recommended, but you will make them slightly faster for use in this lab.

Configure these values:
Healthy threshold: 

2
Unhealthy threshold: 

2
Interval: 

10
This means that the Health Check will be performed every 10 seconds and if the instance responds correctly twice in a row, it will be considered healthy.

Click Next
Targets are the individual instances that will respond to requests from the Load Balancer. You can skip this step.

Choose Create target group

Switch back to the Load Balancer tab. Under the Forward to drop-down select 

LabGroup
.

Choose Create load balancer

The load balancer will show a state of provisioning. There is no need to wait until it is ready. Please continue with the next task.

CREATE A LAUNCH CONFIGURATION
The launch configuration will be used by your Auto Scaling group to specify which AMI to use to create new EC2 instances.

In the left navigation pane, click Launch Templates.

Click Create launch template

Configure these settings:

Launch template name: 

WebServerLaunchConfiguration

Under Application and OS Images (Amazon Machine Image)/Quick Start choose Amazon Linux.

Under Amazon Machine Image (AMI) choose Amazon Linux 2 AMI.

Instance type: t3.micro.

Key pair name: Don’t include in launch template.

Security groups: HTTPAccess. Make sure the security group belongs to Lab VPC.

Expand Advanced details and in the Detailed CloudWatch monitoring menu, select Enable

Under User Data paste in the following:


#!/bin/bash
yum update -y --security
amazon-linux-extras install epel -y
yum -y install httpd php stress
systemctl enable httpd.service
systemctl start httpd
cd /var/www/html
wget http://aws-tc-largeobjects.s3.amazonaws.com/CUR-TF-100-TULABS-1/10-lab-autoscaling-linux/s3/ec2-stress.zip
unzip ec2-stress.zip

echo 'UserData has been successfully executed. ' >> /home/ec2-user/result
find -wholename /root/.*history -wholename /home/*/.*history -exec rm -f {} \;
find / -name 'authorized_keys' -exec rm -f {} \;
rm -rf /var/lib/cloud/data/scripts/*
Click Create launch template
CREATE AN AUTO SCALING GROUP
Your Auto Scaling group will create a minimum number of Amazon EC2 instances that will reside behind your load balancer. In subsequent procedures, you will also add scale-out and scale-in policies that increase or decrease the number of running instances in reaction to alarms triggered by Amazon CloudWatch.

Click View launch templates
You will now create an Auto Scaling group that uses this Launch Template.

Select  WebServerLaunchConfiguration and then in the Actions  menu, select Create Auto Scaling group

Configure the following settings:

Auto Scaling group name: 

WebServersASGroup

Click Next

Network: Lab VPC

Subnet: Select Private Subnet 1 (10.0.1.0/24) and Private Subnet 2 (10.0.3.0/24)

Click Next

Load balancing

Select  Attach to an existing load balancer

Select  Choose from your load balancer target groups.

Existing load balancer target groups LabGroup

Monitoring: Select  Enable group metrics collection within CloudWatch.

This will capture metrics at 1-minute intervals, which allows Auto Scaling to react quickly to changing usage patterns.

Click Next

Group size: Enter the below values

Desired capacity: 

2

Minimum capacity: 

2

Maximum capacity: 

4

This will allow Auto Scaling to automatically add/remove instances, always keeping between 2 and 4 instances running.

Scaling policies - optional

Select  Target tracking scaling policy

Metric type: Average CPU Utilization

Target value: 

40

This tells Auto Scaling to maintain an average CPU utilization across all instances at 40%. Auto Scaling will automatically add or remove capacity as required to keep the metric at, or close to, the specified target value. It adjusts to fluctuations in the metric due to a fluctuating load pattern.

Click Next

Click Next again on the Add notifications section.

Add tags: click Add tag and enter

Key: 

Name

Value: 

WebApp

Click Next

Finally click Create Auto Scaling group

This will launch EC2 instances in private subnets across both Availability Zones.

Your Auto Scaling group will initially show an instance count of zero, but new instances will be launched to reach the Desired count of 2 instances. Note: If you experience an error related to the t3.micro instance type not being available, then rerun this task by selecting t2.micro instead.

VERIFYING THE AUTO SCALING CONFIGURATION
In this task, you will verify that both the Auto Scaling configuration and the load balancer are working by accessing a pre-installed script on one of your servers that will consume CPU cycles, thus triggering the scale out alarm.

In the left navigation pane, click Instances.

Verify that two new instances labelled WebApp are being created as part of your Auto Scaling group.

Wait for the two new instances to complete initialization before you proceed to the next step. Observe the Status Checks field for the instances until it shows that both status checks have completed successfully.

In the left navigation pane, click Target Groups, and then select  your target group (LabGroup).

On the Targets tab in the lower half of your screen, verify that two instances are being created. Keep refreshing this list until the Status of these instances changes to healthy.

You can now test the web application by accessing it via the Load Balancer.

In the left navigation pane, click Load Balancers and then select  webserverloadbalancer.

On the Details tab below, copy the DNS name value (which should look like webserverloadbalancer-xxxxxxxxx.xx-xxxx-x.elb.amazonaws.com). This value will be referred to as load-balancer-url in the next step.

Open a new web browser tab and paste the URL into the address bar, then press Enter.

On the web page, click Start Stress.

This will call the application stress in the background, causing the CPU utilization on the instance that serviced this request to spike to 100%.

In the left navigation pane of the management console, click Auto Scaling Groups.

Select  WebServerASGroup.

Select the Activity tab for your Auto Scaling group. After a few minutes, you should see your Auto Scaling group add a new instance.

This is because CloudWatch detected that the average CPU utilization of your Auto Scaling group exceeded 40%, and your scale-up policy has been triggered in response.

End lab