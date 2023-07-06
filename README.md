# Terraform-module-VPC

The configuration can be contained and reused across several projects or contexts by turning a resource into a Terraform module. Creating a Terraform module for the establishment of a VPC is demonstrated here.

> A methodical approach

* Create a new directory for your module, such as **terraform-modules** or name its as you like.
* Inside the **terraform-modules** directory, create some files named as main.tf,variables.tf,output.tf,datasource.tf.

> Create variables.tf
```
variable "main_network" {
    default = "172.16.0.0/16"
}
variable "project_name" {}
variable "project_env" {}
variable "enable_natgw" {
    type = bool
    default = false
    }
```
> Create datasource.tf

Retrieve the list of availability zones in a particular region.
```
data "aws_availability_zones" "available" {
  state = "available"
}
```
> Create main.tf

In the main.tf file, define resources like VPC CIDR block, Internet gateway, Public & Private subnets, Elastic IP, Nat gateway, Public & Private route tables, Private route if the nat gateway is enabled, Private & Public route table assosiation.
```
resource "aws_vpc" "vpc" {
  cidr_block       = var.main_network
  instance_tenancy = "default"
    enable_dns_hostnames = true

  tags = {
      Name      = "${var.project_name}-${var.project_env}"
  }
}
    
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.vpc.id

  tags = {
    Name      = "${var.project_name}-${var.project_env}"
  }
}
    
resource "aws_subnet" "public" {
    count = 3
  vpc_id     = aws_vpc.vpc.id
  cidr_block = cidrsubnet(var.main_network, 3, "${count.index}")
    map_public_ip_on_launch = true
    availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name      = "${var.project_name}-${var.project_env}-public-${count.index +1}"
  }
}
resource "aws_subnet" "private" {
    count = 3
  vpc_id     = aws_vpc.vpc.id
  cidr_block = cidrsubnet(var.main_network, 3, "${count.index +3}")
    map_public_ip_on_launch = false
    availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name      = "${var.project_name}-${var.project_env}-private-${count.index +1}"
  }
}
resource "aws_eip" "nat-gateway" {
    count = var.enable_natgw == true ? 1 : 0
  
  domain   = "vpc"
      tags = {
    Name      = "${var.project_name}-${var.project_env}-nat-gateway"
      }
}
    
resource "aws_nat_gateway" "nat-gateway" {
    count = var.enable_natgw == true ? 1 : 0
  allocation_id = aws_eip.nat-gateway.0.id
  subnet_id     = aws_subnet.public.1.id

  tags = {
    Name      = "${var.project_name}-${var.project_env}-nat-gateway"
  }
     depends_on = [aws_internet_gateway.igw]
}
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
        Name      = "${var.project_name}-${var.project_env}-route-public"
  }
}
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.vpc.id

  tags = {
        Name      = "${var.project_name}-${var.project_env}-route-private"
  }
}

resource "aws_route" "private" {
    count = var.enable_natgw == true ? 1 : 0
  route_table_id              = aws_route_table.private.id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id = aws_nat_gateway.nat-gateway.0.id
}

resource "aws_route_table_association" "public" {
    count =3
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
    count = 3
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}
```

> Create outputs.tf

The purpose of this is to specify the outputs that your module will make available.
```
output "vpc_id" {
    value = aws_vpc.vpc.id
}

output "public_subnet_ids" {
    value = aws_subnet.public.*.id
}

output "private_subnet_ids" {
    value = aws_subnet.private.*.id
}

output "public-1_subnet_ids" {
    value = aws_subnet.public.0.id
}

output "public-2_subnet_ids" {
    value = aws_subnet.public.1.id
}

output "public-3_subnet_ids" {
    value = aws_subnet.public.3.id
}

output "private-1_subnet_ids" {
    value = aws_subnet.private.0.id
}

output "private-2_subnet_ids" {
    value = aws_subnet.private.1.id
}

output "private-3_subnet_ids" {
    value = aws_subnet.private.3.id
}
```

**Then create a project dir. And create the files like main.tf variables.tf provider.tf and prod.tfvars. In the Terraform project directory,follow the below steps**

> Create variables.tf to pass values
```
variable "access_key" {
    default = "Your access key"
}

variable "secret_key" {
  default = "Your secret key"  
}

variable "region" {
    default = "ap-south-1"
}

variable "main_network" {
    default = "172.16.0.0/16"
}

variable "enable_natgw" {
    type = bool
    default = true
    
}

variable "ami_id" {
    description = "ami_id"
    type = string
    default = "ami-057752b3f1d6c4d6c"
}

variable "instance_type" {}

variable "project_name" {}

variable "project_env" {}

variable "sg_ports_bastion" {
  type    = list(any)
  default = ["22"]
}

variable "sg_ports_frontend" {
  type    = list(any)
  default = ["22", "80", "443"]
}

variable "sg_ports_backend" {
  type    = list(any)
  default = ["22", "3306"]
}
```

> Create provider.tf
```
provider "aws" {
  region     = var.region
  access_key = var.access_key
  secret_key = var.secret_key
    default_tags {
   tags = {
     Env = var.project_env
     Owner       = "Jijin"
     Project     = var.project_name
   }
 }
}
```

> Create main.tf
```
module "vpc" {
    source = "/home/jijin/Downloads/terraform/terraform-modules" 
    main_network = var.main_network
    project_name = var.project_name
    project_env = var.project_env
    enable_natgw = var.enable_natgw
}

resource "aws_key_pair" "uber" {
  key_name   = "${var.project_name}-${var.project_env}"
  public_key = file("uber.pub")
  tags = {
    "Name"    = "${var.project_name}-${var.project_env}"
    
  }
}

resource "aws_security_group" "Bastion" {
  name        = "${var.project_name}-${var.project_env}-Bastion"
  description = "Bastion"
    vpc_id = module.vpc.vpc_id

  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  tags = {
    Name      = "${var.project_name}-${var.project_env}-Bastion"
  }
}
resource "aws_security_group_rule" "Bastion" {
  for_each          = toset(var.sg_ports_bastion)
  type              = "ingress"
  from_port         = each.key
  to_port           = each.key
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  ipv6_cidr_blocks  = ["::/0"]
  security_group_id = aws_security_group.Bastion.id
}

resource "aws_security_group" "Frontend" {
  name        = "${var.project_name}-${var.project_env}-Frontend"
  description = "Frontend"
    vpc_id = module.vpc.vpc_id

  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  tags = {
    Name      = "${var.project_name}-${var.project_env}-Frontend"
  }
}
resource "aws_security_group_rule" "Frontend" {
  for_each          = toset(var.sg_ports_frontend)
  type              = "ingress"
  from_port         = each.key
  to_port           = each.key
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  ipv6_cidr_blocks  = ["::/0"]
  security_group_id = aws_security_group.Frontend.id
}

resource "aws_security_group" "Backend" {
  name        = "${var.project_name}-${var.project_env}-Backend"
  description = "Backend"
    vpc_id = module.vpc.vpc_id

  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  tags = {
    Name      = "${var.project_name}-${var.project_env}-Backend"
  }
}
resource "aws_security_group_rule" "Backend" {
  for_each          = toset(var.sg_ports_backend)
  type              = "ingress"
  from_port         = each.key
  to_port           = each.key
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  ipv6_cidr_blocks  = ["::/0"]
  security_group_id = aws_security_group.Backend.id
}


resource "aws_instance" "Bastion" {
  ami                         = var.ami_id
  instance_type               = var.instance_type
    subnet_id = module.vpc.public-1_subnet_ids

  key_name               = aws_key_pair.uber.key_name
  vpc_security_group_ids = [aws_security_group.Bastion.id]
  tags                   = { "Name" = "${var.project_name}-${var.project_env}-Bastion"
                             
                           }
  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_instance" "Frontend" {
  ami                         = var.ami_id
  instance_type               = var.instance_type
    
    subnet_id = module.vpc.public-2_subnet_ids


  key_name               = aws_key_pair.uber.key_name
  vpc_security_group_ids = [aws_security_group.Frontend.id]
  tags                   = { "Name" = "${var.project_name}-${var.project_env}-Frontend"
                             
                           }
  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_instance" "Backend" {
  ami                         = var.ami_id
  instance_type               = var.instance_type
    
    subnet_id = module.vpc.private-2_subnet_ids

  key_name               = aws_key_pair.uber.key_name
  vpc_security_group_ids = [aws_security_group.Backend.id]
  tags                   = { "Name" = "${var.project_name}-${var.project_env}-Backend"
                           }
  lifecycle {
    create_before_destroy = true
  }
}
```

> Create prod.tfvars
```
instance_type = "t2.micro"
project_env   = "Dev"
project_name  = "Uber"
```

Now that the module has been installed, you may utilise it by launching **terraform init** and **terraform apply -var-file=prod.tfvars** in the root directory of your main project.

Your infrastructure management will be made simpler by designing a Terraform module for a VPC that allows you to reuse and keep consistent configurations across numerous deployments.
