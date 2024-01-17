![]()
# AWS WordPress Deployment Project

## Overview

This project deploys a WordPress site on AWS, utilizing various resources for high availability, fault tolerance, and scalability. The deployment includes a VPC with public and private subnets across two availability zones, an Internet Gateway, MySQL RDS database, Amazon EFS for shared files, EC2 instances hosting the website, an Application Load Balancer, and Route 53 for domain registration.

## Architecture

1. **VPC Configuration:**
   - VPC with public and private subnets in two availability zones.
   - Internet Gateway for communication between instances in the VPC and the Internet.

2. **Availability Zones:**
   - Two availability zones for high availability and fault tolerance.
   - Public subnets for resources like Nat Gateway, Bastion Host, and Application Load Balancer.

3. **Nat Gateway:**
   - Allows instances in private subnets to access the Internet.

4. **MySQL RDS Database:**
   - Hosts the WordPress database.

5. **Amazon EFS:**
   - Enables webservers to access shared files.
   - EFS Mount Targets in each AZ in the VPC.

6. **EC2 Instances:**
   - Hosts the WordPress website.
   - Utilizes Auto Scaling Group for high availability.
   - EFS mounted on `/var/www/html` for shared files.

7. **Application Load Balancer:**
   - Distributes web traffic across the Auto Scaling Group of EC2 instances in multiple AZs.

8. **Route 53:**
   - Used to register the domain name and create a record set for DNS resolution.

## Deployment Scripts

### Install WordPress Script

```bash
# Create the html directory and mount the EFS to it
sudo su
yum update -y
mkdir -p /var/www/html
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport <efs-id>:/ /var/www/html

# Install Apache
sudo yum install -y httpd httpd-tools mod_ssl
sudo systemctl enable httpd 
sudo systemctl start httpd

# Install PHP 7.4
sudo amazon-linux-extras enable php7.4
sudo yum clean metadata
sudo yum install php php-common php-pear -y
sudo yum install php-{cgi,curl,mbstring,gd,mysqlnd,gettext,json,xml,fpm,intl,zip} -y

# Install MySQL 5.7
sudo rpm -Uvh https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
sudo yum install mysql-community-server -y
sudo systemctl enable mysqld
sudo systemctl start mysqld

# Set permissions
sudo usermod -a -G apache ec2-user
sudo chown -R ec2-user:apache /var/www
sudo chmod 2775 /var/www && find /var/www -type d -exec sudo chmod 2775 {} \;
sudo find /var/www -type f -exec sudo chmod 0664 {} \;
chown apache:apache -R /var/www/html 

# Download WordPress files
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
cp -r wordpress/* /var/www/html/

# Create the wp-config.php file
cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php

# Edit the wp-config.php file
nano /var/www/html/wp-config.php

# Restart the webserver
service httpd restart

```

### Create an Application Load Balancer Script

```bash
#!/bin/bash

# Update yum package manager
yum update -y

# Install Apache, mod_ssl, and tools
sudo yum install -y httpd httpd-tools mod_ssl

# Enable and start Apache service
sudo systemctl enable httpd
sudo systemctl start httpd

# Enable PHP 7.4 using Amazon Linux Extras
sudo amazon-linux-extras enable php7.4

# Clean yum metadata
sudo yum clean metadata

# Install PHP and its extensions
sudo yum install php php-common php-pear -y
sudo yum install php-{cgi,curl,mbstring,gd,mysqlnd,gettext,json,xml,fpm,intl,zip} -y

# Install MySQL repository and server
sudo rpm -Uvh https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
sudo yum install mysql-community-server -y

# Enable and start MySQL service
sudo systemctl enable mysqld
sudo systemctl start mysqld

# Mount EFS (Elastic File System)
echo "<efs-id>:/ /var/www/html nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 0 0" >> /etc/fstab
mount -a

# Set ownership of web directory
chown apache:apache -R /var/www/html

# Restart Apache service
sudo service httpd restart
```

### Commands to SSH into an EC2 Instance in the Private Subnet (Mac)

```bash
# Command 1:
ssh-add --apple-use-keychain <the-name-of-your-private-key.pem>

# Command 2:
ssh -A ec2-user@<the-public-ipv4-ip-of-your-bastion-host>

# Command 3:
ssh ec2-user@<the-private-ipv4-ip-of-the-instance-in-the-private-subnet>
```

### Create an HTTPS Listener for the Application Load Balancer

```php
/* SSL Settings */
define('FORCE_SSL_ADMIN', true);

// Get true SSL status from AWS load balancer
if(isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
  $_SERVER['HTTPS'] = '1';
}
```

This README provides an overview of the architecture and includes key deployment scripts and commands for various components in your WordPress deployment project on AWS.
