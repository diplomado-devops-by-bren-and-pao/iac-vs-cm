# Saludos App — Workshop Module 6.2

Laboratorio práctico de **IaC con Terrafom + CM con Ansible para la Saludos App.

## Objetivo

Provisionar infraestructura reproducible en Azure y AWS usando Terraform y configurar esta con Ansible.

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

La carpeta `configuration/ansible/` contiene dos tipos de laboratorios:

#### Provisionamiento de infrastructura y su configuracion con Ansible

El role ec2_aws agrega al inventario temporal el host que va a configurar

```yaml
- name: Crear infraestructura
  hosts: localhost
  roles:
    - role: ec2_aws

- name: Configuracion comun
  hosts: infrastructure_with_ansible
  become: true
  tasks:
    - name: Instalar paquetes base en Amazon Linux
      ansible.builtin.dnf:
        name:
          - python3
          - git 
          - unzip
        state: present
        update_cache: true
      when: ansible_os_family == "RedHat"
```
#### Despliegue de configuración sobre la infraestructura creada por Terraform

Los archivos .ini declaran los inventarios por nube y por tipo de servicio

```yaml
- name: Configuracion comun
  hosts: all
  become: true
  tasks:
    - name: Instalar paquetes base en Amazon Linux
      ansible.builtin.dnf:
        name:
          - python3
          - git 
          - unzip
        state: present
        update_cache: true
      when: ansible_os_family == "RedHat"
    
    - name: Instalar paquetes base en Debian/Ubuntu
      ansible.builtin.apt:
        name:
          - python3
          - git 
          - unzip
        state: present
        update_cache: true
      when: ansible_os_family == "Debian"

  roles:
    - role: docker

- name: Configurar nginx en los servidores frontend
  hosts: frontend
  become: true
  roles:
    - role: nginx
      backend_host: 10.0.0.32
```

## Prerrequisitos

- Terraform
- Ansible
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
