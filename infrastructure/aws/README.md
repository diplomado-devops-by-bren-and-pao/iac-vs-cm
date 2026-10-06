# AWS — Terraform Lab

Guía completa de implementación de Saludos App en AWS.

> Los archivos de implementación pueden estar deliberadamente vacíos. El estudiante crea cada archivo y copia el bloque indicado en el paso correspondiente.

## 1. Objetivo

- **Networking:** módulo propio, sin módulos internos, usando recursos `aws_*` directamente.
- **Compute / Database:** módulos publicados consumidos desde el root.
- Uso de `for_each` para colecciones.
- Asociaciones derivadas cuando la relación puede determinarse desde la propia configuración.

## 2. Arquitectura

```text
VPC 10.0.0.0/24
├── Public Subnet
│   └── Frontend EC2
└── Private Subnet
    ├── Backend EC2
    └── RDS PostgreSQL

Internet Gateway
NAT Gateway
Route Tables
Security Groups
```

## 3. Remote State

```bash
aws s3api create-bucket   --bucket saludos-terraform-state-123456   --region us-east-1
```

```bash
aws s3api put-bucket-versioning   --bucket saludos-terraform-state-123456   --versioning-configuration Status=Enabled
```

```bash
aws s3api put-public-access-block   --bucket saludos-terraform-state-123456   --public-access-block-configuration   BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

## 4. `backend.tf`

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

## 5. Versiones y Provider

`versions.tf`:

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

`provider.tf`:

```hcl
provider "aws" {
  region = var.region
}
```

Validación:

```bash
aws sts get-caller-identity
```

## 6. Variables

`variables.tf`:

```hcl
variable "project_name" {
  type    = string
  default = "saludos"
}

variable "region" {
  type    = string
  default = "us-east-1"
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/24"
}

variable "public_subnet_cidr" {
  type    = string
  default = "10.0.0.0/25"
}

variable "private_subnet_cidr" {
  type    = string
  default = "10.0.0.128/25"
}

variable "availability_zone" {
  type    = string
  default = "us-east-1a"
}

variable "db_username" {
  type    = string
  default = "saludosadmin"
}

variable "db_password" {
  type      = string
  sensitive = true
}

variable "db_name" {
  type    = string
  default = "saludosdb"
}

variable "key_name" {
  type = string
}
```

## 7. Módulo propio de Networking

Estructura:

```text
infrastructure/aws/modules/networking/
├── main.tf
├── variables.tf
└── outputs.tf
```

No debe contener ningún `module` block.

## 8. `modules/networking/variables.tf`

```hcl
variable "project_name" {
  type = string
}

variable "region" {
  type = string
}

variable "vpc_cidr" {
  type = string
}

variable "subnets" {
  type = map(object({
    cidr_block        = string
    availability_zone = string
    public            = bool
  }))
}

variable "security_groups" {
  type = map(object({
    description = string
  }))
}

variable "security_group_rules" {
  type = map(object({
    security_group = string
    type           = string
    protocol       = string
    from_port      = number
    to_port        = number
    cidr_blocks    = list(string)
  }))
}
```

## 9. `modules/networking/main.tf`

### VPC

```hcl
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.project_name}-vpc"
  }
}
```

### Subnets con `for_each`

```hcl
resource "aws_subnet" "main" {
  for_each = var.subnets

  vpc_id                  = aws_vpc.main.id
  cidr_block              = each.value.cidr_block
  availability_zone       = each.value.availability_zone
  map_public_ip_on_launch = each.value.public

  tags = {
    Name = "${var.project_name}-${each.key}-subnet"
  }
}
```

### Internet Gateway

```hcl
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.project_name}-igw"
  }
}
```

### Route Tables

```hcl
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "${var.project_name}-public-rt"
  }
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.project_name}-private-rt"
  }
}
```

### NAT Gateway

```hcl
resource "aws_eip" "nat" {
  domain = "vpc"

  tags = {
    Name = "${var.project_name}-nat-eip"
  }
}

resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.main["public"].id

  tags = {
    Name = "${var.project_name}-nat"
  }

  depends_on = [
    aws_internet_gateway.main
  ]
}
```

### Ruta privada hacia NAT

```hcl
resource "aws_route" "private_default" {
  route_table_id         = aws_route_table.private.id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.main.id
}
```

### Asociaciones de route tables

```hcl
resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.main["public"].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  subnet_id      = aws_subnet.main["private"].id
  route_table_id = aws_route_table.private.id
}
```

No se crea una variable adicional de asociaciones porque en este diseño la relación es fija:

```text
public  → public route table
private → private route table
```

### Security Groups con `for_each`

```hcl
resource "aws_security_group" "main" {
  for_each = var.security_groups

  name        = "${var.project_name}-${each.key}-sg"
  description = each.value.description
  vpc_id      = aws_vpc.main.id

  tags = {
    Name = "${var.project_name}-${each.key}-sg"
  }
}
```

### Reglas de ingreso

```hcl
resource "aws_vpc_security_group_ingress_rule" "main" {
  for_each = {
    for key, rule in var.security_group_rules :
    key => rule
    if rule.type == "ingress"
  }

  security_group_id = aws_security_group.main[each.value.security_group].id
  cidr_ipv4         = each.value.cidr_blocks[0]
  from_port         = each.value.from_port
  to_port           = each.value.to_port
  ip_protocol       = each.value.protocol
}
```

### Reglas de salida

```hcl
resource "aws_vpc_security_group_egress_rule" "main" {
  for_each = {
    for key, rule in var.security_group_rules :
    key => rule
    if rule.type == "egress"
  }

  security_group_id = aws_security_group.main[each.value.security_group].id
  cidr_ipv4         = each.value.cidr_blocks[0]
  from_port         = each.value.from_port
  to_port           = each.value.to_port
  ip_protocol       = each.value.protocol
}
```

## 10. Outputs

`modules/networking/outputs.tf`:

```hcl
output "vpc_id" {
  value = aws_vpc.main.id
}

output "subnet_ids" {
  value = {
    for key, subnet in aws_subnet.main :
    key => subnet.id
  }
}

output "security_group_ids" {
  value = {
    for key, sg in aws_security_group.main :
    key => sg.id
  }
}
```

## 11. Consumir Networking desde el Root

`network.tf`:

```hcl
module "networking" {
  source = "./modules/networking"

  project_name = var.project_name
  region       = var.region
  vpc_cidr     = var.vpc_cidr

  subnets = {
    public = {
      cidr_block        = var.public_subnet_cidr
      availability_zone = var.availability_zone
      public            = true
    }

    private = {
      cidr_block        = var.private_subnet_cidr
      availability_zone = var.availability_zone
      public            = false
    }
  }

  security_groups = {
    frontend = {
      description = "Frontend security group"
    }

    backend = {
      description = "Backend security group"
    }

    database = {
      description = "Database security group"
    }
  }

  security_group_rules = {
    frontend_http = {
      security_group = "frontend"
      type           = "ingress"
      protocol       = "tcp"
      from_port      = 80
      to_port        = 80
      cidr_blocks    = ["0.0.0.0/0"]
    }

    frontend_ssh = {
      security_group = "frontend"
      type           = "ingress"
      protocol       = "tcp"
      from_port      = 22
      to_port        = 22
      cidr_blocks    = ["0.0.0.0/0"]
    }

    backend_api = {
      security_group = "backend"
      type           = "ingress"
      protocol       = "tcp"
      from_port      = 8080
      to_port        = 8080
      cidr_blocks    = [var.vpc_cidr]
    }

    backend_ssh = {
      security_group = "backend"
      type           = "ingress"
      protocol       = "tcp"
      from_port      = 22
      to_port        = 22
      cidr_blocks    = [var.vpc_cidr]
    }

    database_postgres = {
      security_group = "database"
      type           = "ingress"
      protocol       = "tcp"
      from_port      = 5432
      to_port        = 5432
      cidr_blocks    = [var.private_subnet_cidr]
    }
  }
}
```

## 12. Validar Networking

```bash
cd infrastructure/aws
terraform fmt -recursive
terraform init
terraform validate
terraform plan
```

## 13. AMI Ubuntu

`data.tf`:

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}
```

## 14. Frontend EC2 con módulo publicado

`compute.tf`:

```hcl
module "frontend" {
  source  = "terraform-aws-modules/ec2-instance/aws"
  version = "~> 6.0"

  name = "${var.project_name}-frontend"

  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  subnet_id = module.networking.subnet_ids["public"]

  vpc_security_group_ids = [
    module.networking.security_group_ids["frontend"]
  ]

  key_name = var.key_name

  tags = {
    Name = "${var.project_name}-frontend"
  }
}
```

## 15. Backend EC2

```hcl
module "backend" {
  source  = "terraform-aws-modules/ec2-instance/aws"
  version = "~> 6.0"

  name = "${var.project_name}-backend"

  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  subnet_id = module.networking.subnet_ids["private"]

  vpc_security_group_ids = [
    module.networking.security_group_ids["backend"]
  ]

  key_name = var.key_name

  tags = {
    Name = "${var.project_name}-backend"
  }
}
```

## 16. PostgreSQL RDS con módulo publicado

`database.tf`:

```hcl
module "postgresql" {
  source  = "terraform-aws-modules/rds/aws"
  version = "~> 6.0"

  identifier = "${var.project_name}-postgres"

  engine               = "postgres"
  engine_version       = "16"
  family               = "postgres16"
  major_engine_version = "16"
  instance_class       = "db.t3.micro"

  allocated_storage = 20

  db_name  = var.db_name
  username = var.db_username
  password = var.db_password
  port     = 5432

  create_db_subnet_group = true

  subnet_ids = [
    module.networking.subnet_ids["private"]
  ]

  vpc_security_group_ids = [
    module.networking.security_group_ids["database"]
  ]

  publicly_accessible = false

  skip_final_snapshot = true

  tags = {
    Name = "${var.project_name}-postgres"
  }
}
```

## 17. Outputs

`outputs.tf`:

```hcl
output "vpc_id" {
  value = module.networking.vpc_id
}

output "public_subnet_id" {
  value = module.networking.subnet_ids["public"]
}

output "private_subnet_id" {
  value = module.networking.subnet_ids["private"]
}

output "frontend_security_group_id" {
  value = module.networking.security_group_ids["frontend"]
}

output "backend_security_group_id" {
  value = module.networking.security_group_ids["backend"]
}

output "database_security_group_id" {
  value = module.networking.security_group_ids["database"]
}

output "frontend_public_ip" {
  value = module.frontend.public_ip
}

output "database_endpoint" {
  value = module.postgresql.db_instance_endpoint
}
```

## 18. `terraform.tfvars`

```hcl
project_name = "saludos"
region       = "us-east-1"

vpc_cidr            = "10.0.0.0/24"
public_subnet_cidr  = "10.0.0.0/25"
private_subnet_cidr = "10.0.0.128/25"

availability_zone = "us-east-1a"

db_username = "saludosadmin"
db_password = "REEMPLAZAR_CON_UN_PASSWORD_SEGURO"
db_name     = "saludosdb"

key_name = "REEMPLAZAR_CON_TU_KEY_PAIR"
```

## 19. Flujo final

```bash
terraform fmt -recursive
terraform init
terraform validate
terraform plan
terraform apply
terraform output
```

Al finalizar:

```bash
terraform destroy
```

## 20. Resultado y comparación

```text
ROOT
│
├── Networking
│   └── módulo propio
│       └── aws_* directamente
│
├── EC2
│   └── módulo publicado
│
└── RDS
    └── módulo publicado
```

El objetivo es que el estudiante pueda explicar por qué el módulo propio de Networking es una implementación directa de recursos, mientras que Compute y Database consumen abstracciones ya publicadas.

Configuration Management se implementará posteriormente sobre la infraestructura provisionada por Terraform.
