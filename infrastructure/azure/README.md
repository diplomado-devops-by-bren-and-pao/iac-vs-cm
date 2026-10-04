# Azure — Terraform Lab

Esta guía contiene la implementación completa de la infraestructura Azure del laboratorio. Los archivos Terraform del directorio son deliberadamente vacíos: **crea cada archivo y copia el bloque indicado en el paso correspondiente**.

## Objetivo

Construir la infraestructura de Saludos App usando dos estrategias de reutilización:

- **Networking:** módulo propio construido recurso por recurso con recursos nativos de Azure.
- **Compute y Database:** consumo directo de módulos publicados/Azure Verified Modules desde el módulo raíz.

## Arquitectura

```text
ROOT
├── networking → módulo propio → VNet, subnets, NSG, Private DNS
├── compute    → módulo publicado/AVM → VMs
└── database   → módulo publicado/AVM → PostgreSQL Flexible Server
```

# 3. AZURE — IMPLEMENTACIÓN COMPLETA

## 3.1 Crear el Resource Group para el Remote State

Antes de ejecutar `terraform init`, crea manualmente el almacenamiento que Terraform utilizará para su state.

```bash
az group create \
  --name terraform-state-rg \
  --location EastUS
```

## 3.2 Crear Storage Account

El nombre del Storage Account debe ser globalmente único. Si `stsaludostate123` ya existe, cambia el nombre y utiliza el mismo nombre en `backend.tf`.

```bash
az storage account create \
  --name stsaludostate123 \
  --resource-group terraform-state-rg \
  --location EastUS \
  --sku Standard_LRS \
  --kind StorageV2 \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false
```

## 3.3 Crear el container del state

```bash
az storage container create \
  --name tfstate \
  --account-name stsaludostate123 \
  --auth-mode login
```

## 3.4 Crear `infrastructure/azure/backend.tf`

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "stsaludostate123"
    container_name       = "tfstate"
    key                  = "saludos/terraform.tfstate"
  }
}
```

## 3.5 Crear `versions.tf`

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }
}
```

## 3.6 Crear `provider.tf`

```hcl
provider "azurerm" {
  features {}
}
```

## 3.7 Crear `variables.tf`

```hcl
variable "project_name" {
  type        = string
  description = "Project prefix used in resource names."
  default     = "saludos"
}

variable "location" {
  type        = string
  description = "Azure region."
  default     = "East US"
}

variable "admin_username" {
  type        = string
  default     = "azureuser"
}

variable "ssh_public_key_path" {
  type        = string
  description = "Path to the SSH public key."
}

variable "db_admin_username" {
  type      = string
  default   = "saludosadmin"
}

variable "db_admin_password" {
  type      = string
  sensitive = true
}

variable "db_name" {
  type    = string
  default = "saludosdb"
}
```

## 3.8 Crear el módulo propio de Networking

### `modules/networking/variables.tf`

```hcl
variable "project_name" { type = string }
variable "location" { type = string }
variable "resource_group_name" { type = string }
variable "vnet_cidr" { type = string }
variable "gateway_subnet_cidr" { type = string }
variable "frontend_subnet_cidr" { type = string }
variable "backend_subnet_cidr" { type = string }
variable "database_subnet_cidr" { type = string }
```

### `modules/networking/main.tf`

> Este es el punto clave del ejercicio: aquí solo existen recursos `azurerm_*`.

```hcl
resource "azurerm_virtual_network" "main" {
  name                = "${var.project_name}-vnet"
  location            = var.location
  resource_group_name = var.resource_group_name
  address_space       = [var.vnet_cidr]
}

resource "azurerm_subnet" "gateway" {
  name                 = "GatewaySubnet"
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = [var.gateway_subnet_cidr]
}

resource "azurerm_subnet" "frontend" {
  name                 = "${var.project_name}-frontend-subnet"
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = [var.frontend_subnet_cidr]
}

resource "azurerm_subnet" "backend" {
  name                 = "${var.project_name}-backend-subnet"
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = [var.backend_subnet_cidr]
}

resource "azurerm_subnet" "database" {
  name                 = "${var.project_name}-database-subnet"
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = [var.database_subnet_cidr]

  delegation {
    name = "postgres-flexible-server"

    service_delegation {
      name = "Microsoft.DBforPostgreSQL/flexibleServers"
      actions = [
        "Microsoft.Network/virtualNetworks/subnets/join/action"
      ]
    }
  }
}

resource "azurerm_network_security_group" "frontend" {
  name                = "${var.project_name}-frontend-nsg"
  location            = var.location
  resource_group_name = var.resource_group_name
}

resource "azurerm_network_security_rule" "frontend_http" {
  name                        = "allow-http"
  priority                    = 100
  direction                   = "Inbound"
  access                      = "Allow"
  protocol                    = "Tcp"
  source_port_range           = "*"
  destination_port_range      = "80"
  source_address_prefix       = "*"
  destination_address_prefix  = "*"
  resource_group_name         = var.resource_group_name
  network_security_group_name = azurerm_network_security_group.frontend.name
}

resource "azurerm_network_security_rule" "frontend_ssh" {
  name                        = "allow-ssh"
  priority                    = 110
  direction                   = "Inbound"
  access                      = "Allow"
  protocol                    = "Tcp"
  source_port_range           = "*"
  destination_port_range      = "22"
  source_address_prefix       = "*"
  destination_address_prefix  = "*"
  resource_group_name         = var.resource_group_name
  network_security_group_name = azurerm_network_security_group.frontend.name
}

resource "azurerm_network_security_group" "backend" {
  name                = "${var.project_name}-backend-nsg"
  location            = var.location
  resource_group_name = var.resource_group_name
}

resource "azurerm_network_security_rule" "backend_api" {
  name                        = "allow-api-from-vnet"
  priority                    = 100
  direction                   = "Inbound"
  access                      = "Allow"
  protocol                    = "Tcp"
  source_port_range           = "*"
  destination_port_range      = "8080"
  source_address_prefix      = var.vnet_cidr
  destination_address_prefix = "*"
  resource_group_name         = var.resource_group_name
  network_security_group_name = azurerm_network_security_group.backend.name
}

resource "azurerm_network_security_rule" "backend_ssh" {
  name                        = "allow-ssh"
  priority                    = 110
  direction                   = "Inbound"
  access                      = "Allow"
  protocol                    = "Tcp"
  source_port_range           = "*"
  destination_port_range      = "22"
  source_address_prefix       = "*"
  destination_address_prefix = "*"
  resource_group_name         = var.resource_group_name
  network_security_group_name = azurerm_network_security_group.backend.name
}

resource "azurerm_subnet_network_security_group_association" "frontend" {
  subnet_id                 = azurerm_subnet.frontend.id
  network_security_group_id = azurerm_network_security_group.frontend.id
}

resource "azurerm_subnet_network_security_group_association" "backend" {
  subnet_id                 = azurerm_subnet.backend.id
  network_security_group_id = azurerm_network_security_group.backend.id
}

resource "azurerm_private_dns_zone" "postgres" {
  name                = "${var.project_name}.postgres.database.azure.com"
  resource_group_name = var.resource_group_name
}

resource "azurerm_private_dns_zone_virtual_network_link" "postgres" {
  name                  = "${var.project_name}-postgres-vnet-link"
  private_dns_zone_name = azurerm_private_dns_zone.postgres.name
  resource_group_name   = var.resource_group_name
  virtual_network_id    = azurerm_virtual_network.main.id
}
```

### `modules/networking/outputs.tf`

```hcl
output "vnet_id" { value = azurerm_virtual_network.main.id }
output "frontend_subnet_id" { value = azurerm_subnet.frontend.id }
output "backend_subnet_id" { value = azurerm_subnet.backend.id }
output "database_subnet_id" { value = azurerm_subnet.database.id }
output "frontend_nsg_id" { value = azurerm_network_security_group.frontend.id }
output "backend_nsg_id" { value = azurerm_network_security_group.backend.id }
output "postgres_private_dns_zone_id" { value = azurerm_private_dns_zone.postgres.id }
```

## 3.9 Llamar al módulo propio desde `network.tf`

```hcl
resource "azurerm_resource_group" "main" {
  name     = "${var.project_name}-rg"
  location = var.location
}

module "networking" {
  source = "./modules/networking"

  project_name           = var.project_name
  location               = var.location
  resource_group_name    = azurerm_resource_group.main.name
  vnet_cidr              = "10.1.0.0/16"
  gateway_subnet_cidr    = "10.1.0.0/24"
  frontend_subnet_cidr   = "10.1.1.0/24"
  backend_subnet_cidr    = "10.1.2.0/24"
  database_subnet_cidr   = "10.1.3.0/24"
}
```

## 3.10 Validar Networking

```bash
cd infrastructure/azure
terraform fmt -recursive
terraform validate
terraform plan
```

Un solo `module "networking"` en el root representa muchos recursos reales.

## 3.11 Crear la VM Frontend con un módulo AVM

`compute.tf`:

```hcl
module "frontend" {
  source  = "Azure/avm-res-compute-virtualmachine/azurerm"
  version = "0.21.0"

  name                = "${var.project_name}-frontend"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  os_type             = "Linux"
  sku_size            = "Standard_B2s"

  source_image_reference = {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  account_credentials = {
    admin_credentials = {
      username                          = var.admin_username
      ssh_keys                          = [file(pathexpand(var.ssh_public_key_path))]
      generate_admin_password_or_ssh_key = false
    }
  }

  network_interfaces = {
    frontend_nic = {
      name = "${var.project_name}-frontend-nic"

      ip_configurations = {
        primary = {
          name                          = "primary"
          private_ip_subnet_resource_id = module.networking.frontend_subnet_id
          create_public_ip_address      = true
          public_ip_address_name        = "${var.project_name}-frontend-pip"
        }
      }
    }
  }

  custom_data = filebase64("../../cloud-init/azure/frontend-docker.yaml")
}

module "backend" {
  source  = "Azure/avm-res-compute-virtualmachine/azurerm"
  version = "0.21.0"

  name                = "${var.project_name}-backend"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  os_type             = "Linux"
  sku_size            = "Standard_B2s"

  source_image_reference = {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  account_credentials = {
    admin_credentials = {
      username                          = var.admin_username
      ssh_keys                          = [file(pathexpand(var.ssh_public_key_path))]
      generate_admin_password_or_ssh_key = false
    }
  }

  network_interfaces = {
    backend_nic = {
      name = "${var.project_name}-backend-nic"

      ip_configurations = {
        primary = {
          name                          = "primary"
          private_ip_subnet_resource_id = module.networking.backend_subnet_id
          create_public_ip_address      = false
        }
      }
    }
  }
}
```

> Cloud-Init se utiliza solo en Frontend.

## 3.12 Crear PostgreSQL Flexible Server con AVM

`database.tf`:

```hcl
module "postgresql" {
  source  = "Azure/avm-res-dbforpostgresql-flexibleserver/azurerm"
  version = "0.2.3"

  name                = "${var.project_name}-postgres"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name

  administrator_login    = var.db_admin_username
  administrator_password = var.db_admin_password
  server_version        = "16"
  delegated_subnet_id   = module.networking.database_subnet_id
  private_dns_zone_id   = module.networking.postgres_private_dns_zone_id
  public_network_access_enabled = false

  databases = {
    saludos = {
      name = var.db_name
    }
  }

  sku_name = "B_Standard_B1ms"
}
```

## 3.13 Outputs Azure

`outputs.tf`:

```hcl
output "vnet_id" {
  value = module.networking.vnet_id
}

output "frontend_public_ip" {
  value = try(module.frontend.resource_public_ip_addresses["${var.project_name}-frontend-pip"], null)
}

output "database_subnet_id" {
  value = module.networking.database_subnet_id
}

output "database_private_dns_zone_id" {
  value = module.networking.postgres_private_dns_zone_id
}
```

## 3.14 Variables de Azure

`terraform.tfvars` (crear copiando el ejemplo):

```hcl
project_name        = "saludos"
location            = "East US"
admin_username      = "azureuser"
ssh_public_key_path = "~/.ssh/id_rsa.pub"
db_admin_username   = "saludosadmin"
db_admin_password   = "REEMPLAZAR_CON_UN_PASSWORD_SEGURO"
db_name             = "saludosdb"
```

## 3.15 Ejecutar Terraform en Azure

```bash
terraform init
terraform validate
terraform plan
terraform apply
terraform output
```

## 3.16 Demostración Cloud-Init

Crear `cloud-init/azure/frontend-docker.yaml`:

```yaml
#cloud-config
package_update: true
packages:
  - docker.io
runcmd:
  - systemctl enable docker
  - systemctl start docker
```

Después de `apply`, conecta a la VM Frontend y comprueba:

```bash
docker --version
systemctl status docker
```

La enseñanza aquí es:

```text
Terraform = crea la VM
Cloud-Init = bootstrap del primer arranque
Ansible = configuración repetible / Configuration Management
```

---

