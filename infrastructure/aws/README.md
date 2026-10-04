# AWS — Terraform Lab

Esta guía contiene la implementación completa de la infraestructura AWS del laboratorio. Los archivos Terraform del directorio son deliberadamente vacíos: **crea cada archivo y copia el bloque indicado en el paso correspondiente**.

## Objetivo

Construir la infraestructura de la Saludos App usando dos estrategias de reutilización:

- **Networking:** módulo propio construido recurso por recurso con recursos nativos de AWS.
- **Compute y Database:** consumo directo de módulos publicados desde el módulo raíz.

## Arquitectura

```text
ROOT
├── networking → módulo propio → VPC, subnets, IGW, NAT, routing, SG
├── compute    → módulo publicado → EC2
└── database   → módulo publicado → RDS PostgreSQL
```

# 4. AWS — IMPLEMENTACIÓN COMPLETA

## 4.1 Crear bucket para Remote State

En `us-east-1`:

```bash
aws s3api create-bucket \
  --bucket saludos-terraform-state-123456 \
  --region us-east-1

aws s3api put-bucket-versioning \
  --bucket saludos-terraform-state-123456 \
  --versioning-configuration Status=Enabled

aws s3api put-public-access-block \
  --bucket saludos-terraform-state-123456 \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

El bucket debe tener un nombre globalmente único. Si cambias el nombre, actualízalo también en `backend.tf`.

## 4.2 Crear `backend.tf`

```hcl
terraform {
  backend "s3" {
    bucket       = "saludos-terraform-state-123456"
    key          = "saludos/terraform.tfstate"
    region       = "us-east-1"
    use_lockfile = true
    encrypt      = true
  }
}
```

## 4.3 Crear `versions.tf`

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}
```

## 4.4 Crear `provider.tf`

```hcl
provider "aws" {
  region = var.aws_region
}
```

## 4.5 Crear `variables.tf`

```hcl
variable "project_name" {
  type    = string
  default = "saludos"
}

variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "ubuntu_ami" {
  type        = string
  description = "Ubuntu 22.04 AMI for us-east-1."
}

variable "admin_username" {
  type    = string
  default = "ubuntu"
}

variable "ssh_public_key_path" {
  type = string
}

variable "db_username" {
  type    = string
  default = "saludosadmin"
}

variable "db_password" {
  type      = string
  sensitive = true
}
```

## 4.6 Construir el módulo propio de Networking AWS

### `modules/networking/variables.tf`

```hcl
variable "project_name" { type = string }
variable "vpc_cidr" { type = string }
variable "public_cidr" { type = string }
variable "public_cidr_2" { type = string }
variable "private_cidr" { type = string }
variable "private_cidr_2" { type = string }
variable "availability_zone" { type = string }
variable "availability_zone_2" { type = string }
```

### `modules/networking/main.tf`

> Igual que en Azure: recursos AWS directos.

```hcl
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = { Name = "${var.project_name}-vpc" }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${var.project_name}-igw" }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_cidr
  availability_zone       = var.availability_zone
  map_public_ip_on_launch = true
  tags = { Name = "${var.project_name}-public-a" }
}

resource "aws_subnet" "public_2" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_cidr_2
  availability_zone       = var.availability_zone_2
  map_public_ip_on_launch = true
  tags = { Name = "${var.project_name}-public-b" }
}

resource "aws_subnet" "private" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_cidr
  availability_zone = var.availability_zone
  tags = { Name = "${var.project_name}-private-a" }
}

resource "aws_subnet" "private_2" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_cidr_2
  availability_zone = var.availability_zone_2
  tags = { Name = "${var.project_name}-private-b" }
}

resource "aws_eip" "nat" {
  domain = "vpc"
}

resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public.id
  depends_on    = [aws_internet_gateway.main]
  tags = { Name = "${var.project_name}-nat" }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = { Name = "${var.project_name}-public-rt" }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "public_2" {
  subnet_id      = aws_subnet.public_2.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main.id
  }

  tags = { Name = "${var.project_name}-private-rt" }
}

resource "aws_route_table_association" "private" {
  subnet_id      = aws_subnet.private.id
  route_table_id = aws_route_table.private.id
}

resource "aws_route_table_association" "private_2" {
  subnet_id      = aws_subnet.private_2.id
  route_table_id = aws_route_table.private.id
}

resource "aws_security_group" "frontend" {
  name        = "${var.project_name}-frontend-sg"
  description = "Frontend access"
  vpc_id      = aws_vpc.main.id

  tags = { Name = "${var.project_name}-frontend-sg" }
}

resource "aws_vpc_security_group_ingress_rule" "frontend_http" {
  security_group_id = aws_security_group.frontend.id
  cidr_ipv4         = "0.0.0.0/0"
  from_port         = 80
  to_port           = 80
  ip_protocol       = "tcp"
}

resource "aws_vpc_security_group_ingress_rule" "frontend_ssh" {
  security_group_id = aws_security_group.frontend.id
  cidr_ipv4         = "0.0.0.0/0"
  from_port         = 22
  to_port           = 22
  ip_protocol       = "tcp"
}

resource "aws_vpc_security_group_egress_rule" "frontend_all" {
  security_group_id = aws_security_group.frontend.id
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "-1"
}

resource "aws_security_group" "backend" {
  name        = "${var.project_name}-backend-sg"
  description = "Backend access"
  vpc_id      = aws_vpc.main.id
  tags = { Name = "${var.project_name}-backend-sg" }
}

resource "aws_vpc_security_group_ingress_rule" "backend_api" {
  security_group_id = aws_security_group.backend.id
  cidr_ipv4         = var.vpc_cidr
  from_port         = 8080
  to_port           = 8080
  ip_protocol       = "tcp"
}

resource "aws_vpc_security_group_ingress_rule" "backend_ssh" {
  security_group_id = aws_security_group.backend.id
  cidr_ipv4         = var.vpc_cidr
  from_port         = 22
  to_port           = 22
  ip_protocol       = "tcp"
}

resource "aws_vpc_security_group_egress_rule" "backend_all" {
  security_group_id = aws_security_group.backend.id
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "-1"
}

resource "aws_security_group" "database" {
  name        = "${var.project_name}-database-sg"
  description = "Database access"
  vpc_id      = aws_vpc.main.id
  tags = { Name = "${var.project_name}-database-sg" }
}

resource "aws_vpc_security_group_ingress_rule" "database_postgres" {
  security_group_id            = aws_security_group.database.id
  referenced_security_group_id = aws_security_group.backend.id
  from_port                    = 5432
  to_port                      = 5432
  ip_protocol                  = "tcp"
}

resource "aws_vpc_security_group_egress_rule" "database_all" {
  security_group_id = aws_security_group.database.id
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "-1"
}
```

### `modules/networking/outputs.tf`

```hcl
output "vpc_id" { value = aws_vpc.main.id }
output "public_subnet_ids" { value = [aws_subnet.public.id, aws_subnet.public_2.id] }
output "private_subnet_ids" { value = [aws_subnet.private.id, aws_subnet.private_2.id] }
output "frontend_sg_id" { value = aws_security_group.frontend.id }
output "backend_sg_id" { value = aws_security_group.backend.id }
output "database_sg_id" { value = aws_security_group.database.id }
```

## 4.7 Consumir el módulo propio desde `network.tf`

```hcl
module "networking" {
  source = "./modules/networking"

  project_name       = var.project_name
  vpc_cidr           = "10.0.0.0/24"
  public_cidr        = "10.0.0.0/26"
  public_cidr_2      = "10.0.0.64/26"
  private_cidr       = "10.0.0.128/26"
  private_cidr_2     = "10.0.0.192/26"
  availability_zone  = "${var.aws_region}a"
  availability_zone_2 = "${var.aws_region}b"
}
```

## 4.8 Validar Networking AWS

```bash
cd infrastructure/aws
terraform fmt -recursive
terraform validate
terraform plan
```

## 4.9 Frontend y Backend mediante módulo publicado

`compute.tf`:

```hcl
module "frontend" {
  source  = "terraform-aws-modules/ec2-instance/aws"
  version = "6.4.1"

  name                        = "${var.project_name}-frontend"
  ami                         = var.ubuntu_ami
  instance_type               = "t3.micro"
  subnet_id                   = module.networking.public_subnet_ids[0]
  vpc_security_group_ids     = [module.networking.frontend_sg_id]
  associate_public_ip_address = true
  key_name                    = null
}

module "backend" {
  source  = "terraform-aws-modules/ec2-instance/aws"
  version = "6.4.1"

  name                    = "${var.project_name}-backend"
  ami                     = var.ubuntu_ami
  instance_type           = "t3.micro"
  subnet_id               = module.networking.private_subnet_ids[0]
  vpc_security_group_ids = [module.networking.backend_sg_id]
  associate_public_ip_address = false
}
```

> Para acceso SSH a las EC2 debes configurar una clave de acceso de AWS en tu cuenta/región y asociarla según la configuración que uses. En un laboratorio guiado, también puedes utilizar Session Manager si tu AMI y permisos están preparados para ello.

## 4.10 PostgreSQL mediante módulo publicado

`database.tf`:

```hcl
module "postgresql" {
  source  = "terraform-aws-modules/rds/aws"
  version = "7.2.1"

  identifier = "${var.project_name}-postgres"

  engine               = "postgres"
  engine_version       = "16"
  instance_class       = "db.t3.micro"
  allocated_storage    = 20
  max_allocated_storage = 50

  db_name  = "saludosdb"
  username = var.db_username
  password = var.db_password
  port     = 5432

  create_db_subnet_group = true
  subnet_ids              = module.networking.private_subnet_ids
  publicly_accessible    = false
  vpc_security_group_ids = [module.networking.database_sg_id]

  skip_final_snapshot = true
  deletion_protection = false
}
```

## 4.11 Outputs AWS

`outputs.tf`:

```hcl
output "vpc_id" {
  value = module.networking.vpc_id
}

output "frontend_public_ip" {
  value = module.frontend.public_ip
}

output "frontend_public_dns" {
  value = module.frontend.public_dns
}

output "backend_private_ip" {
  value = module.backend.private_ip
}

output "database_endpoint" {
  value     = module.postgresql.db_instance_endpoint
  sensitive = true
}
```

## 4.12 Variables AWS

Crear `terraform.tfvars`:

```hcl
project_name       = "saludos"
aws_region         = "us-east-1"
ubuntu_ami         = "REEMPLAZAR_CON_AMI_UBUNTU_22_04_DE_TU_REGION"
admin_username     = "ubuntu"
ssh_public_key_path = "~/.ssh/id_rsa.pub"
db_username        = "saludosadmin"
db_password        = "REEMPLAZAR_CON_UN_PASSWORD_SEGURO"
```

## 4.13 Ejecutar Terraform AWS

```bash
terraform init
terraform validate
terraform plan
terraform apply
terraform output
```

---

