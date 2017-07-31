Infrastructure as Code (IaC) Terraforming
=====

Recently I've been adopting and implementing an Infrastructure as Code (IaC) philosophy in our shop.  IaC provides ops teams the ability to design, provision, deploy and manage their infrastructures by simply writing code using descriptive or high level languages.  This code can execute tasks such as automatic asset provisioning, app deployment and more importantly configuration management.  

In this blog I'm going to share some of my experiences with IaC using [Terraform](https://www.terraform.io). This blog will cover intermediate usage of terraform though begginers will be able to follow along nicely.

Let's started.

**Note:** The following examples are built using Linux & OSX platforms

### Prerequisites

Before we begin you're going to need the have the following ready to go:

- Amazon Web Services(AWS) Account
- [AWS IAMS Access Key & Access Secret](http://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html)
- [AWS IAM Roles for EC2](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html)
- [AWS S3 Bucket](http://docs.aws.amazon.com/AmazonS3/latest/gsg/CreatingABucket.html) to hold our [terraform state files
- Download & Install [Terraform](https://www.terraform.io/downloads.html) for your platform


### AWS EC2 Roles

AWS Roles & Policies enable you to define and assign privileges to AWS assets such as EC2 Instances so that they have permission to function across other AWS services. For example you can create a role that allows EC2 instances to red/write files to AWS S3.

## Terraform

Terraform enables us to basically access & control our AWS assets via code.  Terraform itself is written in [Go Lang](https://golang.org) but it actually has it's own syntax call [HashiCorp Configuration Language](https://github.com/hashicorp/hcl) and is the syntax I'll be using in this blog.

### Terraform/AWS IAMS Configuration

We must configure Terraform to use the AWS Access Key & Secret that you created earlier. These AWS creds provide Terraform the permission to create and manipulate AWS assets on your behalf.


## Create Project folder & terraform.tf file
Terraform operates within directories and interacts with the terraform related files within the directory. Let's create a new directory for our Terraform project run these commands in a terminal:

```bash
$ mkdir terraform01
$ cd terrform01
```

Now that we're in the terraform01 directory we can create a new Terraform file named `terraform.tf` and open it in your favorite text editor then add these lines of code to it:

```hcl-terraform
terraform {
  backend "s3" {
    bucket = "add your AWs S3 Bucket here ie: foo.bar.com"
    key    = "terraform/terraform01/statefile.tfstate"
    region = "us-east-1"
    encrypt = "true"
    // On OSX you will have to populate these AWS Access & Secret in these terraform init files
    access_key = "add your AWS IAM Access Key"
    secret_key = "add your AWS IAM Secret Key"}
}

// AWS Creds for a specific user
variable "aws_access_key" {
    default = "add your AWS IAM Access Key" 
}
variable "aws_secret_key" {
    default = "add your AWS IAM Access Secret"
}
```

The `terraform.tf` file contains the AWS creds needed to execute tasks on the AWS platform. This file also specifies where your projects state is stored on S3.  Storing the file on S3 provide a safe backup location and also serves as a centralized location for sharing this state file amongst teams.

## Intialize Terraform Project
Now that we've crated the `terraform.tf` we can intialize our terraform project which basically sets the project environment & creates the state file in the location mentioned in the `key` parameter above.  You should see a message that reads `Terraform has been successfully initialized!`

## Terraform main.tf file
We have all the elements in place to start building out our terraform project so lets create a new file named `main.tf` and we will start populating with our project code.

```bash
$ touch maint.tf
```

### AWS Provider Block
The `aws provider` block tells our terraform project which AWS access key and secret to use. Open `main.tf` and add this code to it.

```hcl-terraform
provider "aws" {
  region = "us-east-1"
  access_key = "${var.aws_access_key}"
  secret_key = "${var.aws_secret_key}"

}
```

The `access_key` value `${var.aws_access_key}` points to the `access_key` in our terraform.tf file that we created earlier. Terraform first looks in the terraform.tf file for these values if it exists.


### Terraform var.tf file
Terraform has a native file concept of a variable file which enables you to specify terraform data elements in a file named `var.tf`.  This file will hold the elements and values that we'll be using in our project. Create a new file named `var.tf` and paste this code. Modify the `default` values in these elements to your AWS environment:

```hcl-terraform
variable "ami" {
  description = "AWS Image ID"
  default     = "ami-668f1e70"
}

variable "instance_type"{
  description = "AWS Instance Type"
  default     = "t2.micro"
}

variable "key_pair" {
  description = "AWS EC2 Key Pair Pem"
  default     = "Add your key pair"
}

variable "iam_profile" {
  description = "Name of AWS IAM Profile"
  default = "Add your IAM Profile"
}

variable "security_groups" {
  type        = "list"
  description = "The AWS Security Groups."
  default     = ["Add Security Groups you want to assign to instances"] // List of your AWS VPC security groups
}

variable "subnet_id" {
  description = "AWS Subnet ID"
  default     = "Specify subnet ID to place instance in"
}

```

### AWS Instance User Data
In the AWS EC2 instance creation process you can define bootstrapping tasks to instances.  Let's specify a command that updates & patches the instance when created.  Create a new file named `user_data.sh` and paste the following in the file:

```bash
#!/bin/bash
sudo yum -y update
```

## Final Touches main.tf
You should now have three files in your Terraform project folder:

- main.tf
- terraform.tf
- user_data.sh
- var.tf

Now we can finish defining the `main.tf` file with code that will build out a new AWS EC2 Instance using Terraform.  Open your `main.tf` file and continue populating the file with code:

This element specifies the `user_data` tasks that the instance runs on creation. In this case we're telling Terraform to use the contents of the `user_data.sh` file

```hcl-terraform
data "template_file" "user_data" {
  template = "${file("${path.module}/user_data.sh")}"
}
```

Next we're going to define the AWS EC2 instance element which represents an EC2 instance. Append this code to the `main.tf` file:

```hcl-terraform
resource "aws_instance" "terraform01" {

  count = 1
  ami = "${var.ami}"
  instance_type = "${var.instance_type}"
  subnet_id = "${var.subnet_id}"
  vpc_security_group_ids = "${var.security_groups}"
  key_name = "${var.key_pair}"
  iam_instance_profile = "${var.iam_profile}"
  user_data = "${data.template_file.user_data.rendered}"

  ebs_block_device {
    device_name = "/dev/sda1"
    volume_size = 30
    volume_type = "standard"
    delete_on_termination = true
  }

  tags {
    Name = "terraform01"
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

The instance definition above shows how the contents of the `user_data.sh` and `var.tf` are referenced and utilized by Terraform code.  Some of the `resource "aws_instance"` parameters reference var data elements from the `var.tf` file. For example the `instance_type` parameter value is set to `"${var.instance_type}"` value which is a pointer to the variable element in `var.tf` file

```hcl-terraform
variable "instance_type"{
  description = "AWS Instance Type"
  default     = "t2.micro"
}
```

## Terraform Validate, Plan and Apply

We now have all the key elements in place to create a new AWS EC2 instance using the Terraform application. Run these commands in a terminal (make sure you change directoty into the project folder):

```bash
$ terrform validate
```

This validates your terraform project files to ensure they comply with terraform specs. I suggest you run this command after any changes to your project files.  If you receive any error messages correct them & continue validating until you receive a success message.

### Terraform Plan

Next we'll run this command:

```bash
$ terraform plan
```

This command simulates the execution of your terraform project.  It graphs the elements of related AWS objects and outputs a list of items that will be created, modified or destroyed. The `terraform plan` command give you insight into what to expect without actually creating anything.  It's basically a dry-run.


### Terraform Apply
When all your `terraform validate` and `terraform plan` commands run clean you're ready to execute the apply command that execute your terraform project. In terminal run this command:

```bash
$ terraform apply
```

You will see terraform building out the resources defined in your project into AWS.  Log into the AWS Web Console > EC2 section & you will now see a new EC2 instance initializing named terraform01 in the list.

### Terraform Destroy
Now that we have a running EC2 instance running based on our terraform project you have the ability to destroy these resources with the destroy command:

```bash
$ terraform destroy
```

The destroy command will permanently destroy **ALL** of the AWS objects created in your project so use it with caution.  I would recommend you use the destroy command if you're not going to actually use this newly created instance since you could be charged for active AWS resources that you created.

## Conclusion
This blog is designed to give you a top level view of terraform on how to use it using a simple example.  Terraform is a very powerful tool and I highly recommend tech personnel such as OPs, DevOPS, developers etc... get familiar with Terraform and it's capabilities because from what I've seen Infrastructure as Code is slowly becoming the defacto infrastructure management strategy adopted by organizations.

Stay tuned for followup blogs where I'll expand on this blog & show you how to build out more robust terraform projects.

