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

## STAGE 2A - Create a VPC peering connection between the On-premises and AWS Environments
In production, a Direct Connect or a VPN [used here](https://github.com/chimezdev/aws-bgp-site2site-vpn-connection) would be used but here you are going to simulate it using VPC Peering to configure the connection between the two environments.
- Navigate to the AWS VPC console and click on ***Your VPCs***
- In addition to the default vpc, you will see the ***awsVPC*** and ***onpremVPC*** running in **IPv4 CIDR** ranges ***10.16.0.0/16*** and ***192.168.10.0/24*** respectively.
![Created VPCs](https://github.com/chimezdev/onpremises-to-aws-database-migration/blob/main/Images/Your_VPCs.png)
- Creating Peering Connections
    - On the left menu, scroll down and click on ***Peering connections*** then click on ***Create peering connection***
    - enter *A4L-ON-PREMISES-TO-AWS* as name tag
    - choose *onpremVPC* as *VPC(Requester)* and *awsVPC* as *VPC(Accepter)*
    - scroll down and click ***Create Peering Connection***
    - both the *Requester* and Accepter VPCs are on the same AWS account, as such click on *Actions* then *Accept Request*

## STAGE 2B - Configure Routing in each VPC to send traffic to each other
1. Configure Routes on the On-premises side VPC route by editing the route table
    - click on ***Route tables*** on the left menu of the VPC console
    - select the `onpremPublicRT`, click on the `Routes` tab and then click on `Edit routes` 
    - click on `Add route`, copy and paste the *IPv4 CIDR* of the `awsVPC`
    - as Target, click on the dropdown and select ***Peering Connection to automatically select the peering connection created earlier.
    - click on `Save changes`

2. Configure Route on the AWS Side VPC by editing the route table
    there are 2 route tables on the AWS Side, (Private and Public)
    - select the `awspublicRT` and repeat the above steps to add a route using the `onpremVPC` `IPv4 CIDR` 
    - repeat the process for the `awsPrivateRT` using the `onpremVPC` `IPv4 CIDR`
At the stage we have configured gateway object and route table in both VPCs so both environment can communicate 
![Route tables](https://github.com/chimezdev/onpremises-to-aws-database-migration/blob/main/Images/Route_table.png)

## STAGE 3A- Create the RDS Instance
- Navigate to AWS RDS console
- Create subnet group
    - click on ***Subnet groups*** on the left menu and click on ***Create DB Subnet group*** this is where RDS will launch the instances
    - enter `A4LDBSNGROUP` as **name** and **description
    - choose `awsVPC` from the dropdown and select `us-east-1a` and `us-east-1b` under **availability zones.
    - on the VPC console - subnets, there is a public and private subnets created earlier named `aws-privateA` and `aws-privateB` with `10.16.32.0/20` and `10.16.96.0/20` as IPv4 CIDR respectively
    - under **subnets** select the two subnets above (remember that our database is not accessible to the public therefore the instances should be launched in private subnet.)
    - scroll down and click on `Create`
- Creating the Database
    - Click on `Databases` - `Create database`
    - select `standard create` `MariaDB` as **Engine option** and `free tier` **Template**
    - goto the ***parameters** of the **CloudFormation** - **Stacks** console. there you will copy the parameter values to populate the DB Setting.
    - enter `a4lwordpress` as **DB instance identifier** & **Master username**
    - enter the DBPassword parameter value which is `cats-dogs-rabbits-chickens` as **Master password** and confirm
    - under **Connectivity**, ensure the option `Don’t connect to an EC2 compute resource` is selected.
    click on `Virtual private cloud (VPC)` dropdown and select `awsVPC`
    - ensure the subnet group ` a4ldbsngroup` we created is selected and choose `No` for **public access**
    - select `Choose existing` for **VPC security group (firewall)** and select `DMS-awsSecurityGroupDB-*`
    - don't forget to remove the defaul security group
    - scroll down and expand **Additional Configuration**
    - use `a4lwordpress` for ***Initial database name***
    - scroll down and click `Create database`

## STAGE 3B - Create the EC2 Instance
- Navigate to the EC2 console 
    - Click on **Instances** - **Launch instance** 
    - name should be `awsCatWeb` and select **Amazon Linux** and the free tier **Amazon Linux 2 AMI**
    - set architecture to `64-bit(x86)`
    - choose the free tier eligible instance type
    - choose a keypair or create new keypair
    - click to edit **Network settings and select `awsVPC` as **VPC**
    - select `aws-publicA` for **Subnet** 
    - select **Select existing security group** and select `DMS-awsSecurityGroupWeb-*` 
    - scroll down and expand `Advanced details` 
    - under **IAM Instance profile**, select `DMS-awsInstanceProfile-*`
    - Click `Launch Instance` and then go to *view all instances*
- Install wordpress requirements
    - select the awsCatWeb instance, click on *connect* 
    - on the **SSH client** tab copy the ssh command `ssh -i "your_keypair_name.pem" ec2-user@ec2-52-87-209-171.compute-1.amazonaws.com` 
    - open you terminal in my case, a linux terminal and run the command
    ![ssh to instance](https://github.com/chimezdev/onpremises-to-aws-database-migration/blob/main/Images/sshed_ec2.png)
    - alternatively, you can connect using the `Session Manager` tab - `connect`
    - when you are connected to your linux machine run `sudo bash` for root permission.
    - run `yum -y update` to update the machine
    - run `yum -y install httpd mariadb` to install apache web server and mysql tools.
    - run `amazon-linux-extras install -y lamp-mariadb10.2-php7.2 php7.2` to install php (a prerequisite for wordpress) and make sure apache is running and set to automatically startup should the instance restart.
    ```
    systemctl enable httpd
    systemctl start httpd
    ```
With this, we have a running apache web server ready to connect to our wordpress database.

## STAGE 3C - Migrate wordpress content over
- first we need to edit the SSH config on this machine to allow password authentication temporarily so we can copy the wordpress data across to the `awsCatWeb` machine from the on-premises `CatWeb` Machine
- run `nano /etc/ssh/sshd_config` to open the file with nano
- scroll dow using you device arrow key to the line `PasswordAuthentication no` and change to `PasswordAuthentication yes`
- press `ctrl+o` the `enter` to save and `cltr+x` to exit the editor
- set a password on the `ec2-user` system user by running `passwd ec2-user`
- enter the same **DBPassword**, in production this should be different
- run `service sshd restart` or `systemctl restart sshd` to impliment those changes
- Navigate to the EC2 console and select the `awsCatWeb` instance and note the *private IPv4 address.
- then select the onpremises `CatWeb` instance right-click *connect* connect to the instance
- run `sudo bash`
- change directory `cd /var/www/`. You can run `ls` to see the files in the machine.
- run a secure copy to copy the html folder `scp -rp html ec2-user@private_IP_of_awsCatWeb:/home/ec2-user` and answer `yes`
- copy and enter the `DBPassword` on the parameter tab of the cloudFormation stack
- this will copy the wordpress local files (html folder) from `CatWeb` (on-premises) to `awsCatWeb` (aws)
- back to the `awsCatWeb` server
- do `cd /home/ec2-user`(same dir you are when you sshd) this is where we copied the html folder from onpremises to
- do `ls -la` to view the file
- do `cd html`
- run `cp * -R /var/www/html/` to copy all the content recursive to the webroot of `awsCatWeb`
- Fix Up Permissions and verify **awsCatWeb** works
    - run the following commands to enforce the correct permissions on the files you have just copied 
    ```
    usermod -a -G apache ec2-user   
    chown -R ec2-user:apache /var/www
    chmod 2775 /var/www
    find /var/www -type d -exec chmod 2775 {} \;
    find /var/www -type f -exec chmod 0664 {} \;
    sudo systemctl restart httpd
    ```
- go back to the ec2 console, copy the `public IPv4 DNS` and open in a new tab.
- If it is working, it means we have successfully migrated the web application from on-premise to aws instance which is still loading and pointing to the on-premises database. In the next stage, we will migrate the database 
[webpage](https://github.com/chimezdev/onpremises-to-aws-database-migration/blob/main/Images/webpage_aws.png)

## STAGE 4 - Migrate the database from the CatDB on-premises database to the RDS-MariaDB provissioned in stage 3A above using DMS
1. Create subnet group
    - Search and click **DMS** to navigate to **AWS DMS** console
    - Click on **Subnet groups** on the left menu and then **Create subnet group**
    - provide `A4LDMSSNGROUP` as *Name* and *Description* 
    - select the `awsVPC` from the **VPC** dropdown
    - under `Add subnet` click the dropdown and select `awsPrivateA` and `awsPrivateB` 
    - click on `Create subnet group`
2. Create DMS Replication Instance
    - click on **Replication instances** on the left menu and click on `Create replication instance`
    - enter `A4LONPREMTOAWS` as **Name** and **Description**
    - select `dms.t3.micro` as *instance class*
    - select `Dev or test workload(single-AZ)` under **Multi-AZ**
    - scroll down to **Connectivity and security** ensure `IPv4` is checked
    - select `awsVPC` under **Virtual private cloud(VPC) for IPv4**
    - ensure the `a4ldmssngroup` created earlier is selected under **Replication subnet group** 
    - deselect ***Public accessible***
    - click to expand **Advanced setting** and select `DMS-awsSecurityGroup-*` under **VPC security groups** 
    - click `Create replication instance
3. Creat the DMS Source and Target Endpoints
    - Create Source Endpoint
        - click on **Endpoints** on the left menu and click **Create endpoint**
        - we are creating the source database endpoint so ensure ***Source endpoint*** is checked and ***Select RDS DB instance** is `NOT` selected
        - under **Endpoint identifier** enter `CatDBOnpremises` 
        - **Source engine** should be `MariaDB`
        - select ***Provide access information manually***
        - the **Server name** is the *private IPv4 address* of the on-premises database name `CatDB` in our running instance on the EC2 console.
        - MariaDB port is `3306` so enter this as *Port* 
        - *username is `a4lwordpress` and password is the same *DBPassword value from the output parameter of our cloudFormation stack.
        - finally click on `Create endpoint`
    - Create Target Endpoint
        - click on **Create endpoint
        - select `Target endpoint` 
        - check `Select RDS DB instance` because the **MariaDB** database is RDS-managed
        - most of the values will be automatically populated
        - scroll down and select `Provide access information manually`
        - provide the same **password** only as other fields will be automatically populated
        - scroll down and click on `create endpoint`
    - Test Endpoints
        - go to the RDS console and confirm that the **Replication instance** is **Available** status else wait
        - select any of the endpoints created 
        - click on **Actions** - **Test connections** - **Run Test**
        - if successful the status will change from *testing* to *successful*
        - repeat for the other endpoint
4. Migrate Data
    - Create Database Migration Tasks
        - click on **Database migration tasks** on the left menu - **Create task**
        - **Task identifier** = `A4LONPREMTOAWSWORDPRESS`
        - select the Replication instance created
        - select the *Source* and *Target* database endpoints from the dropdowns
        - choose the **Migration type** in this case we are only migrating existing data
        - scroll down to **Table mappings** and ensure *wizard* editing mode is selected
        - click on `Add new selection rule` 
        - click on `Schema`-`Enter a schema` just refers to the database name and the database table
        - source name we used while creating the database is `a4lwordpress`
        - use `%` as wildcard under *Source table name* to select all tables in the database
        - scroll down and click on `Create task`
        This will go through various *state* from `creating` - `starting` - `Running` and `Load complete` indicating a successful migration
5. Cutover the web application Instance to now Read/Write to the MariaDB database running on AWS
    - Navigate to EC2 console and select the `awsCatWeb`
    - ssh into the instance or connect through the session manager
    - run `sudo bash` to elevate your permission 
    - cd to the directory `/var/www/html` which is where the web contents are
    - if you run `ls` command, you will see the file `wp-config.php`
    - run `nano wp-config.php` to open the file
    - goto RDS console, click to open the running database and copy the **endpoint** 
    - scroll down to `MySQL hostname` change the IP `192.168.10.214` to the replication instance endpoint `a4lwordpress.cb17d2jsxlse.us-east-1.rds.amazonaws.com` 
    - save and exit
    - Run the script below, to update the wordpress database with the new instance DNS name
    ```
    #!/bin/bash
    source <(php -r 'require("/var/www/html/wp-config.php"); echo("DB_NAME=".DB_NAME."; DB_USER=".DB_USER."; DB_PASSWORD=".DB_PASSWORD."; DB_HOST=".DB_HOST); ')
    SQL_COMMAND="mysql -u $DB_USER -h $DB_HOST -p$DB_PASSWORD $DB_NAME -e"
    OLD_URL=$(mysql -u $DB_USER -h $DB_HOST -p$DB_PASSWORD $DB_NAME -e 'select option_value from wp_options where option_id = 1;' | grep http)
    HOST=$(curl http://169.254.169.254/latest/meta-data/public-hostname)
    $SQL_COMMAND "UPDATE wp_options SET option_value = replace(option_value, '$OLD_URL', 'http://$HOST') WHERE option_name = 'home' OR option_name = 'siteurl';"
    $SQL_COMMAND "UPDATE wp_posts SET guid = replace(guid, '$OLD_URL','http://$HOST');"
    $SQL_COMMAND "UPDATE wp_posts SET post_content = replace(post_content, '$OLD_URL', 'http://$HOST');"
    $SQL_COMMAND "UPDATE wp_postmeta SET meta_value = replace(meta_value,'$OLD_URL','http://$HOST');"
    ```
    - press `enter` after pasting it to ensure the entire code runs

Navigate to the EC2 console and stop both intances running on-premises `CatDB` and `CatWeb`
- select each and click `Instance state` - `stop instance`
Test that our web application still runs
- select the `awsCatWeb` instance, copy its `Public IPv4 DNS` and open in a new tab. If this loads, it means our web application now points to the RDS instance.

## STAGE 5 - Cleanup You Account
1. Terminate EC2 Instances
    - select the `awsCatWeb` which you provissioned manually, right click and `Terminate instance`
2. Delete Database
    - navigate to the RDS console
    - select **Database** on the left menu
    - select the `a4lwordpress` click on **Actions** - **Delete**
    - uncheck ***Create final snapshot?*** and ***Retain automated backups*** 
    - acknowledge and then type `delete me` 
    - click on `Delete me`
    - wait for the process to complete before continuing
    - select *Subnet groups* on the left menu
    - select the `a4ldbsngroup` we created, click **Delete** - **Delete**
3. Delete the DMS Service
    - navigate to the DMS console
    - select *Database migration tasks* from the left menu
    - select the `a4lonpremtoawswordpress` task created, click **Actions** - **Delete** - **Delete**
    - click on *Replication instances* , select `a4lonpremtoaws`, click **Actions** - **Delete** - **Delete**
    - click on *Endpoints* on the left menu, select one of the endpoints created, click **Actions** - **Delete** - **Delete**
    - repeat for the other endpoint
    - if these doesn't delete, simply wait for the replication instace to finish deleting
    - click on *Subnet groups* on the left menu, select the subnet `a4ldmssngroup`, click on **Actions** - **Delete** - **Delete**
    - Navigate to the VPC console, select **Route tables** from the left menu
    - select one of the route created say `awsPublicRT`, click on **Routes** tab - **Edit Routes**, click on **Remove** to remove the route you added
    - click on *save changes*
    - repeate for the other two routes
    - click on **Peering connections** on the left menu
    - select the peering connection you created, `A4L-ON-PREMISES-TO-AWS`, click **Actions** - **Delete peering connection** 
    - type *delete* then click on **Delete** to delete
4. Delete CloudFormation Stack
    - Navigate to the CloudFormation console
    - select the `DMS` stack created using the one-click deployment
    - click on **Delete** - **Delete Stack**. This will clean the rest of the resourses provisioned using the one-click deployment.

*** End of Project ***

