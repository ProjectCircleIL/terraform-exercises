---

### **Exercise Objective**
Create a Terraform project that deploys a local Docker environment for a web application. This will include a custom Terraform module for defining containers, use of functions for dynamic naming, for_each for iterating over services, and various configurations using variables, data sources, and locals.

---

### **Structure of the Exercise**

#### **1. File Structure**
Organize the project as follows:
```
terraform-exercise/
├── main.tf
├── variables.tf
├── outputs.tf
├── locals.tf
├── modules/
│   └── docker_service/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
```

---

### **Step-by-Step Guide**

#### **Step 1: Define Variables**
Create a `variables.tf` in the root directory to define inputs:
```hcl
variable "services" {
  description = "List of services to deploy with their image and ports"
  type        = map(object({
    image = string
    ports = list(number)
  }))
}

variable "environment" {
  description = "Environment name (e.g., dev, staging, prod)"
  type        = string
  default     = "dev"
}

variable "container_prefix" {
  description = "Prefix for container names"
  type        = string
  default     = "myapp"
}
```

#### **Step 2: Use Locals**
Define reusable local values in `locals.tf`:
```hcl
locals {
  formatted_services = {
    for name, config in var.services :
    name => {
      container_name = "${var.container_prefix}-${name}-${var.environment}"
      image          = config.image
      ports          = config.ports
    }
  }
}
```

#### **Step 3: Create the Module**
In the `modules/docker_service` directory:

1. **Module Variables**: Define inputs in `variables.tf`:
   ```hcl
   variable "service_name" {
     description = "Name of the service"
     type        = string
   }

   variable "container_name" {
     description = "Name of the container"
     type        = string
   }

   variable "image" {
     description = "Docker image to use"
     type        = string
   }

   variable "ports" {
     description = "List of ports to expose"
     type        = list(number)
   }
   ```

2. **Main Logic**: In `main.tf`, define the Docker container resource:
   ```hcl
   resource "docker_container" "service" {
     name  = var.container_name
     image = var.image

     dynamic "ports" {
       for_each = var.ports
       content {
         internal = ports.value
         external = ports.value
       }
     }
   }
   ```

3. **Outputs**: Expose module outputs in `outputs.tf`:
   ```hcl
   output "container_name" {
     value = var.container_name
   }
   ```

#### **Step 4: Main Configuration**
In the root directory:

1. **Call the Module**: Use the module in `main.tf`:
   ```hcl
   module "docker_services" {
     source = "./modules/docker_service"

     for_each       = local.formatted_services
     service_name   = each.key
     container_name = each.value.container_name
     image          = each.value.image
     ports          = each.value.ports
   }
   ```

2. **Data Source**: Add a data source to retrieve Docker images locally (for demonstration):
   ```hcl
   data "docker_registry_image" "app_images" {
     for_each = var.services

     name = each.value.image
   }
   ```

3. **Outputs**: Define outputs in `outputs.tf`:
   ```hcl
   output "deployed_containers" {
     value = { for name, container in module.docker_services : name => container.container_name }
   }
   ```

---

### **Step 5: Provide Sample Input**
In the `terraform.tfvars` file, define example inputs:
```hcl
services = {
  app1 = {
    image = "nginx:latest"
    ports = [80, 443]
  }
  app2 = {
    image = "redis:alpine"
    ports = [6379]
  }
}

environment = "dev"
container_prefix = "myapp"
```

---
