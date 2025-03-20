
# MultiCloud, DevOps & AI Challenge - Stage 1

 In this first stage, I achieve the following:
 - Used Claude to generate Terraform code
 - Set up an EC2 workstation
 - Installed Terraform on EC2
 - Created an S3 bucket

 ## Step 1: Use claude to generate terraform code to create an S3 bucket
Using the following prompt, generate terraform code to create an s3 bucket:

"Provide Terraform code to create an S3 bucket in AWS with a unique name."

The code provided by claude is similar to the following. Ensure the code is correct and valid by checking the terraform documentation:

```tf
# Configure the AWS Provider
provider "aws" {
  region = "us-east-1" 
}

# Generate a random suffix to ensure bucket name uniqueness
resource "random_string" "bucket_suffix" {
  length  = 8
  special = false
  upper   = false
}

# Create an S3 bucket with a unique name
resource "aws_s3_bucket" "my_bucket" {
  bucket = "my-unique-bucket-${random_string.bucket_suffix.result}"

  tags = {
    Name        = "MyBucket"
    Environment = "Dev"
  }
}

# Configure bucket ownership controls
resource "aws_s3_bucket_ownership_controls" "bucket_ownership" {
  bucket = aws_s3_bucket.my_bucket.id
  rule {
    object_ownership = "BucketOwnerPreferred"
  }
}

# Configure public access block for security
resource "aws_s3_bucket_public_access_block" "public_access" {
  bucket = aws_s3_bucket.my_bucket.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Output the bucket name
output "bucket_name" {
  value = aws_s3_bucket.my_bucket.bucket
}
```
## Step 2 Create IAM Role for EC2
Next, create an IAM role named `EC2Admin` having full administrator access from the AWS management console. Note that permissions should stricter, following the least privilege principle in a production enviromment
![Create Ec2 role](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-1/images/Create%20Ec2%20role.png)

![Create Ec2 role 1](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-1/images/Create%20Ec2%20role%202.png)

![Create Ec2 role 2](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-1/images/Create%20Ec2%20role%203.png)

## Step 3 Launch an EC2 instance
Next, Launch an `t2.micro` EC2 instance named `workstation` with Amazon linux2 ami. Create a security group allowing SSH access form instance connect IP range `18.206.107.24/29`.

![launch ec2 instance 1](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-1/images/launch%20ec2%20instance%201.png)

![launch ec2 instance 2](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-1/images/launch%20ec2%20instance%202.png)

![Ec2 created](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-1/images/Ec2%20created.png)

## Step 4 Connect to EC2 Instance and Install Terraform
Connect to the EC2 instance via instance connect. 


Update system packages and install yum-utils:

```sh
sudo yum update -y
sudo yum install -y yum-utils
```
![workstation update](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-1/images/workstation%20update.png)

Install terraform:

```sh
# Add hashicorp repo
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo

# Install terraform
sudo yum -y install terraform

# Verify terraform version
terraform version

```
![workstation terraform install](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-1/images/workstation%20terraform%20install.png)

Ensure to attach the IAM role to the EC2 instance.
![Attach iam role 1](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-1/images/Attach%20iam%20role%201.png)

![Attach iam role 2](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-1/images/Attach%20iam%20role%202.png)

# Step 5 Apply Terraform Configuration
Create a new directory named `terraform-project` and navigate to it.

```sh
mkdir terraform-project && cd terraform-project
```


Create and open `main.tf` and paste the terraform code initially generated in step 1.

![create terraform project](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-1/images/create%20terraform%20project.png)

![main.tf](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-1/images/main.tf.png)

While in the terraform project folder, Initialize terraform, review the plan and apply the configuration with the following command:

```sh
terraform init
terraform plan
terraform apply
```
![terraform init](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-1/images/terraform%20init.png)

![terraform plan](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-1/images/terraform%20plan.png)

![terraform apply](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-1/images/terraform%20apply.png)

## Step 6 Verify S3 Bucket Creation
Using AWS cli, verify the s3 creation by running the following command:

```
aws s3 ls
```
![aws s3 ls](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-1/images/aws%20s3%20ls.png)

## Step 7 Create DynamoDB tables

Create a file named `dynamodb.tf` within the project folder. Enter the following code to create the Dynamodb tables for cloudmart: `cloudmart_products`, `cloudmart_orders`, `cloudmart_tickets`

```tf
# CloudMart Products Table
resource "aws_dynamodb_table" "cloudmart_products" {
  name           = "cloudmart_products"
  billing_mode   = "PAY_PER_REQUEST"  # On-demand capacity
  hash_key       = "product_id"

  attribute {
    name = "product_id"
    type = "S"
  }

  tags = {
    Name        = "CloudMart Products"
    Environment = "Dev"
    Service     = "CloudMart"
  }
}

# CloudMart Orders Table
resource "aws_dynamodb_table" "cloudmart_orders" {
  name           = "cloudmart_orders"
  billing_mode   = "PAY_PER_REQUEST"  # On-demand capacity
  hash_key       = "order_id"  

  attribute {
    name = "order_id"
    type = "S"
  }

  tags = {
    Name        = "CloudMart Orders"
    Environment = "Dev"
    Service     = "CloudMart"
  }
}

# CloudMart Tickets Table
resource "aws_dynamodb_table" "cloudmart_tickets" {
  name           = "cloudmart_tickets"
  billing_mode   = "PAY_PER_REQUEST"  # On-demand capacity
  hash_key       = "ticket_id"

  attribute {
    name = "ticket_id"
    type = "S"
  }

  tags = {
    Name        = "CloudMart Support Tickets"
    Environment = "Dev"
    Service     = "CloudMart"
  }
}

# Outputs
output "cloudmart_products_table_arn" {
  value = aws_dynamodb_table.cloudmart_products.arn
}

output "cloudmart_orders_table_arn" {
  value = aws_dynamodb_table.cloudmart_orders.arn
}

output "cloudmart_tickets_table_arn" {
  value = aws_dynamodb_table.cloudmart_tickets.arn
}
```

![create dynamodb](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-1/images/create%20dynamodb.png)

![terraform dyna plan](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-1/images/terraform%20dyna%20plan.png)


## Step 8 Verify dynamodb tables creation
Run the following command to verify the existence of the dynamodb tables:

```sh
aws dynamodb list-tables --region us-east-1
```
![verify tables](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-1/images/verify%20tables.png)

