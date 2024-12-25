---

### Terraform Exercise: Multi-Environment Deployment with Remote State and Null Resources

#### Scenario:
You are tasked with managing a multi-environment infrastructure for a web application. Each environment (e.g., `dev`, `staging`, `prod`) should use a shared Terraform state stored remotely. You also need to perform specific actions (e.g., running scripts or sending notifications) using **null_resources** and provisioners.

---

#### Requirements:

1. **Remote State Storage**:
   - Store Terraform state files remotely using an S3 bucket and DynamoDB table for state locking.

2. **Null Resources**:
   - Use a `null_resource` to trigger post-deployment actions, such as notifying a webhook.

3. **Dynamic Blocks**:
   - Use dynamic blocks to define security group rules based on environment-specific variables.

4. **Data Sources**:
   - Fetch data from a remote Terraform state to share outputs between environments.

5. **Local Values**:
   - Use locals to simplify and centralize environment-specific configuration.

6. **Provisioners**:
   - Use provisioners in a `null_resource` to execute a local script that logs deployment details.

---

### Step-by-Step Instructions:

#### Step 1: Configure the Remote State
Create a backend configuration file (`backend.tf`) to store Terraform state in an S3 bucket with DynamoDB for locking.

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "env/${terraform.workspace}/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "terraform-lock-table"
  }
}
```

#### Step 2: Define Variables
Create a `variables.tf` file to define environment-specific variables.

```hcl
variable "environment" {
  type    = string
  default = "dev"
}

variable "webhook_url" {
  type = string
}

variable "cidr_blocks" {
  type = list(string)
  default = ["0.0.0.0/0"]
}
```

#### Step 3: Use Locals for Configuration
Define `locals.tf` to centralize configuration.

```hcl
locals {
  env_name = var.environment
  tags = {
    Environment = local.env_name
    ManagedBy   = "Terraform"
  }
}
```

#### Step 4: Use Null Resources and Provisioners
Create a `main.tf` to define resources, including a null resource with a local script provisioner.

```hcl
resource "null_resource" "notify" {
  triggers = {
    environment = local.env_name
    timestamp   = timestamp()
  }

  provisioner "local-exec" {
    command = <<EOT
    echo "Deployment completed for ${local.env_name}" >> deployment.log
    curl -X POST -H "Content-Type: application/json" -d '{"status":"completed", "environment":"${local.env_name}"}' ${var.webhook_url}
    EOT
  }
}
```

#### Step 5: Fetch Remote State Data
Reference outputs from another Terraform workspace (e.g., `shared`) for shared resources.

```hcl
data "terraform_remote_state" "shared" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "env/shared/terraform.tfstate"
    region = "us-west-2"
  }
}

output "shared_vpc_id" {
  value = data.terraform_remote_state.shared.outputs.vpc_id
}
```

#### Step 6: Define Dynamic Blocks
Add a security group resource using dynamic blocks.

```hcl
resource "aws_security_group" "web_sg" {
  name        = "${local.env_name}-web-sg"
  description = "Web security group for ${local.env_name}"
  vpc_id      = data.terraform_remote_state.shared.outputs.vpc_id

  dynamic "ingress" {
    for_each = var.cidr_blocks
    content {
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = [ingress.value]
    }
  }

  tags = local.tags
}
```

---

### Exercise Tasks:

1. **Set up the Remote State**:
   - Create the S3 bucket and DynamoDB table required for remote state.

2. **Deploy Multiple Environments**:
   - Use Terraform workspaces to create `dev`, `staging`, and `prod` environments.
   - Deploy infrastructure using `terraform apply`.

3. **Trigger Post-Deployment Actions**:
   - Verify the `null_resource` executes the post-deployment script and sends a notification.

4. **Test Dynamic Security Group Rules**:
   - Modify `cidr_blocks` for each environment and observe changes.

5. **Share Outputs Between Environments**:
   - Use the shared VPC ID from `shared` workspace to verify inter-environment dependency.

6. **Clean Up**:
   - Destroy resources and confirm the remote state is updated correctly.

---

