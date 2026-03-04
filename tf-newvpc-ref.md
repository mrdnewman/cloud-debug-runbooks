```hcl
provider "aws" {
  region = var.region
}

variable "region" {
  default = "us-east-1"
}

variable "instance_type" {
  default = "t3.micro"
}

variable "instance_names" {
  type = set(string)

  default = [
    "web1",
    "web2"
  ]
}

variable "vpc_cidr" {
  default = "10.0.0.0/16"
}

variable "public_subnets" {
  type = set(string)

  default = [
    "10.0.1.0/24",
    "10.0.2.0/24"
  ]
}

data "aws_ami" "amazon_linux" {

  most_recent = true

  owners = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*"]
  }
}

resource "aws_vpc" "main" {

  cidr_block = var.vpc_cidr

  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "main-vpc"
  }
}

resource "aws_internet_gateway" "igw" {

  vpc_id = aws_vpc.main.id

  tags = {
    Name = "main-igw"
  }
}

resource "aws_subnet" "public" {

  for_each = var.public_subnets

  vpc_id = aws_vpc.main.id

  cidr_block = each.key

  map_public_ip_on_launch = true

  tags = {
    Name = "public-${each.key}"
  }
}


resource "aws_route_table" "public_rt" {

  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"

    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "public-rt"
  }
}

resource "aws_route_table_association" "public_assoc" {

  for_each = aws_subnet.public

  subnet_id = each.value.id

  route_table_id = aws_route_table.public_rt.id
}

resource "aws_security_group" "web_sg" {

  name = "web-sg"

  vpc_id = aws_vpc.main.id

  ingress {

    from_port = 80
    to_port   = 80
    protocol  = "tcp"

    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {

    from_port = 22
    to_port   = 22
    protocol  = "tcp"

    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {

    from_port = 0
    to_port   = 0
    protocol  = "-1"

    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "web" {

  for_each = var.instance_names

  ami = data.aws_ami.amazon_linux.id

  instance_type = var.instance_type

  subnet_id = values(aws_subnet.public)[0].id

  vpc_security_group_ids = [
    aws_security_group.web_sg.id
  ]

  tags = {
    Name = each.key
  }
}

output "vpc_id" {
  value = aws_vpc.main.id
}

output "instance_ids" {
  value = [
    for i in aws_instance.web :
    i.id
  ]
}

output "instance_public_ips" {
  value = [
    for i in aws_instance.web :
    i.public_ip
  ]
}



