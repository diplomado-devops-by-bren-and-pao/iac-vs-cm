# Azure — Terraform Lab

Guía completa de implementación de Saludos App en Azure.

> Los archivos de implementación pueden estar deliberadamente vacíos. El estudiante crea cada archivo y copia el bloque indicado en el paso correspondiente.

## 1. Objetivo

Construir la infraestructura usando:

- **Networking:** módulo propio, sin módulos internos, usando recursos `azurerm_*` directamente.
- **Compute / Database:** módulos publicados / Azure Verified Modules consumidos desde el módulo raíz.

## 2. Arquitectura

```text
ROOT
├── Resource Group
├── networking → módulo propio
│   ├── VNet
│   ├── Subnets
│   ├── NSGs
│   ├── NSG Rules
│   └── Private DNS
├── frontend → AVM VM
├── backend  → AVM VM
└── postgresql → AVM PostgreSQL
```

Red:

```text
10.1.0.0/16
├── GatewaySubnet  10.1.0.0/24
├── Frontend        10.1.1.0/24 → frontend NSG
├── Backend         10.1.2.0/24 → backend NSG
└── Database        10.1.3.0/24 → PostgreSQL delegation
```

## 3. Remote State

```bash
az group create   --name terraform-state-rg   --location EastUS
```

```bash
az storage account create   --name stsaludostate123   --resource-group terraform-state-rg   --location EastUS   --sku Standard_LRS   --kind StorageV2   --min-tls-version TLS1_2   --allow-blob-public-access false
```

```bash
az storage container create   --name tfstate   --account-name stsaludostate123   --auth-mode login
```

Si el Storage Account ya existe, cambia el nombre por uno globalmente único y úsalo también en el backend.

## 4. `backend.tf`

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

## 5. `versions.tf`

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

## 6. `provider.tf`

```hcl
provider "azurerm" {
  features {}
}
```

Autenticación:

```bash
az login
az account show
```

## 7. `variables.tf`

```hcl
variable "project_name" {
  type        = string
  description = "Project prefix used in resource names."
  default     = "saludos"
}

variable "location" {
  type        = string
  description = "Azure region."
  default     = "Central US"
}

variable "admin_username" {
  type    = string
  default = "azureuser"
}

variable "ssh_public_key_path" {
  type        = string
  description = "Path to the SSH public key."
}

variable "db_admin_username" {
  type    = string
  default = "saludosadmin"
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

## 8. Resource Group

`network.tf`:

```hcl
resource "azurerm_resource_group" "main" {
  name     = "${var.project_name}-rg"
  location = var.location
}
```

## 9. Módulo propio de Networking

Estructura:

```text
infrastructure/azure/modules/networking/
├── main.tf
├── variables.tf
└── outputs.tf
```

**El módulo no contiene ningún `module` block.**

### `modules/networking/variables.tf`

```hcl
variable "project_name" {
  type = string
}

variable "location" {
  type = string
}

variable "resource_group_name" {
  type = string
}

variable "vnet_cidr" {
  type = string
}

variable "subnets" {
  type = map(object({
    name             = string
    address_prefixes = list(string)
    delegation       = optional(string)
    nsg              = optional(string)
  }))
}

variable "network_security_groups" {
  type = map(object({
    name = string
  }))
}

variable "network_security_rules" {
  type = map(object({
    nsg_name                   = string
    priority                   = number
    direction                  = string
    access                     = string
    protocol                   = string
    source_port_range          = string
    destination_port_range     = string
    source_address_prefix      = string
    destination_address_prefix = string
  }))
}
```

### `modules/networking/main.tf`

```hcl
resource "azurerm_virtual_network" "main" {
  name                = "${var.project_name}-vnet"
  location            = var.location
  resource_group_name = var.resource_group_name
  address_space       = [var.vnet_cidr]
}

resource "azurerm_subnet" "main" {
  for_each = var.subnets

  name                 = each.value.name
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = each.value.address_prefixes

  dynamic "delegation" {
    for_each = each.value.delegation != null ? [each.value.delegation] : []

    content {
      name = "delegation"

      service_delegation {
        name = delegation.value
        actions = [
          "Microsoft.Network/virtualNetworks/subnets/join/action"
        ]
      }
    }
  }
}

resource "azurerm_network_security_group" "main" {
  for_each = var.network_security_groups

  name                = each.value.name
  location            = var.location
  resource_group_name = var.resource_group_name
}

resource "azurerm_network_security_rule" "main" {
  for_each = var.network_security_rules

  name                        = each.key
  priority                    = each.value.priority
  direction                   = each.value.direction
  access                      = each.value.access
  protocol                    = each.value.protocol
  source_port_range           = each.value.source_port_range
  destination_port_range      = each.value.destination_port_range
  source_address_prefix       = each.value.source_address_prefix
  destination_address_prefix = each.value.destination_address_prefix
  resource_group_name         = var.resource_group_name
  network_security_group_name = azurerm_network_security_group.main[each.value.nsg_name].name
}

resource "azurerm_subnet_network_security_group_association" "main" {
  for_each = {
    for key, subnet in var.subnets :
    key => subnet
    if subnet.nsg != null
  }

  subnet_id = azurerm_subnet.main[each.key].id

  network_security_group_id = (
    azurerm_network_security_group.main[each.value.nsg].id
  )
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
output "vnet_id" {
  value = azurerm_virtual_network.main.id
}

output "subnet_ids" {
  value = {
    for key, subnet in azurerm_subnet.main :
    key => subnet.id
  }
}

output "network_security_group_ids" {
  value = {
    for key, nsg in azurerm_network_security_group.main :
    key => nsg.id
  }
}

output "postgres_private_dns_zone_id" {
  value = azurerm_private_dns_zone.postgres.id
}
```

## 10. Consumir Networking desde el Root

En `network.tf`:

```hcl
module "networking" {
  source = "./modules/networking"

  project_name        = var.project_name
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  vnet_cidr           = "10.1.0.0/16"

  subnets = {
    gateway = {
      name             = "GatewaySubnet"
      address_prefixes = ["10.1.0.0/24"]
      nsg              = null
    }

    frontend = {
      name             = "${var.project_name}-frontend-subnet"
      address_prefixes = ["10.1.1.0/24"]
      nsg              = "frontend"
    }

    backend = {
      name             = "${var.project_name}-backend-subnet"
      address_prefixes = ["10.1.2.0/24"]
      nsg              = "backend"
    }

    database = {
      name             = "${var.project_name}-database-subnet"
      address_prefixes = ["10.1.3.0/24"]
      delegation       = "Microsoft.DBforPostgreSQL/flexibleServers"
      nsg              = null
    }
  }

  network_security_groups = {
    frontend = {
      name = "${var.project_name}-frontend-nsg"
    }

    backend = {
      name = "${var.project_name}-backend-nsg"
    }
  }

  network_security_rules = {
    frontend_http = {
      nsg_name                   = "frontend"
      priority                   = 100
      direction                  = "Inbound"
      access                     = "Allow"
      protocol                   = "Tcp"
      source_port_range          = "*"
      destination_port_range     = "80"
      source_address_prefix      = "*"
      destination_address_prefix = "*"
    }

    frontend_ssh = {
      nsg_name                   = "frontend"
      priority                   = 110
      direction                  = "Inbound"
      access                     = "Allow"
      protocol                   = "Tcp"
      source_port_range          = "*"
      destination_port_range     = "22"
      source_address_prefix      = "*"
      destination_address_prefix = "*"
    }

    backend_api = {
      nsg_name                   = "backend"
      priority                   = 100
      direction                  = "Inbound"
      access                     = "Allow"
      protocol                   = "Tcp"
      source_port_range          = "*"
      destination_port_range     = "8080"
      source_address_prefix      = "10.1.0.0/16"
      destination_address_prefix = "*"
    }

    backend_ssh = {
      nsg_name                   = "backend"
      priority                   = 110
      direction                  = "Inbound"
      access                     = "Allow"
      protocol                   = "Tcp"
      source_port_range          = "*"
      destination_port_range     = "22"
      source_address_prefix      = "*"
      destination_address_prefix = "*"
    }
  }
}
```

La asociación subnet/NSG **no se pasa desde el root**. Se deriva automáticamente del atributo `nsg` de cada subnet.

## 11. Validar Networking

```bash
cd infrastructure/azure
terraform fmt -recursive
terraform init
terraform validate
terraform plan
```

El plan debe incluir:

- 1 VNet.
- 4 subnets.
- 2 NSG.
- 4 reglas.
- 2 asociaciones subnet/NSG.
- 1 Private DNS Zone.
- 1 VNet link.

## 12. Frontend con AVM

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
  zone                = "1"
  encryption_at_host_enabled = false

  source_image_reference = {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  account_credentials = {
    admin_credentials = {
      username                           = var.admin_username
      ssh_keys                           = [file(pathexpand(var.ssh_public_key_path))]
      generate_admin_password_or_ssh_key = false
    }
  }

  network_interfaces = {
    frontend_nic = {
      name = "${var.project_name}-frontend-nic"

      ip_configurations = {
        primary = {
          name                          = "primary"
          private_ip_subnet_resource_id = module.networking.subnet_ids["frontend"]
          create_public_ip_address      = true
          public_ip_address_name        = "${var.project_name}-frontend-pip"
        }
      }
    }
  }

  custom_data = filebase64("../../cloud-init/azure/frontend-docker.yaml")
}
```

## 13. Backend con AVM

```hcl
module "backend" {
  source  = "Azure/avm-res-compute-virtualmachine/azurerm"
  version = "0.21.0"

  name                = "${var.project_name}-backend"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  os_type             = "Linux"
  sku_size            = "Standard_B2s"
  zone                = "1"
  encryption_at_host_enabled = false

  source_image_reference = {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  account_credentials = {
    admin_credentials = {
      username                           = var.admin_username
      ssh_keys                           = [file(pathexpand(var.ssh_public_key_path))]
      generate_admin_password_or_ssh_key = false
    }
  }

  network_interfaces = {
    backend_nic = {
      name = "${var.project_name}-backend-nic"

      ip_configurations = {
        primary = {
          name                          = "primary"
          private_ip_subnet_resource_id = module.networking.subnet_ids["backend"]
          create_public_ip_address      = false
        }
      }
    }
  }
}
```

## 14. PostgreSQL con AVM

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
  server_version         = "16"

  delegated_subnet_id = module.networking.subnet_ids["database"]
  private_dns_zone_id = module.networking.postgres_private_dns_zone_id

  public_network_access_enabled = false

  databases = {
    saludos = {
      name = var.db_name
    }
  }

  sku_name = "B_Standard_B1ms"
  high_availability = null
}
```

## 15. Outputs

`outputs.tf`:

```hcl
output "vnet_id" {
  value = module.networking.vnet_id
}

output "frontend_subnet_id" {
  value = module.networking.subnet_ids["frontend"]
}

output "backend_subnet_id" {
  value = module.networking.subnet_ids["backend"]
}

output "database_subnet_id" {
  value = module.networking.subnet_ids["database"]
}

output "frontend_public_ip" {
  value = try(
    module.frontend.resource_public_ip_addresses[
      "${var.project_name}-frontend-pip"
    ],
    null
  )
}

output "database_private_dns_zone_id" {
  value = module.networking.postgres_private_dns_zone_id
}
```

## 16. `terraform.tfvars`

```hcl
project_name        = "saludos"
location            = "East US"
admin_username      = "azureuser"
ssh_public_key_path = "PATH_KEY_PUBLICA"

db_admin_username = "saludosadmin"
db_admin_password = "REEMPLAZAR_CON_UN_PASSWORD_SEGURO"
db_name           = "saludosdb"
```

## 17. Flujo final

```bash
terraform fmt -recursive
terraform init
terraform validate
terraform plan
terraform apply
terraform output
```

Al terminar:

```bash
terraform destroy
```

## 18. Cloud-Init

`cloud-init/azure/frontend-docker.yaml`:

```yaml
#cloud-config

package_update: true

packages:
  - docker.io

runcmd:
  - systemctl enable docker
  - systemctl start docker
```

El frontend recibe el archivo con:

```hcl
custom_data = filebase64("../../cloud-init/azure/frontend-docker.yaml")
```

La comparación conceptual queda:

```text
Terraform → provisiona la VM
Cloud-Init → configuración inicial
Ansible → Configuration Management posterior
```

