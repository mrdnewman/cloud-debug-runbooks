```hcl
provider "aws" {
  region = var.region
}


variable "region" {
  default = "us-east-1"
}

variable "instance_names" {
  type = set(string)

  default = [
    "web1",
    "web2"
  ]
}

variable "instance_type" {
  default = "t3.micro"
}


data "aws_vpc" "default" {
  default = true
}

data "aws_subnets" "vpc_subnets" {
  filter {
    name = "vpc-id"

    values = [
      data.aws_vpc.default.id
    ]
  }
}


data "aws_ami" "amazon_linux" {
  most_recent = true

  owners = ["amazon"]

  filter {
    name = "name"

    values = [
      "amzn2-ami-hvm-*"
    ]
  }
}

resource "aws_security_group" "web_sg" {

  name = "web-sg"

  vpc_id = data.aws_vpc.default.id

  ingress {

    from_port = 80
    to_port   = 80
    protocol  = "tcp"

    cidr_blocks = [
      "0.0.0.0/0"
    ]
  }

  egress {

    from_port = 0
    to_port   = 0
    protocol  = "-1"

    cidr_blocks = [
      "0.0.0.0/0"
    ]
  }
}

resource "aws_instance" "web" {

  for_each = var.instance_names

  ami = data.aws_ami.amazon_linux.id

  instance_type = var.instance_type

  subnet_id = tolist(
    data.aws_subnets.vpc_subnets.ids
  )[0]

  vpc_security_group_ids = [
    aws_security_group.web_sg.id
  ]

  tags = {
    Name = each.key
  }
}
