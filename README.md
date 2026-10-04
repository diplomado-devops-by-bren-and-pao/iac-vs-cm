# Saludos App — Workshop Module 6.2

Laboratorio práctico de **Terraform + Configuration Management** para la Saludos App.

## Objetivo

Provisionar infraestructura reproducible en Azure y AWS y dejar preparada la transición hacia Configuration Management.

## Estrategia del laboratorio

La práctica muestra una comparación intencional entre dos formas de reutilización en Terraform:

```text
ROOT
│
├── Networking
│    └── MÓDULO PROPIO
│         └── recursos nativos creados directamente
│
├── Compute
│    └── MÓDULO PUBLICADO / AVM
│
└── Database
     └── MÓDULO PUBLICADO / AVM
```

## Contenido

### Azure

En `infrastructure/azure/README.md` se encuentra la guía completa para construir:

- Remote State con Azure Storage.
- Provider y versiones.
- Módulo propio de Networking con recursos `azurerm_*`.
- VNet y subnets.
- NSG y reglas.
- Private DNS para PostgreSQL.
- VMs mediante módulos publicados/AVM.
- PostgreSQL Flexible Server mediante módulo publicado/AVM.
- Validación con `terraform init`, `validate`, `plan`, `apply`, `output` y `destroy`.
- Demostración de Cloud-Init únicamente sobre la VM Frontend.

### AWS

En `infrastructure/aws/README.md` se encuentra la guía completa para construir:

- Remote State con Amazon S3.
- Provider y versiones.
- Módulo propio de Networking con recursos AWS directos.
- VPC, subnets públicas/privadas, Internet Gateway y NAT Gateway.
- Route tables y asociaciones.
- Security Groups.
- EC2 mediante módulo publicado.
- RDS PostgreSQL mediante módulo publicado.
- Validación y destrucción de la infraestructura.

### Configuration Management

La carpeta `configuration/ansible/` queda preparada como estructura para la segunda parte del módulo.

## Prerrequisitos

- Terraform
- AWS CLI
- Azure CLI
- Git
- Cuenta de AWS y/o Azure

## Flujo general

```text
Terraform
   ↓
Provisioning
   ↓
Infraestructura cloud
   ↓
Cloud-Init (demo de bootstrap inicial en Azure)
   ↓
Configuration Management con Ansible
```

## Orden recomendado

1. Abrir `infrastructure/azure/README.md` y completar Azure.
2. Abrir `infrastructure/aws/README.md` y completar AWS.
3. Continuar con `configuration/ansible/` y completar Configuration Management.

> **Importante:** los archivos de código del repositorio se mantienen vacíos intencionalmente. El objetivo del laboratorio es que el estudiante construya cada archivo siguiendo y copiando los bloques de código de su README correspondiente.
