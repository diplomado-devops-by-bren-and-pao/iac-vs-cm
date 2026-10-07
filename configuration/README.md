# Configuration Management — Ansible Lab

Guía práctica para trabajar dos escenarios diferentes con Ansible:

1. **Configuration Management sobre infraestructura existente**: las máquinas ya existen y Ansible las administra mediante inventarios.
2. **Provisioning + Configuration Management**: Ansible crea una máquina virtual sobre infraestructura de red ya existente, la agrega dinámicamente al inventario y luego la configura.

> Los archivos de implementación pueden estar deliberadamente incompletos. El objetivo es que el estudiante construya o complete cada archivo siguiendo esta guía. El proyecto resuelto se utiliza como referencia del resultado esperado.

---

## 1. Objetivos

Al finalizar el laboratorio el estudiante podrá:

- instalar y utilizar collections de Ansible;
- trabajar con inventarios separados para AWS y Azure;
- organizar hosts por función, por ejemplo `frontend` y `backend`;
- ejecutar tareas comunes sobre múltiples sistemas operativos;
- reutilizar configuración mediante roles;
- utilizar templates Jinja2 y handlers;
- configurar infraestructura existente mediante inventarios;
- crear infraestructura desde Ansible usando collections de cloud;
- registrar el resultado de un módulo con `register`;
- agregar hosts en tiempo de ejecución mediante `add_host`;
- esperar a que una VM esté disponible antes de configurarla;
- diferenciar las credenciales usadas por el controlador de Ansible de las identidades asignadas a las máquinas virtuales.

---

## 2. Arquitectura del laboratorio

La infraestructura de red ya existe.

```text
AWS
├── VPC
├── Subnet frontend
├── Subnet backend
├── Security Groups
└── EC2 creadas previamente o creadas por Ansible

Azure
├── Resource Group
├── VNet
├── Subnet frontend
├── Subnet backend
├── NSG / NIC
└── VM creadas previamente o creadas por Ansible
```

Configuration Management:

```text
Ansible Controller
        |
        +-- Inventario AWS
        |     +-- frontend
        |     `-- backend
        |
        +-- Inventario Azure
        |     +-- frontend
        |     `-- backend
        |
        +-- Tasks comunes
        |     +-- Python
        |     +-- Git
        |     `-- Unzip
        |
        +-- Role Docker
        `-- Role Nginx
              +-- template nginx.conf.j2
              `-- handler
```

---

## 3. Prerrequisitos

En la máquina que ejecutará Ansible:

- Python 3
- pip
- Ansible
- AWS CLI
- Azure CLI
- Git
- acceso SSH a las máquinas
- una llave SSH para AWS
- una llave SSH para Azure

Instalación básica:

```bash
python3 -m pip install --user ansible
ansible --version
```
---

# Parte A — Autenticación del controlador

## 4. AWS: credenciales para que Ansible pueda crear recursos

Las credenciales usadas por Ansible para llamar la API de AWS pertenecen al **controlador de Ansible**.

Para un laboratorio se puede utilizar un usuario IAM con Access Key y Secret Access Key. No utilices credenciales del usuario root.

Configura un perfil:

```bash
aws configure --profile diplomado
```

Ingresa:

```text
AWS Access Key ID
AWS Secret Access Key
Default region name
Default output format
```

Valida:

```bash
aws sts get-caller-identity --profile diplomado
```

Puedes activar el perfil antes de ejecutar Ansible:

```bash
export AWS_PROFILE=diplomado
```

---

## 5. Azure: credenciales para que Ansible pueda crear recursos

### Opción 1 — Azure CLI para el laboratorio

```bash
az login
az account show
```

Si tienes varias suscripciones:

```bash
az account list --output table
az account set --subscription "<SUBSCRIPTION_ID>"
```

La collection de Azure puede utilizar la sesión del Azure CLI.

### Opción 2 — Service Principal para automatización

En Azure, el equivalente práctico para un proceso externo de automatización es un **Service Principal** con un rol RBAC.

```bash
SUBSCRIPTION_ID=$(az account show --query id -o tsv)

az ad sp create-for-rbac \
  --name ansible-diplomado \
  --role Contributor \
  --scopes /subscriptions/$SUBSCRIPTION_ID
```

Guarda los valores retornados y expórtalos:

```bash
export AZURE_SUBSCRIPTION_ID="<subscription-id>"
export AZURE_CLIENT_ID="<appId>"
export AZURE_SECRET="<password>"
export AZURE_TENANT="<tenant>"
```

> En entornos reales conviene limitar el scope del Service Principal al Resource Group o recursos necesarios, en lugar de toda la suscripción.

### Equivalencia conceptual AWS / Azure

```text
Controlador Ansible
AWS   -> IAM User / perfil / credenciales temporales
Azure -> Azure CLI / Service Principal

Identidad de la VM
AWS   -> IAM Role + Instance Profile
Azure -> Managed Identity
```

Si una VM de Azure debe acceder, por ejemplo, a Key Vault, es preferible asignarle una **Managed Identity** en lugar de almacenar un Client Secret dentro de la VM.

---

# Parte B — Actividad 1: configurar infraestructura existente

## 6. Escenario

En esta actividad Ansible **no crea infraestructura**.

Las máquinas ya existen, por ejemplo porque fueron creadas previamente con Terraform.

```text
Terraform / Portal / otro mecanismo
            |
            v
       VM existentes
            |
            v
     inventory.ini
            |
            v
          Ansible
            |
            v
Configuration Management
```

---

## 7. Inventario AWS

Archivo:

```text
configuration/ansible/aws_inventory.ini
```

Ejemplo:

```ini
[frontend]
frontend1 ansible_host=REEMPLAZAR_IP_PUBLICA ansible_user=ec2-user ansible_private_key_file=./diplomado-key.pem

[backend]
backend ansible_host=REEMPLAZAR_IP_BACKEND ansible_user=ec2-user ansible_private_key_file=./diplomado-key.pem
```

Comprueba el inventario:

```bash
ansible-inventory -i aws_inventory.ini --graph
```

Comprueba conectividad:

```bash
ansible -i aws_inventory.ini all -m ping
```

> Si el backend solo tiene IP privada, el controlador necesitará conectividad a esa red o utilizar el frontend/bastion como `ProxyJump`.

---

## 8. Inventario Azure

Archivo:

```text
configuration/ansible/azure_inventory.ini
```

Ejemplo:

```ini
[frontend]
frontend1 ansible_host=REEMPLAZAR_IP_PUBLICA ansible_user=azureuser ansible_private_key_file=./azure-key.pem

[backend]
backend ansible_host=REEMPLAZAR_IP_BACKEND ansible_user=azureuser ansible_private_key_file=./azure-key.pem
```

Valida:

```bash
ansible-inventory -i azure_inventory.ini --graph
ansible -i azure_inventory.ini all -m ping
```

---

## 9. Playbook principal

Archivo:

```text
configuration/ansible/site.yml
```

El playbook permite mostrar que un archivo puede contener varios **plays**.

### Play 1 — Provisioning opcional

```yaml
- name: Crear infraestructura
  hosts: localhost
  roles:
    - role: ec2_aws
```

Este play se ejecuta en el controlador, no en los servidores remotos.

### Play 2 — Configuración común

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
```

Aquí se demuestra:

- tasks normales;
- facts de Ansible;
- condiciones con `when`;
- diferencias entre familias de sistemas operativos;
- reutilización mediante roles.

### Play 3 — Frontend

```yaml
- name: Configurar nginx en los servidores frontend
  hosts: frontend
  become: true

  roles:
    - role: nginx
      backend_host: REEMPLAZAR_IP_BACKEND
```

---

## 10. Role Nginx

Estructura:

```text
roles/nginx/
├── defaults/
│   `-- main.yml
├── handlers/
│   `-- main.yml
├── tasks/
│   `-- main.yml
`-- templates/
    `-- nginx.conf.j2
```

Defaults:

```yaml
---
nginx_listen_port: 80
backend_port: 5000
```

Tasks:

```yaml
---
- name: Instalar nginx
  ansible.builtin.dnf:
    name: nginx
    state: present
    update_cache: true

- name: Agregar configuracion de nginx
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
  notify: Reiniciar nginx

- name: Asegurar que nginx este iniciado
  ansible.builtin.service:
    name: nginx
    state: started
    enabled: true
```

Handler:

```yaml
---
- name: Reiniciar nginx
  ansible.builtin.service:
    name: nginx
    state: restarted
```

Template:

```nginx
server {
    listen {{ nginx_listen_port }} default_server;
    listen [::]:{{ nginx_listen_port }} default_server;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location /api/ {
        proxy_pass http://{{ backend_host }}:{{ backend_port }}/;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

La combinación template + handler permite demostrar que Nginx solo se reinicia cuando la configuración cambia.

---

## 11. Ejecutar Configuration Management

AWS:

```bash
ansible-playbook -i aws_inventory.ini site.yml
```

Azure:

```bash
ansible-playbook -i azure_inventory.ini site.yml
```

Limitar a frontend:

```bash
ansible-playbook -i aws_inventory.ini site.yml --limit frontend
```

Para comprobar idempotencia, ejecuta el playbook nuevamente y compara `changed` con la primera ejecución.

---

# Parte C — Actividad 2: crear y configurar infraestructura con Ansible

## 12. Objetivo

En este escenario Ansible hace dos trabajos:

```text
PLAY 1
localhost
   |
   +-- crear VM
   +-- register
   +-- add_host
   `-- esperar SSH
          |
          v
PLAY 2
nuevo host
   |
   `-- Configuration Management
```

La red, subnet y Security Group/NSG ya existen.

El propósito no es reemplazar Terraform, sino demostrar que una herramienta de Configuration Management también puede consumir APIs de cloud.

---

# Parte C.1 — AWS

## 13. Variables del role AWS

Archivo:

```text
roles/ec2_aws/defaults/main.yml
```

Ejemplo:

```yaml
---
aws_region: us-east-1
aws_ami_id: REEMPLAZAR_AMI
aws_instance_type: t3.micro
aws_key_name: REEMPLAZAR_KEY_NAME
aws_subnet_id: REEMPLAZAR_SUBNET
aws_security_group_id: REEMPLAZAR_SECURITY_GROUP
```

---

## 14. IAM Role de la instancia

El role puede crear una identidad para la EC2 y asociarla mediante un Instance Profile.

```yaml
- name: Crear IAM role
  amazon.aws.iam_role:
    name: get-secret-role
    assume_role_policy_document: "{{ lookup('file', 'trust-policy.json') }}"
    state: present
  register: iam_role
```

```yaml
- name: Adjuntar policy
  amazon.aws.iam_policy:
    iam_type: role
    iam_name: "{{ iam_role.iam_role.role_name }}"
    policy_name: get-secret-ansible
    policy_json: "{{ lookup('file', 'policy.json') }}"
    state: present
```

```yaml
- name: Crear instance profile
  amazon.aws.iam_instance_profile:
    name: "{{ iam_role.iam_role.role_name }}"
    role: "{{ iam_role.iam_role.role_name }}"
    state: present
  register: instance_profile
```

> Este role pertenece a la EC2. No reemplaza las credenciales con las que el controlador de Ansible llama a AWS.

---

## 15. Crear EC2

```yaml
- name: Crear instancia en AWS
  amazon.aws.ec2_instance:
    region: "{{ aws_region }}"
    name: diplomado_devops_ansible_ec2
    image_id: "{{ aws_ami_id }}"
    instance_type: "{{ aws_instance_type }}"
    key_name: "{{ aws_key_name }}"
    iam_instance_profile: "{{ instance_profile.iam_instance_profile.instance_profile_name }}"
    vpc_subnet_id: "{{ aws_subnet_id }}"
    security_group: "{{ aws_security_group_id }}"
    network:
      assign_public_ip: true
    state: running
    wait: true
    tags:
      Name: diplomado_devops_ansible_ec2
      CreatedBy: ansible
  register: ec2_result
```

`register` guarda el resultado del módulo para utilizarlo posteriormente.

---

## 16. `add_host`

La instancia todavía no estaba en un inventario estático.

Podemos agregarla al inventario **en memoria**:

```yaml
- name: Agregar instancia al inventario
  ansible.builtin.add_host:
    name: ec2_aws
    ansible_host: "{{ ec2_result.instances[0].public_ip_address }}"
    ansible_user: ec2-user
    ansible_ssh_private_key_file: ./diplomado-key.pem
    groups:
      - infrastructure_with_ansible
```

`add_host` no modifica físicamente un archivo `.ini`. El host existe en el inventario únicamente durante la ejecución actual.

---

## 17. Esperar a que la instancia pueda configurarse

`wait: true` espera el estado de la EC2, pero el sistema operativo y SSH pueden tardar unos segundos adicionales.

Una opción recomendada es:

```yaml
- name: Esperar a que Ansible pueda conectarse
  ansible.builtin.wait_for_connection:
    timeout: 300
  delegate_to: ec2_aws
```

Otra alternativa es comprobar primero el puerto 22:

```yaml
- name: Esperar SSH
  ansible.builtin.wait_for:
    host: "{{ ec2_result.instances[0].public_ip_address }}"
    port: 22
    timeout: 300
  delegate_to: localhost
```

---


# 25. Comandos útiles

Ver inventario:

```bash
ansible-inventory -i aws_inventory.ini --graph
```

Ping:

```bash
ansible -i aws_inventory.ini all -m ping
```

Ejecutar AWS:

```bash
ansible-playbook -i aws_inventory.ini site.yml
```

Ejecutar Azure:

```bash
ansible-playbook -i azure_inventory.ini site.yml
```

Limitar ejecución:

```bash
ansible-playbook -i aws_inventory.ini site.yml --limit frontend
```

Más detalle durante troubleshooting:

```bash
ansible-playbook -i aws_inventory.ini site.yml -vvv
```

Comprobar sintaxis:

```bash
ansible-playbook site.yml --syntax-check
```

---

# 26. Resultado esperado

Al terminar el laboratorio el estudiante debería poder explicar:

```text
Inventory
   ↓
Play
   ↓
Tasks / Roles
   ↓
Modules
   ↓
Estado del servidor
```

y también:

```text
Cloud API
   ↓
Ansible cloud module
   ↓
register
   ↓
add_host
   ↓
nuevo host en memoria
   ↓
Configuration Management
```

La intención es comparar claramente:

- **infraestructura existente + inventario + configuración**;
- **creación de infraestructura + descubrimiento del host + configuración**;
- credenciales del controlador frente a identidades de las máquinas;
- responsabilidades que se solapan entre IaC y Configuration Management.
