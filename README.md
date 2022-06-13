
# infrastructure-as-code 

Conversion of AWS cloud-formation to terraform Modules - 2. VPC Module and multi environment
Description:
Need to create resources managed by AWS cloudFormation to terraform- Modules & Multi-env 
**Action plan for VPC Module**
1. Create re-usable module
2. create dev/shared/stage environments
3. create the resources in 3 env by calling module.

=========AWS VPC Terraform module=============

Terraform module which creates VPC resources on AWS.

Configuration in this directory creates set of VPC resources which may be sufficient for development environment.

There is a public and private subnet created per availability zone.

Usage
To run this example you need to execute:

$ terraform init
$ terraform plan
$ terraform apply

Note: Run terraform destroy when you don't need these resources as some of the resources may cost money

Requirements	    
-   terraform version
    aws version     
-   Providers
    - aws
-   Modules
-   Resources

Templete structure:

Module/Networking:
- vpc.tf
- variable.tf
- output.tf
Module/DNS
- route53.tf

Environments

we need to create the terraform workspace to differentiat between environments. we need to create different workspace for each environment(dev/shared/stage).
to create terrarom workspace:

terraform workspace new <workspace-name>

- Dev
- Shared
- Stage

=====    Introduction  =====

1. Create VPC using Terraform Modules
2. Define Input Variables for VPC module and reference them in VPC Terraform Module
3. Define local values and reference them in VPC Terraform Module
4. Create terraform.tfvars to load variable values by default from this file
5. Define Output Values for VPC

=====    vpc-module/Networking   =====
In this section, we create local moduel using standard terraform resource definition approach.
Terraform module which creates VPC resources on AWS.
teraform files:
-> vpc.tf
-> variable.tf
-> output.tf

vpc.tf: 
=======
we need to create below resources in this module.

```terraform
/* Local declaration */

locals {
  tagName = "${terraform.workspace}-${var.product}-${var.service}"
  //az_names = "${data.aws_availability_zones.public_subnet_availbility_zone.names}"
  //public_subnet_ids = "${aws_subnet.cary_subnet_public.*.id}"
  stackName = "${terraform.workspace}-${var.product}"
}


resource "aws_vpc" "vpc" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = {
    Name        = "${terraform.workspace}-${var.product}-${var.service}"
    Environment = terraform.workspace
  }
}

/* Public subnet  */
resource "aws_subnet" "public_subnet" {
  vpc_id                  = aws_vpc.vpc.id
  count                   = length(var.public_subnets_cidr)
  cidr_block              = var.public_subnets_cidr[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = false
  tags = {
    Name        = "${var.environment}-${element(var.availability_zones, count.index)}- public-subnet"
    Environment = "${var.environment}"
  }
}

/* Private subnet */

resource "aws_subnet" "private_subnet" {
  vpc_id                  = aws_vpc.vpc.id
  count                   = length(var.private_subnets_cidr)
  cidr_block              = var.private_subnets_cidr[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = false
  tags = {
    Name        = "${var.environment}-${element(var.availability_zones, count.index)}-private-subnet"
    Environment = "${var.environment}"
  }
}


/*==== Subnets ======*/
/* Internet gateway for the public subnet */
resource "aws_internet_gateway" "ig" {
  vpc_id = aws_vpc.vpc.id
  tags = {
    Name        = "${var.environment}-${var.product}-${var.service}-igw"
    Environment = terraform.workspace
  }
}


resource "aws_security_group" "generalSG" {
  name        = "${var.environment}-general-sg"
  description = "Default security group to allow inbound/outbound from the VPC"
  vpc_id      = aws_vpc.vpc.id
  depends_on  = [aws_vpc.vpc]

  tags = {
    Environment = terraform.workspace
  }
}


resource "aws_security_group_rule" "ingress_rules" {
  count = length(var.sg_ingress_rules)

  type              = "ingress"
  from_port         = var.sg_ingress_rules[count.index].from_port
  to_port           = var.sg_ingress_rules[count.index].to_port
  protocol          = var.sg_ingress_rules[count.index].protocol
  cidr_blocks       = [var.sg_ingress_rules[count.index].cidr_block]
  description       = var.sg_ingress_rules[count.index].description
  security_group_id = aws_security_group.generalSG.id
  depends_on        = [aws_security_group.generalSG]
}


/* NAT instance */

resource "aws_instance" "cary_nat" {
  ami = var.cary_nat_ami[var.aws_region]

  instance_type          = "t2.micro"
  subnet_id              = element(aws_subnet.public_subnet.*.id, 0) #"${local.public_subnet_ids[0]}"
  source_dest_check      = false
  vpc_security_group_ids = ["${aws_security_group.nat_SG.id}"]
  tags = {
    Name  = "${local.tagName}-nat-instance"
    stack = "${local.stackName}"
  }
}

/* NAT Security Group */

resource "aws_security_group" "nat_SG" {
  name        = "nat_SG"
  description = "Allow traffic for private subnets"
  vpc_id      = aws_vpc.vpc.id
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]

  }

  tags = {
    Name = "${local.tagName}-nagSG"
  }
}


/* Routing table for public subnet */
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.vpc.id
  tags = {
    Name        = "${local.tagName}-public-route-table"
    Environment = terraform.workspace
  }
}

/* Routing table for private subnet */

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.vpc.id
  route {
    cidr_block  = "0.0.0.0/0"
    instance_id = aws_instance.cary_nat.id
  }
  tags = {
    Name = "${local.tagName}-private-route-table"
  }
}

/* public_internet_gateway */

resource "aws_route" "public_internet_gateway" {
  route_table_id         = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.ig.id
}


/* Route table associations */
resource "aws_route_table_association" "public" {
  count          = length(var.public_subnets_cidr)
  subnet_id      = element(aws_subnet.public_subnet.*.id, count.index)
  route_table_id = aws_route_table.public.id
}

/* private route_table_association */

resource "aws_route_table_association" "private" {
  count          = length(var.private_subnets_cidr)
  subnet_id      = element(aws_subnet.private_subnet.*.id, count.index)
  route_table_id = aws_route_table.private.id
}

/* dhcp_options */

resource "aws_vpc_dhcp_options" "DHCPOpts" {
  domain_name         = var.domain_name
  domain_name_servers = ["AmazonProvidedDNS"]


  tags = {
    Name = "${terraform.workspace}-${var.product}-${var.service}-dopts"
  }
}

/* dhcp_options_association */

resource "aws_vpc_dhcp_options_association" "DHCPOptionsAssociation" {
  vpc_id          = aws_vpc.vpc.id
  dhcp_options_id = aws_vpc_dhcp_options.DHCPOpts.id
}

```



== variable.tf ==
=================

Create variable and refer them in vpc.tf

```terraform

variable "vpc_cidr" {
  type        = string
  description = "CIDR range for VPC"

}

variable "environment" {
  type        = string
  description = "type of environment"

}


variable "aws_region" {
  type        = string
  description = "Type of region"

}

variable "instance_type" {
  type        = string
  description = "Type of instance"

}


variable "cary_nat_ami" {
  description = "AMI for NAt instance"
  type        = map(string)

  default = {
    us-east-1 = "ami-0022f774911c1d690"
    us-east-2 = "ami-0fa49cc9dc8d62c84"
  }

}

variable "service" {
  type        = string
  default     = "vpc"
  description = "type of service"

}


variable "product" {
  type        = string
  default     = "cary"
  description = "type of product"

}


variable "public_subnets_cidr" {
  type        = list(string)
  description = "CIDR range for Public Subnet"

}



variable "private_subnets_cidr" {
  type        = list(string)
  description = "CIDR range for Private Subnet"

}


variable "key_pair" {
  type        = string
  default     = "cary-key-pair"
  description = "EC2 key pair"

}


variable "domain_name" {
  type        = string
  default     = "clinicacary.com"
  description = "Domain Name for DHCP"

}


variable "sg_ingress_rules" {
  type = list(object({
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_block  = string
    description = string
  }))
  default = [
    # {
    #   from_port   = 0
    #   to_port     = 65535
    #   protocol    = "-1"
    #   cidr_block  = "10.90.0.0/16"
    #   description = "dev"
    # },
    {
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_block  = "0.0.0.0/0"
      description = "Allowed HTTP Traffic for All"
    },
    {
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_block  = "0.0.0.0/0"
      description = "Allowed HTTPS Traffic for All"
    },
    {
      from_port   = 8443
      to_port     = 8443
      protocol    = "tcp"
      cidr_block  = "0.0.0.0/0"
      description = "Allowed Pritunal Traffic for All"
    },
    {
      from_port   = 22
      to_port     = 22
      protocol    = "tcp"
      cidr_block  = "122.160.234.6/32"
      description = "Allowed only for impressico primises"
    }
  ]
}


variable "enable_dns_support" {
  type = bool
}
variable "enable_dns_hostnames" {
  type = bool
}


variable "availability_zones" {
  type = list(string)
}

variable "vpc_id" {

}

```



== output.tf ==
===============

define Output values to return results to the calling module, which it can then use to populate arguments elsewhere

```terraform

/*=> vpc_id <=*/

output "vpc_id" {
  value = aws_vpc.vpc.id

}

/*=> "List of IDs of public subnets" <=*/

output "public_subnets" {
  description = "List of IDs of public subnets"

  value = aws_subnet.public_subnet.*
}

/*=> "List of IDs of public subnets" <=*/

output "private_subnets" {
  description = "List of IDs of public subnets"

  value = aws_subnet.private_subnet.*
}

output "generalSG_id" {
  description = "Security group id"

  value = aws_security_group.generalSG.id
}

output "carybucket" {
  description = "bucket name"

  value = aws_s3_bucket.carybucket.bucket
}

```


Once the resources created in the module, then we can reuse them anywhere and create required AWS 
resources.

== Create resources in Dev environment by calling local module created above ==

Dev environment:
===============
we can reuse moduel created and can creat the resources in Dev environment.
Dev folder structure:

1. provider.tf: Providers allow Terraform to interact with cloud providers like aws
2. main.tf: we call module in this file to create the resources.
3. variable.tf: Variables in Terraform are a great way to define centrally controlled reusable      values. The information in Terraform variables is saved independently from the deployment plans, which makes the values easy to read and edit from a single file
4. output.tf: Terraform output values allow you to export structured data about your resources
5. terraform.tfvars: A terraform.tfvars file is used to set the actual values of the variables. You could set default values for all your variables and not use tfvars files at all.

```terraform

provider.tf
===========

provider "aws" {

  region = var.aws_region

}

main.tf:
========

module "vpc" {
  source = "../../Modules/Networking"

  vpc_cidr             = var.vpc_cidr
  vpc_id               = module.vpc.vpc_id
  environment          = var.environment
  aws_region           = var.aws_region
  instance_type        = var.instance_type
  public_subnets_cidr  = var.public_subnets_cidr
  private_subnets_cidr = var.private_subnets_cidr
  enable_dns_hostnames = var.enable_dns_hostnames
  enable_dns_support   = var.enable_dns_support
  domain_name          = var.domain_name
  availability_zones   = var.availability_zones
  #sg_ingress_rules = var.sg_ingress_rules
}

```

variable.tf:
============

```terraform

variable "vpc_cidr" {
  type        = string
  description = "CIDR range for VPC"

}

variable "environment" {
  type        = string
  description = "type of environment"

}
variable "aws_region" {
  type        = string
  description = "Type of region"

}

variable "instance_type" {
  type        = string
  description = "Type of instance"

}

variable "service" {
  type        = string
  default     = "vpc"
  description = "type of service"

}

variable "product" {
  type        = string
  default     = "cary"
  description = "type of product"

}


variable "public_subnets_cidr" {
  type        = list(string)
  description = "CIDR range for Public Subnet"

}

variable "private_subnets_cidr" {
  type        = list(string)
  description = "CIDR range for Private Subnet"

}

variable "key_pair" {
  type        = string
  default     = "cary-key-pair"
  description = "EC2 key pair"

}

variable "cary_nat_ami" {
  description = "AMI for NAt instance"
  type        = map(string)

  default = {
    us-east-1 = "ami-0022f774911c1d690"
    us-east-2 = "ami-0fa49cc9dc8d62c84"
  }

}

variable "domain_name" {
  type        = string
  default     = "clinicacary.com"
  description = "Domain Name for DHCP"

}

variable "sg_ingress_rules" {
  type = list(object({
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_block  = string
    description = string
  }))
  default = [
    {
      from_port   = 0
      to_port     = 65535
      protocol    = "-1"
      cidr_block  = "10.90.0.0/16"
      description = "dev"
    },
    {
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_block  = "0.0.0.0/0"
      description = "Allowed HTTP Traffic for All"
    },
    {
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_block  = "0.0.0.0/0"
      description = "Allowed HTTPS Traffic for All"
    },
    {
      from_port   = 8443
      to_port     = 8443
      protocol    = "tcp"
      cidr_block  = "0.0.0.0/0"
      description = "Allowed Pritunal Traffic for All"
    },
  ]
}



variable "enable_dns_support" {
  type = bool
}
variable "enable_dns_hostnames" {
  type = bool
}

variable "availability_zones" {
  type = list(string)
}

```

terraform.tf:
============
```terraform

aws_accountnumber="741493839224"
environment = "dev"
aws_region="us-east-1"
instance_type = "t2.micro"
vpc_cidr="10.91.0.0/16"
availability_zones=["us-east-1a", "us-east-1b", "us-east-1c"]
private_subnets_cidr=["10.91.1.0/24", "10.91.2.0/24", "10.91.3.0/24"]
public_subnets_cidr=["10.91.101.0/24", "10.91.102.0/24", "10.91.103.0/24"]

enable_dns_support   = true
enable_dns_hostnames = true

```

output.tf
=========
```terraform

output "cary_vpc_id" {
  description = "ID of project VPC"
  value       = module.vpc.vpc_id
}

output "cary_public_subnets_ids" {
  description = "List of IDs of public subnets"
  value       = module.vpc.public_subnets.*.id
}

output "cary_private_subnets_ids" {
  description = "List of IDs of private subnets"
  value       = module.vpc.private_subnets.*.id
}


output "cary_generalSG_id" {
  description = "secutiry group id"
  value       = module.vpc.generalSG_id
}

output "cary_carybucket_name" {
  description = "Bucket name"
  value       = module.vpc.carybucket
}
```


Execute Terraform Commands:
===========================
once you create all the above terraform files, to create resources, first go to working directory where we have terraform files and run below commands.

working directory:
...Cary-Multiaccount-VPC Module/Environments/Dev

First, create Dev workspace using below command

terraform workspace new dev

# Terraform Initialize

terraform init
Observation:
1. Verify if modules got downloaded to .terraform folder

# Terraform Validate
terraform validate

# Terraform plan
terraform plan

# Terraform Apply
terraform apply -auto-approve
Observation:
1) Verify VPC
2) Verify Subnets
3) Verify IGW
4) Verify Public Route for Public Subnets
5) Verify no public route for private subnets
6) Verify Tags .... etc, verify all the defined resources create successfully.

# Terraform Destroy if you dont need 
terraform destroy -auto-approve

same way we can create resurces in other environments(Shared/Stage) as well.


















