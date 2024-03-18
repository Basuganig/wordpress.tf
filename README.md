# wordpress.tf
Introduction
Learn how to deploy WordPress on a two-tier AWS architecture using Terraform. This guide will assist you in creating a robust and scalable infrastructure for hosting your website. By utilizing AWS services and Terraform automation, you can optimize performance and prepare for future growth.

What is Terraform?
Terraform is an open-source tool developed by HashiCorp that automates IT infrastructure management. It enables you to define, modify, and delete cloud resources through declarative configuration files. By using Terraform, you can manage infrastructure as code, which simplifies operations and supports a variety of cloud providers.

Advantages of Terraform:

Infrastructure Automation: Effortlessly provision and manage cloud resources using configuration files.
Multi-Cloud and Multi-Provider: Offers support for various cloud platforms, promoting flexibility and portability.
Modularity: Develop reusable infrastructure modules for scalable and efficient architectures.
Dependency Management: Addresses resource dependencies to guarantee consistent updates and manage interconnected infrastructures.
If you’re new to Terraform, we offer an introductory video tutorial and a demo on YouTube for your convenience.

No alt text provided for this image
What exactly is a 2-Tier Architecture?
A two-tier architecture, also known as a two-tier model, is an IT infrastructure concept that separates application components into two distinct layers: the presentation layer and the data layer.


The presentation layer: Frequently referred to as the front-end layer, is responsible for managing the user interface and facilitating end-user interactions. In the context of WordPress, this layer encompasses the web server, which processes HTTP requests and presents web pages to visitors.
The data layer: Sometimes referred to as the back-end layer, is responsible for data storage and access. In the case of WordPress, this involves managing the database that stores articles, comments, user information, and so on.
Advantages of a Two-Tier Architecture:
Clear Separation: A two-tier architecture distinctly separates responsibilities between the presentation and data layers, simplifying application management and maintenance.


Scalability and Flexibility: Scaling the presentation layer without affecting the data layer allows for outstanding performance and a seamless user experience, especially during traffic surges.

Infrastructure Planning
The overall architecture is illustrated in the diagram below, featuring the highlighted AWS Cloud Services:

No alt text provided for this image
Prerequisites
Make sure you have the following prerequisites in place before beginning the deployment process:



Basic knowledge of Terraform & AWS
Installed AWS CLI, as documented here
Installed Terraform CLI
Let us now begin customizing our project.

Step 1: Configure the Provider
Create a file called provider.tf and use the following code:

provider "aws" {
  region = "ca-central-1"
}

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "4.65.0"
    }
  }
}        
The provider.tf file configures Terraform's AWS provider by defining the AWS region and required provider version.

This file defines the following resources:


Provider “aws”: Specifies Terraform’s AWS provider. It specifies “ca-central-1” as the region, indicating that the resources would be provisioned in the AWS Canada (Central) zone. This resource’s role is to authenticate Terraform with the specified AWS region and grant it access to AWS resources.
Terraform block: Defines the Terraform configuration’s needed providers. It says that the “aws” provider is required with a specific version of 4.65.0 in this scenario. This block guarantees that the correct AWS provider version is utilized for the setup.
Step 2: Configure VPC and Subnets
To hold the essential variables, create a file called variables.tf:

variable "inbound_port_production_ec2" {
  type        = list(any)
  default     = [22, 80]
  description = "inbound port allow on production instance"
}

variable "db_name" {
  type    = string
  default = "wordpressdb"
}

variable "db_user" {
  type    = string
  default = "admin"
}

variable "db_password" {
  type    = string
  default = "Wordpress-AWS2Tier"
}

variable "instance_type" {
  type    = string
  default = "t2.micro"
}

variable "ami" {
  type    = string
  default = "ami-0940df33750ae6e7f"
}

variable "key_name" {
  type    = string
  default = "wordpressKey"
}

variable "availability_zone" {
  type    = list(string)
  default = ["ca-central-1a", "ca-central-1b", "ca-central-1d"]
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "subnet_cidrs" {
  type        = list(string)
  description = "list of all cidr for subnet"
  default     = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24", "10.0.4.0/24", "10.0.5.0/24"]
}

variable "target_application_port" {
  type    = string
  default = "80"
}        
The variables.tf file defines and initializes the Terraform variables required for configuring and deploying the 2-Tier architecture on Amazon Web Services. Inbound ports, database settings, instance type, AMI ID, availability zones, VPC CIDR, subnet CIDRs, and target application ports are among the variables listed. Customize these variables to meet the needs of your project.

Note: Remember to create the Key pair wordpressKey.

Create a vpc.tf file and add the following content:

resource "aws_vpc" "infrastructure_vpc" 
  cidr_block           = var.vpc_cidr
  instance_tenancy     = "default"
  enable_dns_support   = "true" //gives you an internal domain name
  enable_dns_hostnames = "true" //gives you an internal host name
  instance_tenancy     = "default"
  tags = {
    Name = "2-tier-architecture-vpc"
  }
}


# It enables our vpc to connect to the internet
resource "aws_internet_gateway" "tier_architecture_igw" {
  vpc_id = aws_vpc.infrastructure_vpc.id
  tags = {
    Name = "2-tier-architecture-igw"
  }
}


# first ec2 instance public subnet
resource "aws_subnet" "ec2_1_public_subnet" {
  vpc_id                  = aws_vpc.infrastructure_vpc.id
  cidr_block              = var.subnet_cidrs[1]
  map_public_ip_on_launch = "true" //it makes this a public subnet
  availability_zone       = var.availability_zone[0]
  tags = {
    Name = "first ec2 public subnet"
  }
}


# second ec2 instance public subnet
resource "aws_subnet" "ec2_2_public_subnet" {
  vpc_id                  = aws_vpc.infrastructure_vpc.id
  cidr_block              = var.subnet_cidrs[2]
  map_public_ip_on_launch = "true" //it makes this a public subnet
  availability_zone       = var.availability_zone[1]
  tags = {
    Name = "second ec2 public subnet"
  }
}


# database private subnet
resource "aws_subnet" "database_private_subnet" {
  vpc_id                  = aws_vpc.infrastructure_vpc.id
  cidr_block              = var.subnet_cidrs[4]
  map_public_ip_on_launch = "false" //it makes this a private subnet
  availability_zone       = var.availability_zone[1]
  tags = {
    Name = "database private subnet"
  }
}


# database read replica private subnet
resource "aws_subnet" "database_read_replica_private_subnet" {
  vpc_id                  = aws_vpc.infrastructure_vpc.id
  cidr_block              = var.subnet_cidrs[3]
  map_public_ip_on_launch = "false"
  availability_zone       = var.availability_zone[0]
  tags = {
    Name = "database read replica private subnet"
  }
}        
The Virtual Private Cloud (VPC) and subnets for the 2-Tier architecture are defined in the vpc.tf file. It establishes the VPC using the provided CIDR block, as well as DNS support, hostnames, and a gateway to link the VPC to the internet. It also generates two public subnets for EC2 instances, a private subnet for the database, and a read replica for the database.

Prepare the route_table.tf file and include the following content:

resource "aws_route_table" "infrastructure_route_table" {
  vpc_id = aws_vpc.infrastructure_vpc.id

  route {
    //associated subnet can reach everywhere
    cidr_block = "0.0.0.0/0"
    //CRT uses this IGW to reach internet
    gateway_id = aws_internet_gateway.tier_architecture_igw.id
  }

}

# attach ec2 1 subnet to an internet gateway
resource "aws_route_table_association" "route-ec2-1-subnet-to-igw" {
  subnet_id      = aws_subnet.ec2_1_public_subnet.id
  route_table_id = aws_route_table.infrastructure_route_table.id
}

# attach ec2 2 subnet to an internet gateway
resource "aws_route_table_association" "route-ec2-2-subnet-to-igw" {
  subnet_id      = aws_subnet.ec2_2_public_subnet.id
  route_table_id = aws_route_table.infrastructure_route_table.id
}        
The route_table.tf file configures the VPC's route table. It generates a route table with the internet gateway as the default route. It then links the EC2 subnets to the route table, allowing them to connect to the internet through the configured route.

Step 3: Establish Security Groups
Create the security_group.tf file and include the following content:

# Create a security  group for production node to allow traffic 
resource "aws_security_group" "production-instance-sg" {
  name        = "production-instance-sg"
  description = "Security from who allow inbound traffic on port 22 and 9090"
  vpc_id      = aws_vpc.infrastructure_vpc.id

  # dynamic block who create two rules to allow inbound traffic 
  dynamic "ingress" {
    for_each = var.inbound_port_production_ec2
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = -1
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Create a security  group for database to allow traffic on port 3306 and from ec2 production security group
resource "aws_security_group" "database-sg" {
  name        = "database-sg"
  description = "security  group for database to allow traffic on port 3306 and from ec2 production security group"
  vpc_id      = aws_vpc.infrastructure_vpc.id

  ingress {
    description     = "Allow traffic from port 3306 and from ec2 production security group"
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.production-instance-sg.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = -1
    cidr_blocks = ["0.0.0.0/0"]
  }
}        
The security_group.tf file uses Terraform to define two AWS security groups. The first security group (aws_security_group.production-instance-sg) accepts inbound traffic from any source (0.0.0.0/0) on defined ports (these ports are provided in the file variables.tf variable inbound_port_production_ec2). The second security group (aws_security_group.database-sg) enables inbound traffic from the production security group on port 3306. All outbound traffic is permitted by both security groups.

Step 4: Implement an Application Load Balancer.
Create a loadbalancer.tf file and include the following content:

# Creation of application LoadBalancer
resource "aws_lb" "application_loadbalancer" {
  name               = "application-loadbalancer"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.production-instance-sg.id]
  subnets            = [aws_subnet.ec2_2_public_subnet.id, aws_subnet.ec2_1_public_subnet.id]
}

# Target group for application loadbalancer
resource "aws_lb_target_group" "target_group_alb" {
  name     = "target-group-alb"
  port     = var.target_application_port
  protocol = "HTTP"
  vpc_id   = aws_vpc.infrastructure_vpc.id
}
# attach target group to an instance
resource "aws_lb_target_group_attachment" "attachment1" {
  target_group_arn = aws_lb_target_group.target_group_alb.arn
  target_id        = aws_instance.production_1_instance.id
  port             = var.target_application_port
  depends_on = [
    aws_instance.production_1_instance,
  ]
}
# attach target group to an instance
resource "aws_lb_target_group_attachment" "attachment2" {
  target_group_arn = aws_lb_target_group.target_group_alb.arn
  target_id        = aws_instance.production_2_instance.id
  port             = var.target_application_port
  depends_on = [
    aws_instance.production_2_instance,
  ]
}
# attach target group to a loadbalancer
resource "aws_lb_listener" "external-elb" {
  load_balancer_arn = aws_lb.application_loadbalancer.arn
  port              = var.target_application_port
  protocol          = "HTTP"
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.target_group_alb.arn
  }
}        
The loadbalancer.tf file uses Terraform to configure the application load balancer (ALB) and its associated resources. It builds the ALB with the name, type, security groups, and subnets that you specify. It also defines the ALB's target group, adds production instances to the group, and configures the listener to transmit traffic to the group.

Step 5: Create EC2 instances and an RDS database.
You must construct a main.tf file with the following content to provide EC2 instances and an RDS database:

resource "aws_instance" "production_1_instance" {
  ami                    = var.ami
  instance_type          = var.instance_type
  subnet_id              = aws_subnet.ec2_1_public_subnet.id
  vpc_security_group_ids = [aws_security_group.production-instance-sg.id]
  key_name               = var.key_name
  user_data              = file("install_script.sh")
  tags = {
    Name = "Production instance 1"
  }
  depends_on = [
    aws_db_instance.rds_master,
  ]
}

resource "aws_instance" "production_2_instance" {
  ami                    = var.ami
  instance_type          = var.instance_type
  subnet_id              = aws_subnet.ec2_2_public_subnet.id
  vpc_security_group_ids = [aws_security_group.production-instance-sg.id]
  key_name               = var.key_name
  user_data              = file("install_script.sh")
  tags = {
    Name = "Production instance 2"
  }
  depends_on = [
    aws_db_instance.rds_master,
  ]
}
resource "aws_db_subnet_group" "database_subnet" {
  name       = "db subnet"
  subnet_ids = [aws_subnet.database_private_subnet.id, aws_subnet.database_read_replica_private_subnet.id]
}
resource "aws_db_instance" "rds_master" {
  identifier              = "master-rds-instance"
  allocated_storage       = 10
  engine                  = "mysql"
  engine_version          = "5.7.37"
  instance_class          = "db.t3.micro"
  db_name                 = var.db_name
  username                = var.db_user
  password                = var.db_password
  backup_retention_period = 7
  multi_az                = false
  availability_zone       = var.availability_zone[1]
  db_subnet_group_name    = aws_db_subnet_group.database_subnet.id
  skip_final_snapshot     = true
  vpc_security_group_ids  = [aws_security_group.database-sg.id]
  storage_encrypted       = true
  tags = {
    Name = "my-rds-master"
  }
}
resource "aws_db_instance" "rds_replica" {
  replicate_source_db    = aws_db_instance.rds_master.identifier
  instance_class         = "db.t3.micro"
  identifier             = "replica-rds-instance"
  allocated_storage      = 10
  skip_final_snapshot    = true
  multi_az               = false
  availability_zone      = var.availability_zone[0]
  vpc_security_group_ids = [aws_security_group.database-sg.id]
  storage_encrypted      = true
  tags = {
    Name = "my-rds-replica"
  }
}        
The main.tf file includes the core Terraform settings for generating various AWS resources. It defines EC2 instances, RDS instances, and a database subnet group's resources.

This file defines the following resources:


aws_instance “production_1_instance” and “production_2_instance”: The instances use the AMI, instance type, subnet ID, security group ID, key name, user data script, and tags that were supplied. They are dependent on the RDS master instance’s availability.
the aws_db_subnet_group “database_subnet”: resource defines the database subnet group, specifying a name and the private subnet IDs where RDS instances will be deployed.
aws_db_instance “rds_master”: This resource is responsible for creating the master RDS instance. It defines the database engine, engine version, instance class, database name, username, password, backup retention period, availability zone, subnet group name, security group ID, and other setup settings. It identifies the RDS instance as my-rds-master.
aws_db_instance “rds_replica”: generates a replica RDS instance that replicates from the master RDS instance. It has the same setup choices as the previous one, but with a different identity and availability zone.
Do the following changes to the install_script.sh file:

#!/bin/bash

#install docker 
sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo apt  install docker-compose -y
sudo systemctl enable docker
sudo systemctl start docker
sudo echo ${aws_db_instance.rds_master.password} | sudo tee /root/db_password.txt > /dev/null
sudo echo ${aws_db_instance.rds_master.username} | sudo tee /root/db_username.txt > /dev/null
sudo echo ${aws_db_instance.rds_master.endpoint} | sudo tee /root/db_endpoint.txt > /dev/null
# create docker-compose
sudo cat <<EOF | sudo tee docker-compose.yml > /dev/null
version: '3'
services:
  wordpress:
   image: wordpress:4.8-apache
   ports:
    - 80:80
   environment:
     WORDPRESS_DB_USER: $(cat /root/db_username.txt)
     WORDPRESS_DB_HOST: $(cat /root/db_endpoint.txt)
     WORDPRESS_DB_PASSWORD: $(cat /root/db_password.txt)
   volumes:
     - wordpress-data:/var/wwww/html
volumes:
  wordpress-data:
EOF



# Next Do this Command
sudo docker-compose up
        
The shell script install_script.sh installs Docker and Docker-compose, configures a WordPress environment and executes the application.

Create a file called outputs.tf and add the following content:

output "public_1_ip" {
  value = aws_instance.production_1_instance.public_ip
}

output "public_2_ip" {
  value = aws_instance.production_2_instance.public_ip
}
output "rds_endpoint" {
  value = aws_db_instance.rds_master.endpoint
}
output "rds_username" {
  value = aws_db_instance.rds_master.username
}

output "rds_name" {
  value = aws_db_instance.rds_master.db_name
}
# print the DNS of load balancer
output "lb_dns_name" {
  description = "The DNS name of the load balancer"
  value       = aws_lb.application_loadbalancer.dns_name
}        
The outputs.tf file is used to define the outputs of the Terraform code's resources. It specifies the EC2 instances' public IP addresses, the RDS endpoint, username, and database name, as well as the application load balancer's DNS name.

The application load balancer will utilize these outputs to connect to the RDS database instance and access the application.

Step 6: Deployment
To begin the deployment process, we must first install Terraform in our project directory. This step ensures that Terraform downloads the required providers and configures the backend. In your terminal, type the following command:

terraform init        
No alt text provided for this image
After launching Terraform, we can produce an execution plan to preview the modifications to our AWS infrastructure. The plan describes in detail the resources that will be generated, updated, or eliminated.

Execute the following command to generate the plan:

terraform plan        
No alt text provided for this image
No alt text provided for this image
We can proceed with deploying our infrastructure on AWS after we are satisfied with the execution plan. Terraform will supply the required resources and configure them to our specifications.

Run the following command to deploy the infrastructure:

terraform apply        
Before proceeding with the deployment, Terraform will ask for confirmation. To proceed, type yes and hit Enter.

No alt text provided for this image
It will take some time to complete the deployment procedure. Terraform will show the status and progress of each resource being generated.

No alt text provided for this image
Terraform will return information about the created resources, including as IP addresses, DNS names, RDS endpoint, user, and database names, once the deployment is complete.

No alt text provided for this image
Finally, let us check our setup in our AWS Console:

No alt text provided for this image
No alt text provided for this image
No alt text provided for this image
No alt text provided for this image
No alt text provided for this image
The WordPress installation is accessible via the Load Balancer domain that was generated:

No alt text provided for this image
NB: It’s worth noting that we can use AWS ACM to connect it to a custom domain and an HTTP certificate.
