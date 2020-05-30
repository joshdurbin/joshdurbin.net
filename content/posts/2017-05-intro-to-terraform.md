+++
title = "Intro to terraform, IaC"
date = "2017-05-05"
tags = ["terraform"]
+++

## why IAC?

- deterministic infrastructure
- documentation / clarity
- self-service
- version control

## terraform introduction

- open source, written in GO, leveraged in HCL (similar to JSON)
- declarative -- code describes end state instead of the journey to end state
- limited expressive power, but tools like interpolation exist
- timing and dependency hierarchy are handled by resources definitions
- state: execution compares current state vs. actual state vs. desired state

## core terms

1. providers - AWS , google compute, azure, etc
2. resources - most things
3. data sources - data queried at runtime from provider(s)
4. variables
5. variable interpolation and interpolation functions
6. output variables
7. modules

## providers

```
provider "aws" {
  region = "us-west-2"
}
```

## resources

... likely make up the bulk of your IAC. For AWS they contain things
like: ec2 instances, subnets, routing, rules, s3 buckets, rds instances,
rds clusters, etc... etc...

```
resource "aws_vpc" "prod_vpc" {

  cidr_block = "10.0.0.0/16"

  tags {

    Name = "${var.environment_descriptor}"
    Environment = "${var.environment_descriptor}"
    managed-by-terraform = true
  }
}
```

## data

... provider specific data points that are queried in real time for
use with other resources, aspects of TF

```
data "aws_ami" "most_recent_ubuntu_xenial" {
  most_recent = true

  filter {
    name = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-xenial-16.04-amd64-server-*"]
  }

  filter {
    name = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["099720109477"]
}
```

## variables

```
variable "production_vpc_cidr" {
  default = "10.0.0.0/16"
}

resource "aws_vpc" "prod_vpc" {

  cidr_block = "${var.production_vpc_cidr}" // variable usage

  ...
}
```

## sample infrastructure

Now we'll run through an example terraform architecture meant to establish an environment in AWS with
the following specifications:

- VPC with three networking subnets for an ELB spread across three zones; us-west-2a, us-west-2b, us-west-2c
- Subnet for a web server residing in us-west-2a
- EC2 Instance of type t2.nano running nginx w/ a default install / hello world page
- Data pulled from AWS AMI repos meant to bootstrap the aforementioned EC2 instance

### sample infrastructure; variables

The following block defines a list of CIDRs used for the subnets meant to house the ELB instance, a list of AV
zones that are associated with our subnet instances, another subnet CIDR for the EC2 instance, etc... These are
all used in the later resource definitions.

```terraform
variable "production_public_elb_subnets" {
  description = "A list of subnet CIDRs to be used for the production, public ELB instances"
  type = "list"
  default = ["10.0.0.0/24", "10.0.1.0/24", "10.0.2.0/24"]
}

variable "availability_zones" {
  description = "A list of availablity zones to use for various cloud resources"
  type = "list"
  default = ["us-west-2a", "us-west-2b", "us-west-2c"]
}

variable "production_webserver_subnet" {
  default = "10.0.10.0/24"
}

variable "environment_descriptor" {
  default = "production"
}

variable "production_vpc_cidr" {
  default = "10.0.0.0/16"
}

variable "zero_address_default_route_cidr" {
  default = "0.0.0.0/0"
}
```

### sample infrastructure; data

As mentioned before, data are items pulled or queried at runtime and have strongly defined
resources whose properties are available for use in the remaining infrastructure (resources). Data can also
be pulled from the filesystem or be derived from the rendering of so-called templates.

The following types of data are referenced in the block below:

- data is queried, pulled from AWS' AMI registery for te latest Ubuntu Xneial 16.04 amd64 AMI
- the caller (the access/secret key pair) account ID
- template data, a static template consisting of no variables, used as the user data argument for an EC2 instance

```terraform
data "aws_ami" "most_recent_ubuntu_xenial" {
  most_recent = true

  filter {
    name = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-xenial-16.04-amd64-server-*"]
  }

  filter {
    name = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["099720109477"] # Canonical
}

data "aws_caller_identity" "current_identify" {}

data "template_file" "ec2_startup_script" {

  template = <<EOF
#!/bin/bash -xe
apt-get install nginx-core
EOF
}
```

### sample infrastructure; providers

This is pretty straight forward; but we need to define AWS as a provider within our terraform
codebase. A lot of defaults are assumed here, like key pair/profile, etc...

```
provider "aws" {
  region = "us-west-2"
}
```

### sample infrastructure; networking

Here we have our networking code. In this code we take a subset of the aforementioned variables
and leverage them to produce a:

- VPC
- Internet Gateway
- Subnets for the ELB
- Subnet for the EC2 Instance

Public networking is made available for these networking stacks, hence the position definition
of `map_public_ip_on_launch`, meaning resources launched within these subnets have external IPs
-- also known as dual-homed.

The interesting thing in this block is the interpolation found in the resource `production_public_elb`.
In this block we actually create `n` resources where `n` is the lenght of the variable `production_public_elb_subnets`, which
in this example is 3. All 3 subnets reside in the same VPC but have different CIDRs and different availability zones. In order
to iterate and retrieve this data we rely on the `count` parameter, defined as the lenght of the variable `production_public_elb_subnets`
**and** the assumption that the arrays we'll iterate and retrieve on (based on index) are of the same size, ex:

`cidr_block = "${var.production_public_elb_subnets[count.index]}"` and
`availability_zone = "${var.availability_zones[count.index]}"`.

Full example:

```terraform
resource "aws_vpc" "prod_vpc" {

  cidr_block = "${var.production_vpc_cidr}"

  tags {

    Name = "${var.environment_descriptor}"
    Environment = "${var.environment_descriptor}"
    managed-by-terraform = true
  }
}

resource "aws_internet_gateway" "prod_vpc_internet_gateway" {
  vpc_id = "${aws_vpc.prod_vpc.id}"
}

resource "aws_route" "associate_internet_gateway_with_prod_vpc" {
  route_table_id = "${aws_vpc.prod_vpc.default_route_table_id}"
  gateway_id = "${aws_internet_gateway.prod_vpc_internet_gateway.id}"
  destination_cidr_block = "${var.zero_address_default_route_cidr}"
}

resource "aws_subnet" "production_public_elb" {

  count = "${length(var.production_public_elb_subnets)}"

  vpc_id = "${aws_vpc.prod_vpc.id}"
  cidr_block = "${var.production_public_elb_subnets[count.index]}"
  availability_zone = "${var.availability_zones[count.index]}"

  tags {
    Name = "${var.environment_descriptor}-elb"
    Environment = "${var.environment_descriptor}"
    managed-by-terraform = true
  }
}

resource "aws_subnet" "production_webserver" {

  vpc_id = "${aws_vpc.prod_vpc.id}"
  cidr_block = "${var.production_webserver_subnet}"
  map_public_ip_on_launch = true
  availability_zone = "us-west-2a"

  tags {

    Name = "production-webserver"
    managed-by-terraform = true
  }
}
```

### sample infrastructure; security

This section defines the two security groups for our infrastrucutre, which is two:

1. For the ELB subnets allowing traffic from the outside world in on port 80
2. For the EC2 subnet allowing traffic in only on port 80 from the ELB subnet CIDRs **and**
in from the world on port 22 (SSH)

This section also defines a key pair using the local execution user (you, more than likely) via
your public key at `~/.ssh/id_rsa.pub`.

```terraform
resource "aws_key_pair" "keys_to_the_castles" {
  key_name = "castle_keys"
  public_key = "${file("~/.ssh/id_rsa.pub")}"
}

resource "aws_security_group" "production_elb" {

  name = "production-elb"
  description = "Governs the ELB traffic"
  vpc_id = "${aws_vpc.prod_vpc.id}"

  ingress {
    protocol = "tcp"
    from_port = 80
    to_port = 80
    cidr_blocks = ["${var.zero_address_default_route_cidr}"]
  }

  egress {
    protocol = "-1"
    from_port = 0
    to_port = 0
    cidr_blocks = ["${var.zero_address_default_route_cidr}"]
  }

  tags {
    Name = "production-elb"
    managed-by-terraform = true
  }
}

resource "aws_security_group" "production_webserver" {

  name = "production-webserver"
  description = "Governs the nginx box traffic"
  vpc_id = "${aws_vpc.prod_vpc.id}"

  ingress {
    protocol = "tcp"
    from_port = 80
    to_port = 80
    cidr_blocks = ["${aws_subnet.production_public_elb.*.cidr_block}"]
  }

  ingress {
    protocol = "tcp"
    from_port = 22
    to_port = 22
    cidr_blocks = ["${var.zero_address_default_route_cidr}"]
  }

  egress {
    protocol = "-1"
    from_port = 0
    to_port = 0
    cidr_blocks = ["${var.zero_address_default_route_cidr}"]
  }

  tags {
    Name = "production-webserver"
    managed-by-terraform = true
  }
}
```

### sample infrastructure; app stack

Finally we have the "application stack" consisting of the ELB itself and the EC2 instance.

Multiple subnets were created under the resource key `production_public_elb` and their reference and id
extraction can be seen on the 4th line of the following block. This is very basic interpolation in
Terraform.

The ELB will have instances in each of these subnets and bind to the attached security group, with
health checking enabled and will direct traffic explicitly at the EC2 instance named in
the instances section.

**Note:** *In a real world scenario this would direct traffic to an AutoScaling group, with LaunchConfigurations, etc..., etc...

The EC2 instance is provisioned using the existing SSH key from the security section. Nginx is **not** installed by
default in the AMI we're using to provision this instance, thus we must install it ourselves. This is accomplished via the `userdata` argument
to the instance. `userdata` defines a script to be executed once the instance comes online. In this example, we're using the aforementioned data
defined in a template to populate the user data argument.

In this example, we're expecting a local file, a sibling of the terraform code itself named `bootstrap.sh` to be present. The contents
of this file are:

The full body of the "application stack" is as follows:

```
resource "aws_elb" "production_elb" {

  name = "production-elb"
  subnets = ["${aws_subnet.production_public_elb.*.id}"]
  security_groups = ["${aws_security_group.production_elb.id}"]

  listener {
    instance_port = "80"
    instance_protocol = "http"
    lb_port = "80"
    lb_protocol = "http"
  }

  health_check {
    healthy_threshold = 2
    unhealthy_threshold = 3
    timeout = 2
    target = "HTTP:80/"
    interval = 5
  }

  cross_zone_load_balancing = true
  idle_timeout = 60
  connection_draining = true
  connection_draining_timeout = 300

  instances = ["${aws_instance.production_webserver.id}"]

  tags {
    Name = "production-elb"
    managed-by-terraform = true
  }
}

resource "aws_instance" "production_webserver" {

  ami = "${data.aws_ami.most_recent_ubuntu_xenial.id}"
  instance_type = "t2.nano"
  subnet_id = "${aws_subnet.production_webserver.id}"
  user_data = "${data.template_file.ec2_startup_script}"
  key_name = "${aws_key_pair.keys_to_the_castles.key_name}"
  vpc_security_group_ids = ["${aws_security_group.production_webserver.id}"]

  root_block_device {
    volume_type = "gp2"
    volume_size = 8
  }

  tags {
    Name = "production-webserver"
    managed-by-terraform = true
  }

}
```

Executing all of this will product the ELB, the single EC2 instance w/ nginx with direct
SSH access from the world and HTTP/80 access from the ELBs, which are open the world.

Yay!

Running through this example shows only part of the power of terraform. Once you've got these resources
tracked and executed against the provider (AWS), try making changes to the variables, CIDRS, etc...
and watch what happens. It's magic! (except its not -- a basic resource graph and mapped API calls)

Goooo Terraform!

## tips

- If using InteliJ, use the [HCL Language Support Module](https://plugins.jetbrains.com/plugin/7808-hashicorp-terraform--hcl-language-support)
- Use a consistent tag; like "managedByTerraform = true" across all
resources that support tags. Why? To query for resources managed by TF
in the event of a catastrophic state failure.
- Keep state for environments separate, be sure to encrypt remote state storage
- Wrap TF in scripts, a Makefile, or use 0.9.x which supports locking for
the AWS provider w/ configured DynamoDB tables