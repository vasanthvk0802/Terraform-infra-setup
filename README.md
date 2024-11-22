# Terraform-infra-setup
Terraform setup using HCL Language
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 3.0"
    }
  }
}

# Configure the AWS Provider
provider "aws" {
  region = "eu-west-2"
}

resource "aws_vpc" "MYVPC_VK" {
  cidr_block       = "10.0.0.0/16"
  instance_tenancy = "default"

  tags = {
    Name = "myvpc"
  }
}

resource "aws_subnet" "public_subnet" {
  vpc_id     = aws_vpc.MYVPC_VK.id
  cidr_block = "10.0.1.0/24"

  tags = {
    Name = "public_SN"
  }
}

resource "aws_subnet" "private_subnet" {
  vpc_id     = aws_vpc.MYVPC_VK.id
  cidr_block = "10.0.2.0/24"

  tags = {
    Name = "private_SN"
  }
}

resource "aws_internet_gateway" "IGW_VK" {
  vpc_id = aws_vpc.MYVPC_VK.id

  tags = {
    Name = "igw"
  }
}

resource "aws_route_table" "public_RT" {
  vpc_id = aws_vpc.MYVPC_VK.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.IGW_VK.id
  }

  tags = {
    Name = "publicrt"
  }
}

resource "aws_route_table_association" "public_subnet_association" {
  subnet_id      = aws_subnet.public_subnet.id
  route_table_id = aws_route_table.public_RT.id
}

resource "aws_eip" "MY_EIP" {
  vpc      = true
}

resource "aws_nat_gateway" "MYNGW_Vk" {
  allocation_id = aws_eip.MY_EIP.id
  subnet_id     = aws_subnet.public_subnet.id

  tags = {
    Name = "NGW"
  }
}

resource "aws_route_table" "private_RT" {
  vpc_id = aws_vpc.MYVPC_VK.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_nat_gateway.MYNGW_Vk.id
  }

  tags = {
    Name = "privatert"
  }
}

resource "aws_route_table_association" "private_subnet_association" {
  subnet_id      = aws_subnet.private_subnet.id
  route_table_id = aws_route_table.private_RT.id
}

resource "aws_security_group" "public_SG" {
  name        = "public_SG"
  description = "Allow TLS inbound traffic"
  vpc_id      = aws_vpc.MYVPC_VK.id

  ingress {
    description      = "TLS from VPC"
    from_port        = 0
    to_port          = 65535
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }
  
  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
  }

  tags = {
    Name = "publicsg"
  }
}


resource "aws_security_group" "private_SG" {
  name        = "private_SG"
  description = "Allow TLS inbound traffic from public subnet"
  vpc_id      = aws_vpc.MYVPC_VK.id

  ingress {
    description      = "TLS from VPC"
    from_port        = 0
    to_port          = 65535
    protocol         = "tcp"
    cidr_blocks      = ["10.0.1.0/24"]
  }

  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
  }

  tags = {
    Name = "privatesg"
  }
}

resource "aws_instance" "Public_instance" {
  ami                                             = "ami-0acc77abdfc7ed5a6"
  instance_type                                   = "t2.micro"
  availability_zone                               = "eu-west-2a"
  associate_public_ip_address                     = "true"
  vpc_security_group_ids                          = [aws_security_group.public_SG.id]
  subnet_id                                       = aws_subnet.public_subnet.id
  key_name                                        = "Terra"

    tags = {
    Name = "My_Public_instance"
  }
}

resource "aws_instance" "Private_instance" {
  ami                                             = "ami-0acc77abdfc7ed5a6"
  instance_type                                   = "t2.micro"
  availability_zone                               = "eu-west-2b"
  associate_public_ip_address                     = "false"
  vpc_security_group_ids                          = [aws_security_group.private_SG.id]
  subnet_id                                       = aws_subnet.private_subnet.id
  key_name                                        = "Terra"

    tags = {
    Name = "My_Private_instance"
  }
}
