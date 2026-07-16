# Project 1 Cloud Migration — Terraform AWS WordPress Deployment

A hands-on AWS Cloud Engineering project demonstrating how to provision cloud infrastructure with Terraform and migrate a traditional WordPress web application onto an Amazon Linux EC2 instance.

This project builds the AWS networking environment from scratch, deploys an EC2 instance, configures a complete Apache, PHP, and MariaDB application stack, and installs WordPress as a working cloud-hosted website.

---

## What This Project Does

- Provisions AWS infrastructure using Terraform
- Creates a custom Virtual Private Cloud
- Creates a public subnet inside the VPC
- Attaches an Internet Gateway to the VPC
- Configures a public route table
- Associates the route table with the public subnet
- Creates an EC2 Security Group
- Allows SSH and HTTP traffic
- Launches an Amazon Linux 2023 EC2 instance
- Retrieves the latest Amazon Linux AMI dynamically
- Uses an existing AWS EC2 key pair
- Installs and configures Apache
- Installs PHP and required extensions
- Installs and configures MariaDB
- Creates a dedicated WordPress database and database user
- Downloads and deploys WordPress
- Connects WordPress to MariaDB
- Verifies the complete application from a web browser

---

## Tech Stack

| Tool | Purpose |
|---|---|
| AWS | Cloud infrastructure platform |
| Terraform | Infrastructure as Code |
| Amazon EC2 | Virtual cloud server |
| Amazon Linux 2023 | Server operating system |
| Amazon VPC | Private cloud network |
| Security Groups | Instance-level firewall |
| Apache HTTP Server | Web server |
| PHP | WordPress application runtime |
| MariaDB | Relational database |
| WordPress | Content management system |
| SSH | Secure remote server access |
| PowerShell | Local command-line environment |
| VS Code | Terraform development environment |
| Git | Source control |
| GitHub | Remote repository |

---

## Architecture

```text
                         Internet
                            |
                            v
                    Internet Gateway
                            |
                            v
                       AWS VPC
                      10.0.0.0/16
                            |
                            v
                    Public Route Table
                 0.0.0.0/0 -> Internet Gateway
                            |
                            v
                 Route Table Association
                            |
                            v
                      Public Subnet
                     10.0.1.0/24
                            |
                            v
                     Security Group
                    SSH 22 | HTTP 80
                            |
                            v
                  Amazon Linux 2023 EC2
                            |
               +------------+------------+
               |            |            |
               v            v            v
            Apache         PHP        MariaDB
               |            |            |
               +-------- WordPress ------+
                            |
                            v
                       Web Browser
```

---

## Application Request Flow

```text
Browser
   |
   | HTTP Request
   v
Apache
   |
   | Passes index.php
   v
PHP
   |
   | Executes WordPress code
   v
MariaDB
   |
   | Returns posts, users, pages, and settings
   v
PHP
   |
   | Builds HTML
   v
Apache
   |
   | Sends HTTP response
   v
Browser
```

---

## Project Structure

```text
project-1-cloud-migration/
├── README.md
├── provider.tf
├── vpc.tf
├── security.tf
├── ec2.tf
├── outputs.tf
├── screenshots/
│   ├── terraform-init.png
│   ├── terraform-plan.png
│   ├── terraform-apply.png
│   ├── aws-vpc.png
│   ├── aws-subnet.png
│   ├── internet-gateway.png
│   ├── route-table.png
│   ├── security-group.png
│   ├── ec2-running.png
│   ├── ssh-connected.png
│   ├── apache-running.png
│   ├── apache-browser.png
│   ├── mariadb-running.png
│   ├── wordpress-database.png
│   ├── wordpress-files.png
│   ├── wp-config.png
│   ├── wordpress-installer.png
│   └── wordpress-success.png
└── notes/
    └── troubleshooting.md
```

---

# Terraform Configuration

## provider.tf

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = "us-east-2"
}
```

The provider configuration tells Terraform to use AWS and deploy resources into the `us-east-2` region.

---

## vpc.tf

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "wordpress-vpc"
  }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-2a"
  map_public_ip_on_launch = true

  tags = {
    Name = "public-subnet"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "wordpress-igw"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "public-route-table"
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}
```

This file creates the networking foundation:

- VPC
- Public subnet
- Internet Gateway
- Route table
- Default internet route
- Route table association

---

## security.tf

```hcl
resource "aws_security_group" "web_sg" {
  name        = "wordpress-security-group"
  description = "Allow SSH and HTTP"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "SSH"

    from_port = 22
    to_port   = 22
    protocol  = "tcp"

    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTP"

    from_port = 80
    to_port   = 80
    protocol  = "tcp"

    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port = 0
    to_port   = 0
    protocol  = "-1"

    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "wordpress-security-group"
  }
}
```

The Security Group permits:

- SSH traffic on port `22`
- HTTP traffic on port `80`
- All outbound traffic

> For a production environment, SSH should be restricted to a trusted administrator IP address instead of `0.0.0.0/0`.

---

## ec2.tf

```hcl
data "aws_ssm_parameter" "amazon_linux" {
  name = "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64"
}

data "aws_key_pair" "sedo_key" {
  key_name = "sedo-key"
}

resource "aws_instance" "wordpress" {
  ami           = data.aws_ssm_parameter.amazon_linux.value
  instance_type = "t3.micro"

  key_name = data.aws_key_pair.sedo_key.key_name

  subnet_id = aws_subnet.public.id

  vpc_security_group_ids = [
    aws_security_group.web_sg.id
  ]

  associate_public_ip_address = true

  tags = {
    Name = "wordpress-server"
  }
}
```

The AMI is retrieved dynamically from AWS Systems Manager Parameter Store instead of using a hardcoded AMI ID.

---

## outputs.tf

```hcl
output "instance_public_ip" {
  description = "Public IP of the WordPress server"
  value       = aws_instance.wordpress.public_ip
}

output "instance_public_dns" {
  description = "Public DNS of the WordPress server"
  value       = aws_instance.wordpress.public_dns
}

output "instance_id" {
  description = "EC2 Instance ID"
  value       = aws_instance.wordpress.id
}
```

These outputs display the information required to connect to and verify the EC2 instance.

---

# Prerequisites

- AWS account
- Terraform installed
- AWS CLI installed
- AWS credentials configured
- Existing EC2 key pair
- Matching private `.pem` file
- PowerShell or Linux terminal
- VS Code
- Git

---

# Configure AWS Credentials

Configure the AWS CLI:

```bash
aws configure
```

Verify the authenticated account:

```bash
aws sts get-caller-identity
```

This confirms which AWS account Terraform will modify.

---

# Initialize Terraform

```bash
terraform init
```

This command:

- Downloads the AWS provider
- Creates the `.terraform` directory
- Creates or updates `.terraform.lock.hcl`
- Prepares the project for Terraform commands

---

# Format the Configuration

```bash
terraform fmt
```

This formats all Terraform files using standard HCL formatting.

---

# Validate the Configuration

```bash
terraform validate
```

Expected output:

```text
Success! The configuration is valid.
```

---

# Preview the Infrastructure

```bash
terraform plan
```

The plan should include resources similar to:

```text
aws_vpc.main
aws_subnet.public
aws_internet_gateway.main
aws_route_table.public
aws_route_table_association.public
aws_security_group.web_sg
aws_instance.wordpress
```

Expected summary:

```text
Plan: 7 to add, 0 to change, 0 to destroy.
```

---

# Deploy the Infrastructure

```bash
terraform apply
```

Enter:

```text
yes
```

Terraform creates the infrastructure and returns:

- EC2 public IP
- EC2 public DNS
- EC2 instance ID

---

# Connect to the EC2 Instance

Use the corresponding private key:

```powershell
ssh -i "C:\Users\<username>\Downloads\sedo-key.pem" ec2-user@<public-ip>
```

Amazon Linux uses:

```text
ec2-user
```

as the default administrative account.

---

# Verify the Server

Check the current user:

```bash
whoami
```

Check the operating system:

```bash
cat /etc/os-release
```

Check the kernel:

```bash
uname -r
```

Check the hostname:

```bash
hostname
```

Check the current directory:

```bash
pwd
```

---

# Update Amazon Linux

```bash
sudo dnf update -y
```

This downloads and installs available package and security updates.

---

# Install Apache

```bash
sudo dnf install httpd -y
```

Enable and start Apache:

```bash
sudo systemctl enable --now httpd
```

Verify:

```bash
sudo systemctl status httpd
```

Test locally:

```bash
curl localhost
```

---

# Verify Apache from a Browser

Open:

```text
http://<ec2-public-ip>
```

A successful response confirms:

- The EC2 instance has a public IP
- Port `80` is allowed
- The route table is working
- The Internet Gateway is attached
- Apache is running

---

# Install PHP

```bash
sudo dnf install php php-mysqlnd php-fpm php-mbstring php-xml -y
```

Verify PHP:

```bash
php -v
```

Restart Apache:

```bash
sudo systemctl restart httpd
```

PHP executes the WordPress application code and produces HTML for Apache to return to the browser.

---

# Install MariaDB

```bash
sudo dnf install mariadb105-server -y
```

Enable and start MariaDB:

```bash
sudo systemctl enable --now mariadb
```

Verify the service:

```bash
sudo systemctl status mariadb
```

Verify the installed version:

```bash
mariadb --version
```

---

# Configure the WordPress Database

Open the MariaDB client:

```bash
sudo mariadb
```

Create the database:

```sql
CREATE DATABASE wordpress;
```

Verify:

```sql
SHOW DATABASES;
```

Create a dedicated WordPress database user:

```sql
CREATE USER 'wpuser'@'localhost'
IDENTIFIED BY 'REPLACE_WITH_A_SECURE_PASSWORD';
```

Grant access to the WordPress database:

```sql
GRANT ALL PRIVILEGES
ON wordpress.*
TO 'wpuser'@'localhost';
```

Reload privileges:

```sql
FLUSH PRIVILEGES;
```

Verify the database user:

```sql
SELECT User, Host FROM mysql.user;
```

Exit MariaDB:

```sql
EXIT;
```

> Never commit the real database password to GitHub.

---

# Download WordPress

Move into the temporary directory:

```bash
cd /tmp
```

Download WordPress:

```bash
wget https://wordpress.org/latest.tar.gz
```

Verify:

```bash
ls -lh
```

Extract the archive:

```bash
tar -xzf latest.tar.gz
```

Verify:

```bash
ls -l
```

The extracted application should be located at:

```text
/tmp/wordpress
```

---

# Deploy WordPress into Apache

Remove the temporary Apache test page:

```bash
sudo rm -f /var/www/html/index.html
```

Copy the WordPress files into Apache's DocumentRoot:

```bash
sudo cp -a /tmp/wordpress/. /var/www/html/
```

Verify:

```bash
ls /var/www/html
```

Expected files include:

```text
index.php
wp-admin
wp-content
wp-includes
wp-config-sample.php
```

---

# Configure Ownership and Permissions

Give Apache ownership of the application:

```bash
sudo chown -R apache:apache /var/www/html
```

Configure directory permissions:

```bash
sudo find /var/www/html -type d -exec chmod 755 {} \;
```

Configure file permissions:

```bash
sudo find /var/www/html -type f -exec chmod 644 {} \;
```

Verify:

```bash
ls -ld /var/www/html
ls -l /var/www/html | head
```

---

# Configure WordPress

Move into the Apache DocumentRoot:

```bash
cd /var/www/html
```

Create the active WordPress configuration:

```bash
sudo cp wp-config-sample.php wp-config.php
```

Edit the file:

```bash
sudo vi wp-config.php
```

Configure the database connection:

```php
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'wpuser' );
define( 'DB_PASSWORD', 'REPLACE_WITH_A_SECURE_PASSWORD' );
define( 'DB_HOST', 'localhost' );
```

`localhost` is used because WordPress and MariaDB are running on the same EC2 instance.

Verify the configuration:

```bash
sudo grep -E "DB_NAME|DB_USER|DB_HOST" /var/www/html/wp-config.php
```

Avoid printing or capturing the password in screenshots or logs.

---

# Restart Apache

```bash
sudo systemctl restart httpd
```

Verify:

```bash
sudo systemctl status httpd
```

---

# Complete the WordPress Installation

Open:

```text
http://<ec2-public-ip>
```

Complete the installation wizard:

1. Select a language
2. Enter the site title
3. Create the WordPress administrator account
4. Enter an email address
5. Install WordPress
6. Log in to the WordPress dashboard

---

# Verification

Check Apache:

```bash
sudo systemctl status httpd
```

Check MariaDB:

```bash
sudo systemctl status mariadb
```

Check PHP:

```bash
php -v
```

Test the web server locally:

```bash
curl localhost
```

Test database authentication:

```bash
mariadb -u wpuser -p wordpress
```

Check the Terraform state:

```bash
terraform state list
```

Display Terraform outputs:

```bash
terraform output
```

---

# Terraform Workflow

```text
Write Configuration
        |
        v
terraform init
        |
        v
terraform fmt
        |
        v
terraform validate
        |
        v
terraform plan
        |
        v
terraform apply
        |
        v
AWS Infrastructure
```

---

# WordPress Deployment Workflow

```text
Amazon Linux
      |
      v
Install Apache
      |
      v
Install PHP
      |
      v
Install MariaDB
      |
      v
Create Database
      |
      v
Create Database User
      |
      v
Download WordPress
      |
      v
Copy WordPress Files
      |
      v
Configure wp-config.php
      |
      v
Restart Apache
      |
      v
Complete Browser Installation
```

---

# Useful Terraform Commands

Initialize:

```bash
terraform init
```

Format:

```bash
terraform fmt
```

Validate:

```bash
terraform validate
```

Preview:

```bash
terraform plan
```

Deploy:

```bash
terraform apply
```

Show outputs:

```bash
terraform output
```

Show managed resources:

```bash
terraform state list
```

Destroy the environment:

```bash
terraform destroy
```

---

# Useful Linux Commands

Check services:

```bash
sudo systemctl status httpd
sudo systemctl status mariadb
```

Restart services:

```bash
sudo systemctl restart httpd
sudo systemctl restart mariadb
```

Enable services:

```bash
sudo systemctl enable httpd
sudo systemctl enable mariadb
```

Check listening ports:

```bash
sudo ss -tulnp
```

Check Apache logs:

```bash
sudo journalctl -u httpd
```

Check MariaDB logs:

```bash
sudo journalctl -u mariadb
```

Test Apache:

```bash
curl localhost
```

---

# Troubleshooting

## No Matching EC2 Key Pair Found

Example:

```text
Error: no matching EC2 Key Pair found
```

Cause:

The AWS key pair existed in a different AWS region.

Resolution:

- Confirm the Terraform provider region
- Confirm the region where the EC2 key pair exists
- Ensure the key pair name matches exactly

List key pairs:

```bash
aws ec2 describe-key-pairs \
  --region us-east-2 \
  --query "KeyPairs[*].KeyName" \
  --output table
```

---

## Invalid Availability Zone

Example:

```text
Value (us-east-1a) for parameter availabilityZone is invalid
```

Cause:

The provider was configured for `us-east-2`, but the subnet used `us-east-1a`.

Resolution:

```hcl
availability_zone = "us-east-2a"
```

The region and Availability Zone must match.

---

## SSH Private Key Not Found

Example:

```text
Warning: Identity file not accessible
```

Cause:

The `.pem` path was incorrect.

Resolution:

Use the full local path:

```powershell
ssh -i "C:\Users\<username>\Downloads\sedo-key.pem" ec2-user@<public-ip>
```

---

## Permission Denied with systemctl

Example:

```text
Failed to enable unit: Access denied
```

Cause:

The command was executed without administrative permissions.

Resolution:

```bash
sudo systemctl enable --now httpd
```

---

## Apache Installed but Inactive

Check:

```bash
sudo systemctl status httpd
```

Start and enable:

```bash
sudo systemctl enable --now httpd
```

---

## Apache Works Locally but Not from Browser

Verify:

- EC2 has a public IP
- Security Group permits TCP port `80`
- Subnet route table contains `0.0.0.0/0`
- Default route targets the Internet Gateway
- Internet Gateway is attached to the VPC
- Apache is running

---

## WordPress Database Connection Error

Example:

```text
Error establishing a database connection
```

Verify:

- MariaDB is running
- Database name is correct
- Database username is correct
- Password matches exactly
- `DB_HOST` is correct
- User permissions were granted

Test manually:

```bash
mariadb -u wpuser -p wordpress
```

---

## Linux Commands Run in PowerShell

Example:

```text
vi is not recognized
sudo is disabled on this machine
```

Cause:

Linux commands were entered in Windows PowerShell instead of inside the EC2 SSH session.

Resolution:

Reconnect to the EC2 instance first:

```powershell
ssh -i "C:\path\to\sedo-key.pem" ec2-user@<public-ip>
```

Confirm the prompt resembles:

```text
[ec2-user@ip-10-0-1-x ~]$
```

Then run Linux commands.

---

## Public IP Address Changed

A standard EC2 public IPv4 address may change after stopping and starting the instance.

Options:

- Run `terraform output`
- Check the EC2 console
- Assign an Elastic IP for a persistent address
- Use Route 53 with a domain name

---

# Security Improvements

For a production environment:

- Restrict SSH to a trusted administrator IP
- Add HTTPS using TLS
- Use an Application Load Balancer
- Store the database in a private subnet
- Replace local MariaDB with Amazon RDS
- Use AWS Secrets Manager for credentials
- Use an IAM role instead of long-lived access keys
- Add CloudWatch monitoring and alarms
- Enable automated backups
- Configure Route 53 DNS
- Use an Elastic IP only when required
- Store Terraform state in S3
- Use DynamoDB or S3 locking capabilities for Terraform state protection
- Add multiple Availability Zones
- Use an Auto Scaling Group
- Regularly patch WordPress, plugins, themes, and the operating system

---

# Key Concepts Demonstrated

- Infrastructure as Code
- Terraform providers
- Terraform resources
- Terraform data sources
- Terraform state
- Terraform dependency graph
- AWS VPC networking
- CIDR addressing
- Public subnets
- Internet Gateways
- Route tables
- Route table associations
- Security Groups
- Amazon EC2
- SSH authentication
- Linux package management
- Linux systemd services
- Apache web hosting
- PHP application execution
- MariaDB administration
- SQL users and permissions
- Principle of Least Privilege
- WordPress deployment
- Application troubleshooting
- Cloud migration fundamentals

---

# Project Outcome

This project successfully demonstrated the complete migration and deployment lifecycle of a traditional WordPress application on AWS.

The environment was:

1. Defined using Terraform
2. Provisioned inside a custom AWS VPC
3. Connected to the internet through an Internet Gateway
4. Protected with a Security Group
5. Hosted on an Amazon Linux EC2 instance
6. Configured with Apache
7. Configured with PHP
8. Configured with MariaDB
9. Deployed with WordPress
10. Verified through a public web browser

This project demonstrates practical experience with AWS infrastructure, Terraform, Linux system administration, web servers, databases, networking, security, and end-to-end cloud application deployment.

---

# Author

**Farrell L. Shelton — Linux Infrastructure Engineer | RHCSA Certified**

GitHub:

https://github.com/folsedo
