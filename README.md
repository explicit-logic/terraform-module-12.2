# Module 12 - Infrastructure as Code with Terraform

This repository contains a demo project created as part of my **DevOps studies** in the [TechWorld with Nana – DevOps Bootcamp](https://www.techworld-with-nana.com/devops-bootcamp).

**Demo Project:** Modularize Project

**Technologies used:** Terraform, AWS, Docker, Linux, Git

**Project Description:**

- Divide Terraform resources into reusable modules

---

### Prerequisites

Complete this demo project before start:

Automate AWS Infrastructure

https://github.com/explicit-logic/terraform-module-12.1

Create `terraform.tfvars` and set your values

```sh
cp terraform.tfvars.example terraform.tfvars
```

---

### Modules

> Container for multiple resources, used together

Why Modules?

- Organize and group configurations
- Encapsulate into distinct logical components
- Re-use
- Group multiple resources into a logical unit

```
Input variables -> Module -> Output values
```

- **Input variables** — like function arguments
- **Output values** — like function return values

---

### Project structure

```
terraform-module-12.2/
├── main.tf                 # Root: provider, VPC, module calls
├── variables.tf            # Root input variables
├── outputs.tf              # Root outputs (ec2_public_ip)
├── providers.tf            # Terraform + AWS provider version pin
├── terraform.tfvars        # Your values (gitignored)
├── terraform.tfvars.example
├── entry-script.sh         # EC2 user_data bootstrap (Docker + nginx)
└── modules/
    ├── subnet/
    │   ├── main.tf         # aws_subnet, aws_internet_gateway, aws_default_route_table
    │   ├── variables.tf
    │   ├── outputs.tf      # subnet
    │   └── providers.tf
    └── webserver/
        ├── main.tf         # aws_default_security_group, aws_ami, aws_key_pair, aws_instance
        ├── variables.tf
        ├── outputs.tf      # instance
        └── providers.tf
```

---

### Prepare modules directory

- Create `modules` directory and subfolders for modules: `subnet`, `webserver`
- Create tf files inside each module dir: `main.tf`, `outputs.tf`, `variables.tf`, `providers.tf`

Each module's `providers.tf` pins the AWS provider so the module stays self-describing:

```tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "6.41.0"
    }
  }
}
```

---

### Create `subnet` module

- Move `"aws_subnet" "myapp-subnet-1"`, `"aws_internet_gateway" "myapp-igw"`, and `"aws_default_route_table" "main-rtb"` to `modules/subnet/main.tf`
- Replace references to resources that no longer live inside the module with input variables
- Declare those inputs in the module's `variables.tf`

file: `modules/subnet/variables.tf`
```tf
variable subnet_cidr_block {}
variable avail_zone {}
variable env_prefix {}
variable vpc_id {}
variable default_route_table_id {}
```

- Expose the subnet as a module output so the root (and other modules) can reference it

file: `modules/subnet/outputs.tf`
```tf
output "subnet" {
  value = aws_subnet.myapp-subnet-1
}
```

- Wire the module into the root `main.tf` and pass variables

```tf
module "myapp-subnet" {
  source = "./modules/subnet"
  subnet_cidr_block = var.subnet_cidr_block
  avail_zone = var.avail_zone
  env_prefix = var.env_prefix
  vpc_id = aws_vpc.myapp-vpc.id
  default_route_table_id = aws_vpc.myapp-vpc.default_route_table_id
}
```

> The `aws_instance.myapp-server` resource's `subnet_id` will now reference the module's output: `subnet_id = module.myapp-subnet.subnet.id`. This reference will move into the webserver module in the next step.

- Apply terraform configuration

```sh
terraform init
```

![](./images/init-subnet-module.png)

> `terraform init` is required whenever you add or change a `module` block — it installs/links the module source.

```sh
terraform plan
```

![](./images/plan-subnet-module.png)

```sh
terraform apply --auto-approve
```

![](./images/apply-subnet-module.png)

EC2 instance was created and running
![](./images/ec2-instance.png)

---

### Create `webserver` module

- Move `"aws_default_security_group" "default-sg"`, `"aws_ami" "latest-amazon-linux-image"` (data source), `"aws_key_pair" "ssh-key"`, and `"aws_instance" "myapp-server"` to `modules/webserver/main.tf`
- Replace external references with input variables

file: `modules/webserver/variables.tf`
```tf
variable vpc_id {}
variable my_ip {}
variable env_prefix {}
variable image_name {}
variable public_key_location {}
variable instance_type {}
variable subnet_id {}
variable avail_zone {}
```

- Expose the EC2 instance as a module output

file: `modules/webserver/outputs.tf`
```tf
output "instance" {
  value = aws_instance.myapp-server
}
```

- Wire the module into the root `main.tf` and pass variables (including the subnet id from the other module)

```tf
module "myapp-server" {
  source = "./modules/webserver"
  vpc_id = aws_vpc.myapp-vpc.id
  my_ip = var.my_ip
  env_prefix = var.env_prefix
  image_name = var.image_name
  public_key_location = var.public_key_location
  instance_type = var.instance_type
  subnet_id = module.myapp-subnet.subnet.id
  avail_zone = var.avail_zone
}
```

- Surface the public IP at the root so it's visible after `apply`

file: `outputs.tf`
```tf
output "ec2_public_ip" {
  value = module.myapp-server.instance.public_ip
}
```

After modularization, the root `main.tf` is left with only the provider, the VPC, and the two module calls:

```tf
provider "aws" {
  region = "eu-central-1"
}

resource "aws_vpc" "myapp-vpc" {
  cidr_block = var.vpc_cidr_block
  tags = {
    Name: "${var.env_prefix}-vpc"
  }
}

module "myapp-subnet" { ... }
module "myapp-server" { ... }
```

- Apply terraform configuration

```sh
terraform init
```

![](./images/init-webserver-module.png)

```sh
terraform plan
```

![](./images/plan-webserver-module.png)

```sh
terraform apply --auto-approve
```

![](./images/apple-webserver-module.png)

SSH to the instance
```sh
ssh ec2-user@<ec2_public_ip>
# verify the user_data script ran
docker -v
```

![](./images/ssh.png)

The app (nginx in a container) is exposed on port 8080: `http://<ec2_public_ip>:8080`.

---

### Clean up

Destroy all managed resources when finished to avoid AWS charges:

```sh
terraform destroy --auto-approve
```

---

### Key takeaways

- **Modules are the unit of reuse** — group related resources behind a clean input/output interface
- **Outputs are the module's public API** — expose only what callers need (`subnet`, `instance`)
- **Cross-module wiring happens at the root** — one module's output becomes another module's input (`module.myapp-subnet.subnet.id` → `module.myapp-server` as `subnet_id`)
- **Pin provider versions per module** — each module's own `providers.tf` keeps it portable
- **Run `terraform init` after adding or changing a module block** — Terraform needs to resolve the new source
