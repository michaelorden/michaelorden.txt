# Defining providers we will use in this deployement
# We only need the aws provides. Terraform will retrive the aws provider from hashicorp bublic registry
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "2.20.2"
    }
    aws = {
      source  = "hashicorp/aws"
      version = "~> 3.0"
    }
  }
}
provider "aws" {
  profile = "default"
  region  = "us-west-2"
}
# First we need to create an aws VPC. We need a VPC to run EC2 ubuntu ami
# I am going to run an ami-08d70e59c07c61a3a with t2.micro instnace_type to keep things free
# ami ids change from region to region. You can find them here https://cloud-images.ubuntu.com/locator/ec2/
# Creating the VPC
# The code block below uses the aws_vpc module from the aws provider in hashicorp registry
# Documentation available here: https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest
resource "aws_vpc" "nginx-vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = "true"
  enable_dns_hostnames = "true"
  enable_classiclink   = "false"
  instance_tenancy     = "default"
}
# Creating a public subnet
# Public subnet is required for the instances in the VPC to communicate over the internet
# Documentation is available here: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/subnet
resource "aws_subnet" "prod-subnet-public-1" {
  vpc_id                  = aws_vpc.nginx-vpc.id // Referencing the id of the VPC from abouve code block
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = "true" // Makes this a public subnet
  availability_zone       = "us-west-2a"
}
# Creating an Internet Gateway
# Require for the VPC to communicate with the internet
# Documentation is available here: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/internet_gateway
resource "aws_internet_gateway" "prod-igw" {
  vpc_id = aws_vpc.nginx-vpc.id
}
# Create a custom route table for public subnets
# Public subnets can reach to the internet buy using this
# Documentation is available here: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route_table
resource "aws_route_table" "prod-public-crt" {
  vpc_id = aws_vpc.nginx-vpc.id
  route {
    cidr_block = "0.0.0.0/0"                      //associated subnet can reach everywhere
    gateway_id = aws_internet_gateway.prod-igw.id //CRT uses this IGW to reach internet
  }

  tags = {
    Name = "prod-public-crt"
  }
}
# Route table association for the public subnets
# Documentation is available here: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route_table_association
resource "aws_route_table_association" "prod-crta-public-subnet-1" {
  subnet_id      = aws_subnet.prod-subnet-public-1.id
  route_table_id = aws_route_table.prod-public-crt.id
}
# Security group
# Creating this so we can SSH into the VPC and the ami instance to instal nginx port 22
# This also allow us to access the nginx server via the public ip address port 80
# Documentation is available here: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group
resource "aws_security_group" "ssh-allowed" {

  vpc_id = aws_vpc.nginx-vpc.id


  egress {
    from_port   = 0
    to_port     = 0
    protocol    = -1
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
# Setting up the aws ssh key. You need to generate one and store it in the same directory
# We use this to establish the SSH connection to install nginx using remore-exec
# I think these are best handles using the terraform cloud workspace. I'm not sure how to do that yet.
# Documentation is available here: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/key_pair
resource "aws_key_pair" "aws-key" {
  key_name   = "aws-key"
  public_key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDLfpBl1iJSMGc2nEWLErxD6ng/VJ6UtXgQLyt8KeOvc1xDLNM6OVTKhwwQwOcBCj8ayszzAgvwNMNtvIML8Sf4vGd09jHeKU9hlMJ01GYzgjoQG2TWG029x3S4WS0uX4l+s3C1KduhLu31XTpHMjQ3/TW/EWHoAK9hrucTDcK70+ftNuFXUc3RqKuLcs9hudT8eBiw+SLHGBMSBpfeVj6+yzUyiASud6G1OOiDfoDXevMlQWIRwHb4Em2S0ElTjPQLEdU2lsUgMKMAMt5y/k4bBI5OAFRNgFY54/aJhEvBXB4BoDWomfsn6E5QPNSCxIwcVfFuimbzxA/q399IprD4O9jFdWr3g4UzzLsL7ze85n7rDqtNIpCn2IIhwWuwOiG/2kqHIisfddU7WGxZcPA0XIaKQlQEiafUAFw2kDqLp20OANf9WhEoS3/NY9RTIYekdntFxfHrBeDdQnpWUG1qoywugE643bA5wlGCzRsCHuffmv6mbDQoYYhLMRiLAF0= root@worker01"
}
# Setting up the EC2 instnace
# We are installing ubunto as the core OD
resource "aws_instance" "nginx_server" {
  ami           = "ami-08d70e59c07c61a3a"
  instance_type = "t2.micro"
  tags = {
    Name = "nginx_server"
  }

  # VPC
  subnet_id = aws_subnet.prod-subnet-public-1.id
  # Security Group
  vpc_security_group_ids = ["${aws_security_group.ssh-allowed.id}"]
  # the Public SSH key
  key_name = aws_key_pair.aws-key.id

  # nginx installation
  # storing the nginx.sh file in the EC2 instnace
  provisioner "file" {
    source      = "nginx1.sh"
    destination = "/tmp/nginx1.sh"
  }
  # Exicuting the nginx.sh file
  # Terraform does not reccomend this method becuase Terraform state file cannot track what the scrip is provissioning
  provisioner "remote-exec" {
    inline = [
      "chmod +x /tmp/nginx1.sh",
      "sudo /tmp/nginx1.sh"
    ]
  }
# docker installation
  # storing the docker.sh file in the EC2 instnace
#  provisioner "file" {
#    source      = "docker.sh"
#    destination = "/tmp/docker.sh"
#  }
  # Exicuting the docker.sh file
  # Terraform does not reccomend this method becuase Terraform state file cannot track what the scrip is provissioning
#  provisioner "remote-exec" {
#    inline = [
#      "chmod +x /tmp/docker.sh",
#      "sudo /tmp/docker.sh"
#    ]
#  }

  # Setting up the ssh connection to install the nginx server
  connection {
    type        = "ssh"
    host        = self.public_ip
    user        = "ubuntu"
    private_key = file("${var.PRIVATE_KEY_PATH}")
  }
}
