root@ip-10-0-1-75:~# docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED        STATUS        PORTS                               NAMES
3c8b30b009ea   nginx:latest   "/docker-entrypoint.…"   16 hours ago   Up 16 hours   0.0.0.0:80->80/tcp, :::80->80/tcp   docker-nginx
root@ip-10-0-1-75:~# docker run --name docker-nginx-1 -p 81:80 nginx:latest &
[1] 13377
root@ip-10-0-1-75:~# /docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2022/08/19 23:15:06 [notice] 1#1: using the "epoll" event method
2022/08/19 23:15:06 [notice] 1#1: nginx/1.23.1
2022/08/19 23:15:06 [notice] 1#1: built by gcc 10.2.1 20210110 (Debian 10.2.1-6) 
2022/08/19 23:15:06 [notice] 1#1: OS: Linux 4.4.0-1112-aws
2022/08/19 23:15:06 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
2022/08/19 23:15:06 [notice] 1#1: start worker processes
2022/08/19 23:15:06 [notice] 1#1: start worker process 33

root@ip-10-0-1-75:~# docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS          PORTS                               NAMES
3af2a35655ef   nginx:latest   "/docker-entrypoint.…"   17 seconds ago   Up 16 seconds   0.0.0.0:81->80/tcp, :::81->80/tcp   docker-nginx-1
3c8b30b009ea   nginx:latest   "/docker-entrypoint.…"   16 hours ago     Up 16 hours     0.0.0.0:80->80/tcp, :::80->80/tcp   docker-nginx
root@ip-10-0-1-75:~# curl http://localhost:81
172.17.0.1 - - [19/Aug/2022:23:15:52 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/7.47.0" "-"
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
root@ip-10-0-1-75:~# docker stop 3af2a35655ef
2022/08/19 23:16:16 [notice] 1#1: signal 3 (SIGQUIT) received, shutting down
2022/08/19 23:16:16 [notice] 33#33: gracefully shutting down
2022/08/19 23:16:16 [notice] 33#33: exiting
2022/08/19 23:16:16 [notice] 33#33: exit
2022/08/19 23:16:16 [notice] 1#1: signal 17 (SIGCHLD) received from 33
2022/08/19 23:16:16 [notice] 1#1: worker process 33 exited with code 0
2022/08/19 23:16:16 [notice] 1#1: exit
3af2a35655ef
root@ip-10-0-1-75:~# docker rm 3af2a35655ef
3af2a35655ef
[1]+  Done                    docker run --name docker-nginx-1 -p 81:80 nginx:latest
root@ip-10-0-1-75:~# docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED        STATUS        PORTS                               NAMES
3c8b30b009ea   nginx:latest   "/docker-entrypoint.…"   16 hours ago   Up 16 hours   0.0.0.0:80->80/tcp, :::80->80/tcp   docker-nginx
root@ip-10-0-1-75:~# client_loop: send disconnect: Broken pipe
root@worker01:~# cat main.tf
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
