### Terraform Hands-On Exercise: State Management and Backend Configuration

This exercise will guide you through key Terraform concepts, including state files, backend configurations, state lock resolution, state file corruption, and handling resource dependency loops. You’ll build and manage a simple infrastructure while exploring these topics.

---

### **Objective**
By the end of this exercise, you will:
1. Understand how Terraform manages state files.
2. Configure a remote backend with S3 and DynamoDB.
3. Resolve issues such as state lock, state file corruption, and dependency loops.

---

### **Prerequisites**
1. Terraform installed on your machine.
2. An AWS account with appropriate permissions.
3. AWS CLI configured with your credentials.

---

### **Exercise Steps**

#### **Step 1: Basic Terraform State File**
1. Create a directory for your project:
   ```bash
   mkdir terraform-exercise
   cd terraform-exercise
   ```

2. Write a simple Terraform configuration (`main.tf`) to create an S3 bucket:
   ```hcl
   provider "aws" {
     region = "us-east-1"
   }

   resource "aws_s3_bucket" "example" {
     bucket = "example-terraform-bucket"
     acl    = "private"
   }
   ```

3. Initialize Terraform and apply the configuration:
   ```bash
   terraform init
   terraform apply
   ```

4. Observe the `terraform.tfstate` file generated in your directory.

---

#### **Step 2: State File Manipulation**
1. Use the `terraform state` commands to inspect and manipulate state:
   ```bash
   terraform state list
   terraform state show aws_s3_bucket.example
   ```

2. Rename the resource in the state file:
   ```bash
   terraform state mv aws_s3_bucket.example aws_s3_bucket.new_example
   ```

3. Verify the changes by listing the state again.

---

#### **Step 3: Backend Configuration with S3 and DynamoDB**
1. Modify `main.tf` to configure a remote backend:
   ```hcl
   terraform {
     backend "s3" {
       bucket         = "my-terraform-backend"
       key            = "state/terraform.tfstate"
       region         = "us-east-1"
       dynamodb_table = "terraform-lock-table"
     }
   }
   ```

2. Create the required S3 bucket and DynamoDB table manually (or using Terraform):
   ```bash
   aws s3api create-bucket --bucket my-terraform-backend --region us-east-1
   aws dynamodb create-table --table-name terraform-lock-table \
     --attribute-definitions AttributeName=LockID,AttributeType=S \
     --key-schema AttributeName=LockID,KeyType=HASH \
     --billing-mode PAY_PER_REQUEST
   ```

3. Reinitialize Terraform to migrate the state to the backend:
   ```bash
   terraform init
   ```

4. Verify the state file is now stored remotely in S3.

---

#### **Step 4: Fixing a State Lock**
1. Simulate a lock by running:
   ```bash
   aws dynamodb put-item --table-name terraform-lock-table \
     --item '{"LockID": {"S": "state/terraform.tfstate"}}'
   ```

2. Try to run a Terraform command to observe the lock error:
   ```bash
   terraform plan
   ```

3. Unlock the state manually:
   ```bash
   terraform force-unlock <LOCK_ID>
   ```

---

#### **Step 5: Fixing State File Corruption**
1. Simulate a corrupted state by modifying the state file in the S3 bucket:
   - Download the state file.
   - Edit it to introduce errors.
   - Re-upload the corrupted file.

2. Run a Terraform command to observe the error:
   ```bash
   terraform plan
   ```

3. Restore the state from a backup:
   ```bash
   aws s3 cp s3://my-terraform-backend/state/terraform.tfstate.backup .
   aws s3 cp ./terraform.tfstate.backup s3://my-terraform-backend/state/terraform.tfstate
   ```

---

#### **Step 6: Fixing Resource Dependency Loops**
1. Modify `main.tf` to introduce a dependency loop:
   ```hcl
   resource "aws_s3_bucket" "example" {
     bucket = "example-terraform-bucket"
     acl    = "private"

     depends_on = [aws_s3_bucket.example]
   }
   ```

2. Run `terraform plan` to observe the dependency loop error.

3. Fix the dependency loop by removing the circular dependency.

---

### **Wrap-Up**
1. Destroy the infrastructure:
   ```bash
   terraform destroy
   ```

2. Clean up resources (S3 bucket, DynamoDB table) manually or with Terraform.

---

### **Deliverables**
- A screenshot of the S3 bucket created.
- A screenshot showing resolution of the state lock.
- A written explanation of how you fixed the corrupted state file and resolved the dependency loop.

