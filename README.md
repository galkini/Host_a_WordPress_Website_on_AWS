![Alt text](/WordPress_web_site_on_AWS.png)

# WordPress Website Hosting on AWS

This project demonstrates how to host a WordPress website on AWS using various AWS services to ensure reliability, fault tolerance, and security. The project includes a reference architecture diagram and deployment scripts available in the [GitHub repository](https://github.com/galkini/Host_a_WordPress_Website_on_AWS).

## Architecture Overview

1. **VPC Configuration**: A Virtual Private Cloud (VPC) with both public and private subnets across two different availability zones was configured to provide isolation and segmentation.
2. **Internet Gateway**: Deployed an Internet Gateway to facilitate connectivity between VPC instances and the Internet.
3. **Security Groups**: Established Security Groups to act as network firewalls.
4. **Availability Zones**: Utilized two Availability Zones to enhance system reliability and fault tolerance.
5. **Public Subnets**: Leveraged Public Subnets for components such as the NAT Gateway and Application Load Balancer.
6. **EC2 Instance Connect Endpoint**: Implemented for secure connections to assets within both public and private subnets.
7. **Private Subnets**: Positioned web servers (EC2 instances) within Private Subnets for enhanced security.
8. **NAT Gateway**: Configured to allow instances in the private Application and Data subnets to access the Internet.
9. **EC2 Instances**: Hosted the WordPress website on these instances.
10. **Application Load Balancer**: Employed to distribute web traffic evenly to an Auto Scaling Group across multiple Availability Zones.
11. **Auto Scaling Group**: Automatically managed EC2 instances to ensure availability, scalability, and elasticity.
12. **GitHub**: Used for version control and collaboration by storing web files.
13. **Certificate Manager**: Ensured secure application communications.
14. **SNS**: Configured Simple Notification Service to alert about activities within the Auto Scaling Group.
15. **Route 53**: Registered the domain name and set up a DNS record.
16. **Amazon EFS**: Used for shared file storage across multiple EC2 instances.
17. **Amazon RDS**: Used for database management.

## Setup Instructions

The following steps outline how to deploy the WordPress website on an EC2 instance using a bash script.

### Prerequisites

- An AWS account with necessary permissions to create VPC, subnet, EC2, security groups, etc.
- A GitHub repository containing the WordPress website files.
- A registered domain name (optional but recommended).

### Steps to Deploy

#### EC2 Instance Setup

1. **Launch an EC2 Instance**:
    - Use an AMI that supports yum (e.g., Amazon Linux 2 AMI).
    - Ensure the instance is in the public subnet and has a security group that allows HTTP traffic (port 80).

2. **Connect to the EC2 Instance**:
    - Use EC2 Instance Connect or your preferred method (SSH).

3. **Run the Initialization Script**:

    ```bash
    # Switch to the root user
    sudo su
    
    # Update the software packages on the EC2 instance
    sudo yum update -y
    
    # Create an html directory
    sudo mkdir -p /var/www/html
    
    # Environment variable for EFS DNS name
    EFS_DNS_NAME=fs-064e9505819af10a4.efs.us-east-1.amazonaws.com
    
    # Mount the EFS to the html directory
    sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport "$EFS_DNS_NAME":/ /var/www/html
    
    # Install the Apache web server, enable it to start on boot, and then start the server immediately
    sudo yum install -y httpd
    sudo systemctl enable httpd
    sudo systemctl start httpd
    
    # Install PHP 8 along with several necessary extensions for WordPress to run
    sudo dnf install -y \
    php \
    php-cli \
    php-cgi \
    php-curl \
    php-mbstring \
    php-gd \
    php-mysqlnd \
    php-gettext \
    php-json \
    php-xml \
    php-fpm \
    php-intl \
    php-zip \
    php-bcmath \
    php-ctype \
    php-fileinfo \
    php-openssl \
    php-pdo \
    php-tokenizer
    
    # Install the MySQL version 8 community repository
    sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm
    
    # Install the MySQL server
    sudo dnf install -y mysql80-community-release-el9-1.noarch.rpm
    sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
    sudo dnf repolist enabled | grep "mysql.*-community.*"
    sudo dnf install -y mysql-community-server
    
    # Start and enable the MySQL server
    sudo systemctl start mysqld
    sudo systemctl enable mysqld
    
    # Set permissions
    sudo usermod -a -G apache ec2-user
    sudo chown -R ec2-user:apache /var/www
    sudo chmod 2775 /var/www && find /var/www -type d -exec sudo chmod 2775 {} \;
    sudo find /var/www -type f -exec sudo chmod 0664 {} \;
    sudo chown apache:apache -R /var/www/html
    
    # Download WordPress files
    wget https://wordpress.org/latest.tar.gz
    tar -xzf latest.tar.gz
    sudo cp -r wordpress/* /var/www/html/
    
    # Create the wp-config.php file
    sudo cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
    
    # Edit the wp-config.php file
    sudo vi /var/www/html/wp-config.php
    
    # Restart the web server
    sudo service httpd restart
    ```

#### Auto Scaling Group Launch Template

1. **Launch Template Script**:

    ```bash
    #!/bin/bash
    # Update the software packages on the EC2 instance
    sudo yum update -y
    
    # Install the Apache web server, enable it to start on boot, and then start the server immediately
    sudo yum install -y httpd
    sudo systemctl enable httpd
    sudo systemctl start httpd
    
    # Install PHP 8 along with several necessary extensions for WordPress to run
    sudo dnf install -y \
    php \
    php-cli \
    php-cgi \
    php-curl \
    php-mbstring \
    php-gd \
    php-mysqlnd \
    php-gettext \
    php-json \
    php-xml \
    php-fpm \
    php-intl \
    php-zip \
    php-bcmath \
    php-ctype \
    php-fileinfo \
    php-openssl \
    php-pdo \
    php-tokenizer
    
    # Install the MySQL version 8 community repository
    sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm
    
    # Install the MySQL server
    sudo dnf install -y mysql80-community-release-el9-1.noarch.rpm
    sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
    sudo dnf repolist enabled | grep "mysql.*-community.*"
    sudo dnf install -y mysql-community-server
    
    # Start and enable the MySQL server
    sudo systemctl start mysqld
    sudo systemctl enable mysqld
    
    # Environment variable for EFS DNS name
    EFS_DNS_NAME=fs-02d3268559aa2a318.efs.us-east-1.amazonaws.com
    
    # Mount the EFS to the html directory
    echo "$EFS_DNS_NAME:/ /var/www/html nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 0 0" >> /etc/fstab
    mount -a
    
    # Set permissions
    sudo chown apache:apache -R /var/www/html
    
    # Restart the web server
    sudo service httpd restart
    ```

## Repository

Find all configuration files, diagrams, and scripts in the [GitHub repository](https://github.com/galkini/Host_a_WordPress_Website_on_AWS).

## Additional Configuration

1. **Domain Name and SSL**:
    - Use AWS Certificate Manager to create and manage certificates for SSL/TLS.
    - Integrate the certificate with the Application Load Balancer.
    - Configure Route 53 to route the domain name to the Load Balancer.

2. **Notifications and Monitoring**:
    - Set up Amazon SNS for notifications about auto-scaling activities.
    - Use Amazon CloudWatch for monitoring instance health and performance metrics.

## Conclusion

By following the steps outlined above, you can successfully host a WordPress website on AWS with a robust and scalable architecture. This setup leverages various AWS services to ensure high availability, security, and efficient management of resources.
