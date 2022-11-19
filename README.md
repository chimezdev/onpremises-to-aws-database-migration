# ADVANCED DEMO ON DATABASE MIGRATION USING DATABASE MIGRATION SERVICE

## Migrate an OnPremises web application with a self-managed mariaDB database server into EC2 webserver and RDS-MariaDB database on AWS.
The on-premises environment is simulated using EC2 likewise the database.
Original work by [Adrian Cantrill](https://www.youtube.com/@LearnCantrill/videos)

- States of this Demo
    - STAGE 1: Provision the environment using CloudFormation via a One-click deployment.
    - STAGE 2: Establish Private Connectiviy between the environments via VPC Peering
    - STAGE 3: Create and Configure the AWS side of the infrastructure
    - STAGE 4: Migrate Database and Cutover
    - STAGE 5: Account cleanup

## STAGE 1
- Login to the AWS Console as user with admin permission and make sure to be on `us-east-1 ` region.
[Click](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/quickcreate?templateURL=https://learn-cantrill-labs.s3.amazonaws.com/aws-dms-database-migration/DMS.yaml&stackName=DMS) to provision the base infrastructure.
- Take note of the values of the following ***parameters***
    - DBName
    - DBPassword
    - DBRootPassword
    - DBUser
- Scroll to the bottom, check the capabilities box and click ***Create Stack***
- Wait for the stack to be ***CREATE_COMPLETE*** state
- Navigate to the *EC2* console click on *Running Instaces*, the two instances *CatWEB* and *CatDB* will be running.
- Select the *CatWEB*, copy and open in a new tab the ***Public IPv4 DNS***.
- This will load the web application running on the on-premises environment.
![Webpage served from onpremises](https://github.com/chimezdev/onpremises-to-aws-database-migration/blob/main/Images/webpage.png)

## STAGE 2 - Create a VPC peering connection between the On-premises and AWS Environments
In production, a Direct Connect or a VPN would be used but here you are going to simulate it using VPC Peering to configure the connection between the two environments.
- Navigate to the AWS VPC console and click on ***Your VPCs***
- In addition to the default vpc, you will see the ***awsVPC*** and ***onpremVPC*** running in **IPv4 CIDR** ranges ***10.16.0.0/16*** and ***192.168.10.0/24*** respectively.
- Creating Peering Connections
    - On the left menu, scroll down and click on ***Peering connections*** then click on ***Create peering connection***
    - enter *A4L-ON-PREMISES-TO-AWS* as name tag
    - choose *onpremVPC* as *VPC(Requester)* and *awsVPC* as *VPC(Accepter)*
    - scroll down and click ***Create Peering Connection***
    - both the *Requester* and Accepter VPCs are on the same AWS account, as such click on *Actions* then *Accept Request*

