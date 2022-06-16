# AWS Basic EC2 Instance

Create a basic EC2 instance with Terraform.

Features include:

- Select which subnet to put the instance in 
- `cloud-init` script capabilities including cloud-init variables
- Add SSH key for instance
- Control security groups

## Basic configuration

The following shows how to configure an instance within a VPC. For this example, we:

- Create a default Ubuntu instance that starts an Apache server
- Create a default user called tfaws
- Register an SSH key that we can use to enter the instance within a narrow IP range

```bash

### Variables ###

variable "ip_range" {
  type    = string
  default = "0.0.0.0/0" #### CHANGE 0.0.0.0 TO YOUR IP, e.g. 192.182.0.1/32
}

variable "project_tags" {
  type        = map(string)
  description = "Tags used for aws tutorial"
  default = {
    project = "aws-terraform-test"
  }
}

variable "ssh_key" {
  type = object({
    name  = string
    value = string
  })
  description = "SSH key for aws tutorial"
  default = {
    name  = "dev"
    value = "" #### COPY AND PASTE YOUR PUBLIC SSH KEY HERE
  }
}

variable "anywhere_cidr" {
  type        = string
  default     = "0.0.0.0/0"
  description = "all IPs. Used for internet access"
}

variable "ec2_user" {
  type        = string
  default     = "tfaws"
  description = "Sudo user who we'll create upon initialization"
}

### Modules ###

module "aws_vpc" {
  source  = "wesuuu/aws-vpc/basic"
  version = "0.2.0"

  ssh_ip_range = var.ip_range
  project_tags = var.project_tags
}

module "instance" {
  source = "./instance"

  vpc_id       = module.aws_vpc.vpc_id
  subnet_id    = module.aws_vpc.subnets["public-subnet-1"].id
  project_tags = var.project_tags

  ami_id     = "ami-0ee8244746ec5d6d4"
  public_key = var.ssh_key

  aws_security_group_rules = [
    {
      type        = "ingress"
      description = "TLS from VPC"
      from_port   = 22
      to_port     = 22
      protocol    = "tcp"
      cidr_blocks = [var.ip_range]
    },
    {
      type        = "ingress"
      description = "https"
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = [var.ip_range]
    },
    {
      type        = "ingress"
      description = "http"
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = [var.ip_range]
    },
    {
      type        = "egress"
      description = "connect to internet"
      from_port   = 0
      to_port     = 0
      protocol    = "-1"
      cidr_blocks = ["0.0.0.0/0"]
    }
  ]

  instance_tags = {
    Name = "dev"
  }
  cloud_init_filepath = "./start-instance.yml"
  cloud_init_vars = {
    ec2_user = var.ec2_user
    ssh_key  = var.ssh_key.value
  }
}
```

Here's it's accompanying `cloud-init` script:

```yaml
#cloud-config
repo_update: true

packages:
- apache2

runcmd:
- systemctl start apache2
- systemctl enable apache2

users:
- default
- name: ${ec2_user}
  sudo: ALL=(ALL) NOPASSWD:ALL
  shell: /bin/bash
  ssh_authorized_keys:
  - ${ssh_key}
```