

---

# Dynamic eCommerce Web Application on AWS

## Project Overview
This project involves the creation of a dynamic, highly available eCommerce web application hosted on AWS. The infrastructure is designed for online retail stores and leverages AWS services to ensure scalability and reliability.

## Architecture
The application is hosted within an AWS VPC configured with multiple subnets:
- **Public Subnets**: Host the NAT Gateways and provide internet access to resources in private subnets.
- **Private Application Subnets**: Contain the EC2 instances that run the web application, managed by an Auto Scaling group.
- **Private Data Subnets**: Host the RDS database instances for data persistence.

Additional components include:
- **S3 Buckets**: One for storing SQL files and another for web files.
- **Application Load Balancer (ALB)**: Distributes incoming application traffic across multiple EC2 instances.
- **Route 53**: Manages the DNS for the application.
- **Certificate Manager**: Handles SSL/TLS certificates.
- **IAM Roles and Users**: Provide AWS resource access management.

## Technologies Used
- **EC2**: For running the application and web servers.
- **RDS MySQL**: Database storage.
- **S3**: To store web and SQL data files.
- **Route 53, ALB, Auto Scaling**: For traffic distribution and scalability.
- **Flyway**: For database migration.

## Installation & Configuration Steps
1. **AWS Configuration**:
   - Set up a VPC with designated subnets for application and data.
   - Configure security groups and roles for secure access.
   - Set up the EC2 instances within the Auto Scaling group.
   - Deploy the RDS instances in a multi-AZ configuration for high availability.

2. **Server Setup**:
   - Install and configure the necessary software on EC2 instances, including HTTPD, PHP, and MySQL.
   - Utilize `install-configure-app.sh` to automate the setup of the web server environment and application deployment from an S3 bucket.

3. **Database Migration**:
   - Use Flyway for database migrations, pulling SQL files from an S3 bucket to the RDS instance using `migrate-sql-data.sh`.

4. **PHP Configuration**:
   - Modify `AppServiceProvider.php` to enforce HTTPS in production environments.
   - Ensure all environment settings, like database credentials, are correctly configured in `.env` files.

## Challenges and Resolutions
- **MySQL Version Compatibility**: Initially faced issues with MySQL version 5.7.44, which were resolved by upgrading to version 8.0.36 to ensure compatibility with the latest features and security patches.
- **Database Credential Quotations**: Encountered syntax issues in the `.env` file; resolved by correctly placing quotation marks around the database credentials, following the updated syntax requirements in newer PHP versions.

## Final Thoughts
This project not only demonstrates the ability to architect and deploy a scalable, secure eCommerce platform on AWS but also highlights problem-solving skills through debugging and upgrading system components. The detailed documentation and clear step-by-step instructions ensure that the setup is reproducible and maintainable.

## Scripts Included in the Project

### 1. **Database Migration Script (migrate-sql-data.sh)**

This script is used for migrating SQL data from an S3 bucket to an AWS RDS instance using Flyway:

```bash
#!/bin/bash

S3_URI=s3://sf-sql-files/V1__shopwise.sql
RDS_ENDPOINT=dev-rds-db.czsbrkswowxt.us-east-1.rds.amazonaws.com
RDS_DB_NAME=*****        # use your DB Name
RDS_DB_USERNAME=******** # use your DB Username
RDS_DB_PASSWORD=*******  # use your own credentials and never show them

# Update all packages
sudo yum update -y

# Download and extract Flyway
sudo wget -qO- https://download.red-gate.com/maven/release/com/redgate/flyway/flyway-commandline/10.12.0/flyway-commandline-10.12.0-linux-x64.tar.gz | tar -xvz
# Create a symbolic link to make Flyway accessible globally
sudo ln -s $(pwd)/flyway-10.12.0/flyway /usr/local/bin

# Create the SQL directory for migrations
sudo mkdir sql

# Download the migration SQL script from AWS S3
sudo aws s3 cp "$S3_URI" sql/

# Run Flyway migration
flyway -url=jdbc:mysql://"$RDS_ENDPOINT":3306/"$RDS_DB_NAME" \
  -user="$RDS_DB_USERNAME" \
  -password="$RDS_DB_PASSWORD" \
  -locations=filesystem:sql \
  migrate
```

### 2. **PHP Configuration Script (boot-function-code)**

This script snippet from `AppServiceProvider.php` enforces HTTPS in production environments:

```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     */
    public function register(): void
    {
        if (env('APP_ENV') === 'production') {
            \Illuminate\Support\Facades\URL::forceScheme('https');
        }
    }

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        // Add boot logic here
    }
}
```

### 3. **Application Installation and Configuration Script (install-configure-app.sh)**

This script handles the initial setup of the web server and PHP, along with configuring the environment on EC2 instances:

```bash
#!/bin/bash

# This command updates all the packages on the server to their latest versions
sudo yum update -y

# This series of commands installs the Apache web server, enables it to start on boot, and then starts the server immediately
sudo yum install -y httpd
sudo systemctl enable httpd
sudo systemctl start httpd

# This command installs PHP along with several necessary extensions for the application to run
sudo dnf install -y \
    php \
    php-pdo \
    php-openssl \
    php-mbstring \
    php-exif \
    php-fileinfo \
    php-xml \
    php-ctype \
    php-json \
    php-tokenizer \
    php-curl \
    php-cli \
    php-fpm \
    php-mysqlnd \
    php-bcmath \
    php-gd \
    php-cgi \
    php-gettext \
    php-intl \
    php-zip

## These commands Installs MySQL version 8
# Install the MySQL Community repository
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm
#
# Install the MySQL server
sudo dnf install -y mysql80-community-release-el9-1.noarch.rpm
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
dnf repolist enabled | grep "mysql.*-community.*"
sudo dnf install -y mysql-community-server
#
# Start and enable the MySQL server
sudo systemctl start mysqld
sudo systemctl enable mysqld

# This command enables the 'mod_rewrite' module in Apache on an EC2 Linux instance. It allows the use of .htaccess files for URL rewriting and other directives in the '/var/www/html' directory
sudo sed -i '/<Directory "\/var\/www\/html">/,/<\/Directory>/ s/AllowOverride None/AllowOverride All/' /etc/httpd/conf/httpd.conf

# Environment Variable
S3_BUCKET_NAME=******* # Use your own bucket name

# This command downloads the contents of the specified S3 bucket to the '/var/www/html' directory on the EC2 instance
sudo aws s3 sync s3://"$S3_BUCKET_NAME" /var

/www/html

# This command changes the current working directory to '/var/www/html', which is the standard directory for hosting web pages on a Unix-based server
cd /var/www/html

# This command is used to extract the contents of the application code zip file that was previously downloaded from the S3 bucket
sudo unzip shopwise.zip

# This command recursively copies all files, including hidden ones, from the 'shopwise' directory to the '/var/www/html/'
sudo cp -R shopwise/. /var/www/html/

# This command permanently deletes the 'shopwise' directory and the 'shopwise.zip' file.
sudo rm -rf shopwise shopwise.zip

# This command set permissions 777 for the '/var/www/html' directory and the 'storage/' directory
sudo chmod -R 777 /var/www/html
sudo chmod -R 777 storage/

# This command will open th vi editor and allow you to edit the .env file to add your database credentials
sudo vi .env

# This command will restart the Apache server
sudo service httpd restart
```
---
