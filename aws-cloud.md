# AWS Cloud Solution For 2 Company Websites Using Nginx as a Reverse Proxy

## Pretasks:

- Configure AWS account and Organization Unit
  - Create an AWS account (ignore if you already have one)
  - Create an Organization Unit
  - Click 'Add an AWS account' and create a new AWS account from there with name as Dev
  - Invite another AWS account( You can open a totally new AWS account for this)
  - Login and accept the invitation in the new AWS account
    ![1](https://user-images.githubusercontent.com/47898882/132263388-5cda326b-452e-44e4-9c3f-dfc651ec410e.JPG)
- Create a free domain name from http://www.freenom.com
- Create a hosted zone in AWS Route 53
  - Go to the Route 53 Console
  - Click 'Create Hosted Zone'
  - For Domain name, enter the domain name you got from freenom
  - Enter a description (if you want)
  - For type, select Public Hosted Zone
  - Click Create Hosted Zone
  - Click on the created hosted zone and copy the contents of the NS record
  - Click 'Manage Domain' next to your domain name, and click Management Tools and select Nameservers
  - Click 'Use custom nameservers' radio button
  - Replace the content there with the items you got from Route 53 (one per line)
    ![2](https://user-images.githubusercontent.com/47898882/132263392-04fcf914-acf0-425b-8d2d-5647c8e07f60.JPG)
    ![3](https://user-images.githubusercontent.com/47898882/132263394-606cdaa2-b39a-438b-adb2-5971eb340d13.JPG)
- Ensure to tag all resources you create (Project, Environment, Name etc)

## Step 1: TLS Certificates from Amazon Certificate Manager (ACM)

- Navigate to AWS ACM
- Under 'Provision certificates' click Get started
- Click Request a certificate
- Enter the domain name you registered (<_domain-name>.com_).Also enter additional domain names:
  - tooling.<_domain-name>.com_
  - www.<_domain-name>.com_
- Click Next
- Select DNS validation and click Next
- Tag the certificate, click Review then confirm and request
- Click Continue
- Click 'Export DNS Configuration file'
- Go to Route 53
- Create a new CNAME record with items from the DNS configuration.csv file downloaded.

## Step 2: Setup a Virtual Private Cloud

![tooling_project_15](https://user-images.githubusercontent.com/76074379/123254593-b4064680-d4a3-11eb-8099-329e9fb7c060.png)

- Create a VPC from the VPC Management Console use a large enough CIDR block (/16)
  ![4](https://user-images.githubusercontent.com/47898882/132263573-df6447e2-225c-4460-8c0f-e554a99cd047.JPG)

- Create subnets as shown in the diagram above

  ![5](https://user-images.githubusercontent.com/47898882/132263678-43075988-ce72-4f52-837d-68d12214416e.JPG)

  - For the public subnet, enable auto-assign IP by selecting the subnet (after you've created it) and clicking Actions button on the top right, then select Modify auto-assign IP settings and enable it
    ![6](https://user-images.githubusercontent.com/47898882/132263725-42d4b33b-90a6-497a-aed2-0c5a24bdab75.JPG)

- Create a route table and associate it with the public subnets
  - Select the route table you created, click Actions on the top and click 'Edit Subnet associations'
  - Select the public subnets and click save
    ![7](https://user-images.githubusercontent.com/47898882/132406134-b94429e1-dae0-443a-a7ef-9a59bbcf2be3.JPG)
- Create a route table for the private subnets
  - Repeat the steps above
    ![8](https://user-images.githubusercontent.com/47898882/132406139-57a63672-a49c-4d4e-bc4b-c29074853382.JPG)
- Create an Internet Gateway, select it and click Actions the click Attach to VPC and attach it to the VPC you created
  ![9](https://user-images.githubusercontent.com/47898882/132406140-90dffb16-b024-4550-8988-23fa783f225c.JPG)

- Add a new route to your public subnet route table
  - Select the route table, click Actions and 'Edit routes'
  - For destination, enter 0.0.0.0/0
  - For target, select Internet Gateway and click the Internet Gateway you created
  - Click Save
    ![10](https://user-images.githubusercontent.com/47898882/132406923-f919f4ae-f07d-4c7a-8fcf-420341828d53.JPG)
- Create a NAT Gateway for your private subnets
- Allocate three Elastic IPs and associate one of them to the NAT Gateway(the other two are for the Bastion Servers)
- Add a new route to your private route table with destination as 0.0.0.0/0 and target as the NAT Gateway you created
  ![11](https://user-images.githubusercontent.com/47898882/132406931-1cad9ba4-7719-466e-9357-799e13c59141.JPG)

- Create a security group for:
  - Nginx servers: Access to nginx servers should only be from the Application Load Balancer
  - Bastion servers: Access to bastion servers should only be from the IPs of your workstation
  - Application Load Balancer: ALB should be open to the internet
  - Webservers: Webservers should only be accessible from the Nginx servers
  - Data Layer: This comprises the RDS and EFS servers. Access to RDS should only be from Webservers, while Nginx and Webservers can have access to EFS
    ![12](https://user-images.githubusercontent.com/47898882/132406934-14310f3d-d304-46e7-aa6a-00372c489f1a.JPG)

## Step 3: Setup EFS

- Navigate to EFS from your Management Console
- Click create file system from the right
- Click Customize
- Enter the name for the EFS
- Tag the resource
- Leave everything else and click next
- Select the VPC you created, select the two AZs and choose the private subnets
- Select the EFS security group for each AZ
- Click next, next then create
  ![13](https://user-images.githubusercontent.com/47898882/132412797-a9291463-1188-45b9-845d-8525e9f88f58.JPG)

- Create an EFS access point. (Give it a name and leave all other settings as default)
  ![14](https://user-images.githubusercontent.com/47898882/132412805-42dc90b3-0d13-48fa-91a8-357396079bf0.JPG)

## Step 4: Setup RDS

### Step 4.1: Create a KMS key

- Navigate to AWS KMS
- Click create key
- Make sure it's symmetric
- Give the key an alias
- For 'Define Key admininstrative privileges', select AWSServiceRoleForRDS and rdsMonitoringRole

![{AB88012C-7F94-4616-89DE-BDB4C40B2F74} png](https://user-images.githubusercontent.com/76074379/124367186-e6cdde80-dc09-11eb-9ed9-c6e4619c4899.jpg)

- Select the same thing for Key usage
- Click Finish

### Step 4.2: Create a DB Subnet Group

- Navigate to RDS Management Console
- Click the three horizontal lines on the top left
- Select Subnet groups
- Click Create DB subnet group
- Enter the name, description and select your VPC
- Under Add subnets, select the two AZs your data layer subnets are in and select the two private data layer subnets.
- Click Create

![15](https://user-images.githubusercontent.com/47898882/132412806-e096c986-792e-4f66-8708-a3b1d18ee1ca.JPG)

### Step 4.3: Create RDS Instance

- Navigate to RDS Management Console
- Click Create database
- For Engine options, select MySQL
- For Template, choose Dev/Test
- Enter a name for your DB under DB instance identifier
- Enter Master username and passsword
- Choose the smallest possible instance class (to reduce costs)
- Under Availability, select do not create a standby instance
- Select your VPC, select the subnet group you created and also the data layer security group
- Leave everything else and scroll down to Additional configuration
- Enter initial database name (but i'll personally recommend you connect to it from your webservers and create required databases)
- Leave everything else, scroll down to Encryption and select the KMS key you created
- Scroll down and click Create database

![{1D2C2675-1F4A-4E2C-8FF9-ED89510F0C23} png](https://user-images.githubusercontent.com/76074379/124367257-72476f80-dc0a-11eb-9ce7-c0625f28019a.jpg)


## Step 5: Proceed with Compute Resources

### Step 5.1: Setup Compute Resources for Nginx

- Provision EC2 Instances for Nginx

  - Create a t2.micro RHEL 8 instance in any of your two public AZs
  - Login as root user & install the following packages

    ```
    yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

    yum install -y dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm

    yum install wget vim python3 telnet htop git mysql net-tools chrony -y

    systemctl start chronyd

    systemctl enable chronyd

    ```

  - Configure SeLinux Policies for Nginx

    ```
    setsebool -P httpd_can_network_connect=1
    setsebool -P httpd_can_network_connect_db=1
    setsebool -P httpd_execmem=1
    setsebool -P httpd_use_nfs 1

    ```

  - Set up a self-signed certificate for the nginx instance, so traffic can be forwarded to the webservers in the private subnet using https(443) protocol

    ```
    sudo mkdir /etc/ssl/private

    sudo chmod 700 /etc/ssl/private

    openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/ACS.key -out /etc/ssl/certs/ACS.crt

    sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
    ```

  - Install amazon efs utils for mounting target on the elastic file system.

    ```
    git clone https://github.com/aws/efs-utils

    cd efs-utils

    yum install -y make

    yum install -y rpm-build

    make rpm

    yum install -y  ./build/amazon-efs-utils*rpm
    ```

- Create an AMI from the instance
  - Right click on the instance
  - Select Image and click Create Image
  - Give the AMI a name
- Prepare Launch Template for Nginx
  - From EC2 Console, click Launch Templates from the left pane
  - Choose the Nginx AMI
  - Select the instance type (t2.micro)
  - Select the key pair
  - Select the security group
  - Add resource tags
  - Click Advanced details, scroll down to the end and configure the user data script to install nginx, clone the repo where we have, so we can use the updated configuration for our proxy server. Move the _reverse.conf_ file into nginx and write it into the _nginx.conf_ file
    ```
    yum install -y nginx
    systemctl start nginx
    systemctl enable nginx
    git clone https://github.com/brpo01/ACS-project-config.git
    mv /ACS-project-config/reverse.conf /etc/nginx/
    mv /etc/nginx/nginx.conf /etc/nginx/nginx.conf-distro
    cd /etc/nginx/
    touch nginx.conf
    sed -n 'w nginx.conf' reverse.conf
    systemctl restart nginx
    rm -rf reverse.conf
    rm -rf /ACS-project-config
    ```
- Configure Target Group (for both Port 80)

  - Select instances as target type
  - Enter the target group name
  - Select the VPC you created
  - For health checks, select HTTP(port 80) and health check path as /healthstatus
  - Add Tags
  - Register Nginx instances as targets

- Configure Application Load Balancer (ALB) for nginx. Nginx instances should only accept connections coming from the external ALB and deny any connections directly to it.

- Create an internet facing ALB

  - From the EC2 Console, click Load Balancers.
  - On the block for Application Load Balancers, click create
  - Enter the name for the load balancer
  - Since it's for the Nginx servers, add a HTTPS Listener
  - Select the VPC you created, check the two AZs and add the public subnets you have. Click next.
  - Select the certificate you created on ACM
  - On the next page, select the ALB security group
  - Configure routing, select the Nginx target group for port 443
  - Register Nginx instances as targets (if you were not able to configure target group for HTTPS(port 443) in the above step)
  - for health checks, select HTTPS(port 443) and health check path as /healthstatus
  - Click Review and complete the process
  - You may add HTTP Listener and the corresponding Target Group

- Configure Autoscaling for Nginx

  - Enter the name
  - Select the Nginx launch template, click Next
  - For Load Balancer, Click Existing Load Balancer and add the right Target Groups( for both port 80 and 443)
  - Select the VPC and select the two public subnets you created, click Next
  - For health checks, select ELB too. Click Next.
  - For Group size, enter 2 for minimum and desired capacity, 4 as maximum capacity
  - For Scaling policies, select Target Tracking scaling policy and set the cpu utilization target value as 90
  - Click Next and add Notifications, create a new SNS topic and enter your email under 'With these recipients'
  - Add Tags

### Step 5.2: Setup Compute Resources for Bastion

- Provision EC2 Instances for Bastion server

  - Create a t2.micro RHEL 8 instance in any of your two public AZs where you created Nginx instances
  - Install the following packages
    ```
    yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm 
    
    yum install -y dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm 
    
    yum install wget vim python3 telnet htop git mysql net-tools chrony -y 
    
    systemctl start chronyd 
    
    systemctl enable chronyd
    ```
  - Attach an Elastic IP to each of the servers
  - Create an AMI from the instance
    - Right click on the instance
    - Select Image and click Create Image
    - Give the AMI a name

- Prepare Launch Template for Nginx
  - From EC2 Console, click Launch Templates from the left pane
  - Choose the Bastion AMI
  - Select the instance type (t2.micro)
  - Select the key pair
  - Select the security group
  - Add resource tags
  - Click Advanced details, scroll down to the end and configure the user data script to update the yum repo and install mysql, git & ansible.
    ```
    #!/bin/bash
    yum install -y mysql
    yum install -y git tmux
    yum install -y ansible
    ```
- Configure Target Groups

  - Select instances as target type
  - Enter the target group name
  - Select the VPC you created
  - For health checks, select HTTPS and health check path as /healthstatus
  - Add Tags
  - Register Bastion instances as targets

- Before we create autoscaling for the wordpress & tooling server, let us create databases in the RDS instance. To do this, log into the RDS instance using the bastion host and create the databases. There'll be a prompt for a password, input the password you used in creating the rds instance.

```
mysql -h acs-database.cdqpbjkethv0.us-east-1.rds.amazonaws.com -u ACSadmin -p
CREATE DATABASE toolingdb; 
CREATE DATABASE wordpressdb;
```

- Configure Autoscaling for Nginx

  - Enter the name
  - Select the Bastion launch template, click Next
  - Select the VPC and select the two public subnets you created, click Next
  - For health checks, select ELB too. Click Next.
  - For Group size, enter 2 for minimum and desired capacity, 4 as maximum capacity
  - For Scaling policies, select Target Tracking scaling policy and set the cpu utilization target value as 90
  - Click Next and add Notifications, create a new SNS topic and enter your email under 'With these recipients'
  - Add Tags

### Step 5.3: Setup Compute Resources for Webservers (Wordpress & Tooling)

We have to create two launch templates for Wordpress and Tooling respectively.

- Provision two EC2 Instances one for Tooling and another for Wordpress

  - Create a t2.micro RHEL 8 instance in any of your two public AZs where you created Nginx instances
  - Install the following packages
    ```
    yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

    yum install -y dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm

    yum install wget vim python3 telnet htop git mysql net-tools chrony -y

    systemctl start chronyd

    systemctl enable chronyd
    ```
  - Configure SeLinux Policies for Nginx
    ```
    setsebool -P httpd_can_network_connect=1
    setsebool -P httpd_can_network_connect_db=1
    setsebool -P httpd_execmem=1
    setsebool -P httpd_use_nfs 1
    ```

  - Set up a self-signed certificate for the nginx instance, so traffic can be forwarded to the webservers in the private subnet using https(443) protocol

    ```
    yum install -y mod_ssl

    openssl req -newkey rsa:2048 -nodes -keyout /etc/pki/tls/private/ACS.key -x509 -days 365 -out /etc/pki/tls/certs/ACS.crt

    vi /etc/httpd/conf.d/ssl.conf
    ```
  - Create an AMI from the instance
    - Right click on the instance
    - Select Image and click Create Image
    - Give the AMI a name

- Prepare Launch Template for Webservers
  - From EC2 Console, click Launch Templates from the left pane
  - Choose the Wordpress AMI
  - Select the instance type (t2.micro)
  - Select the key pair
  - Select the security group
  - Add resource tags
  - Click Advanced details, scroll down to the end and configure the user data script and other necessary programs, modules and dependencies.
    ```
    #!/bin/bash
    mkdir /var/www/
    sudo mount -t efs -o tls,accesspoint=fsap-0f9364679383ffbc0 fs-8b501d3f:/ /var/www/
    yum install -y httpd 
    systemctl start httpd
    systemctl enable httpd
    yum module reset php -y
    yum module enable php:remi-7.4 -y
    yum install -y php php-common php-mbstring php-opcache php-intl php-xml php-gd php-curl php-mysqlnd php-fpm php-json
    systemctl start php-fpm
    systemctl enable php-fpm
    wget http://wordpress.org/latest.tar.gz
    tar xzvf latest.tar.gz
    rm -rf latest.tar.gz
    cp wordpress/wp-config-sample.php wordpress/wp-config.php
    mkdir /var/www/html/
    cp -R /wordpress/* /var/www/html/
    cd /var/www/html/
    touch healthstatus
    sed -i "s/localhost/acs-database.cdqpbjkethv0.us-east-1.rds.amazonaws.com/g" wp-config.php 
    sed -i "s/username_here/ACSadmin/g" wp-config.php 
    sed -i "s/password_here/admin12345/g" wp-config.php 
    sed -i "s/database_name_here/wordpressdb/g" wp-config.php 
    chcon -t httpd_sys_rw_content_t /var/www/html/ -R
    systemctl restart httpd
    ```
- Configure Target Groups

  - Select instances as target type
  - Enter the target group name
  - Select the VPC you created
  - For health checks, select HTTP and health check path as /healthstatus
  - Add Tags
  - Register Tooling instance as target

- Create an internal ALB

  - From the EC2 Console, click Load Balancers.
  - On the block for Application Load Balancers, click create
  - Enter the name for the load balancer
  - Select the VPC you created, check the two AZs and add the private subnets you have. Click next.
  - On the next page, select the webserver security group
  - Configure routing, select the Tooling target group
  - Register targets (unnecessary if you configured your target group correctly)
  - Click Review and complete the process

- Configure Autoscaling for Webservers(Tooling and Wordpress)

  - Enter the name
  - Select the appropriate launch template, click Next
  - Select the VPC and select the two public subnets you created, click Next
  - For health checks, select ELB too. Click Next.
  - For Group size, enter 2 for minimum and desired capacity, 4 as maximum capacity
  - For Scaling policies, select Target Tracking scaling policy and set the target value as 90
  - Click Next and add Notifications, create a new SNS topic and enter your email under 'With these recipients'
  - Add Tags

- Repeat the above process for the tooling server and use the script below as the user data for launching our templates.

```
#!/bin/bash
mkdir /var/www/
sudo mount -t efs -o tls,accesspoint=fsap-01c13a4019ca59dbe fs-8b501d3f:/ /var/www/
yum install -y httpd 
systemctl start httpd
systemctl enable httpd
yum module reset php -y
yum module enable php:remi-7.4 -y
yum install -y php php-common php-mbstring php-opcache php-intl php-xml php-gd php-curl php-mysqlnd php-fpm php-json
systemctl start php-fpm
systemctl enable php-fpm
git clone https://github.com/Livingstone95/tooling-1.git
mkdir /var/www/html
cp -R /tooling-1/html/*  /var/www/html/
cd /tooling-1
mysql -h acs-database.cdqpbjkethv0.us-east-1.rds.amazonaws.com -u ACSadmin -p toolingdb < tooling-db.sql
cd /var/www/html/
touch healthstatus
sed -i "s/$db = mysqli_connect('mysql.tooling.svc.cluster.local', 'admin', 'admin', 'tooling');/$db = mysqli_connect('acs-database.cdqpbjkethv0.us-east-1.rds.amazonaws.com ', 'ACSadmin', 'admin12345', 'toolingdb');/g" functions.php
chcon -t httpd_sys_rw_content_t /var/www/html/ -R
systemctl restart httpd
```
  
## Step 6: Configure DNS with Route 53

- Create a CNAME record that points www.domain-name.com to the DNS name of your internal load balancer
- Create a CNAME record that points tooling.domain-name.com to the DNS name of your internal load balancer

![16](https://user-images.githubusercontent.com/47898882/132913054-1061b27c-1d27-482a-9e42-bb39fbb038bb.JPG)

- Go to your browser and test the setup using the domian name you used to route traffic from the internal loadbalancer to the webservers tooling & wordpress) in the route53 records.

![{9DA7708A-4274-402A-A491-E96AFEDE7E74} png](https://user-images.githubusercontent.com/76074379/124367321-09142c00-dc0b-11eb-8c41-9d7122a459a2.jpg)

![{DC558EFC-2080-480A-AD2D-F60E0DDAA084} png](https://user-images.githubusercontent.com/76074379/124367332-1d582900-dc0b-11eb-8d9a-0cdad5d4491a.jpg)
