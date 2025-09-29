# K0s Ansible Cluster

Playbook de Ansible para automatizar la creación y configuración de un cluster de Kubernetes usando k0s.

## Tabla de Contenidos

- [Descripción](#descripción)
- [Requisitos del Sistema](#requisitos-del-sistema)
- [Estructura del Cluster](#estructura-del-cluster)
- [Configuración](#configuración)
- [Instalación](#instalación)
- [Uso](#uso)
- [Arquitectura](#arquitectura)
- [Verificación](#verificación)
- [Solución de Problemas](#solución-de-problemas)
- [Referencias](#referencias)

## Descripción

Este proyecto automatiza el despliegue de un cluster de Kubernetes de alta disponibilidad utilizando k0s. El playbook configura automáticamente:

- Nodo maestro principal
- Nodos de plano de control adicionales para alta disponibilidad
- Nodos worker para ejecutar las cargas de trabajo
- Configuración de red y módulos del kernel necesarios
- Balanceador de carga para el plano de control

## Requisitos del Sistema

### Hardware Mínimo

- **Nodo Maestro/Control**: 2 CPU, 2GB RAM, 20GB almacenamiento
- **Nodos Worker**: 1 CPU, 1GB RAM, 10GB almacenamiento
- Red privada entre todos los nodos

### Software

- Sistema operativo basado en RHEL
- Python 3.6+ instalado en el nodo de control
- Acceso SSH a todos los nodos del cluster
- Privilegios sudo en todos los nodos

### Dependencias

- Ansible 2.9+

## Estructura del Cluster

El cluster se compone de tres tipos de nodos:

- **Master**: Nodo maestro principal que inicializa el cluster
- **Control-planes**: Nodos adicionales del plano de control para alta disponibilidad
- **Workers**: Nodos que ejecutan las cargas de trabajo de aplicaciones

## Configuración

### 1. Configurar el Inventario

Edita el archivo `inventory.yaml` con las direcciones IP de tus nodos:

```yaml
all:
  children:
    control-planes:
      hosts:
        control-plane1:
          ansible_host: 192.168.1.101
        control-plane2:
          ansible_host: 192.168.1.102
    master:
      hosts:
        master:
          ansible_host: 192.168.1.100
    workers:
      hosts:
        worker1:
          ansible_host: 192.168.1.111
        worker2:
          ansible_host: 192.168.1.112
        worker3:
          ansible_host: 192.168.1.113
  vars:
    k0s_load_balancer: 192.168.1.200    # IP del balanceador
    k0s_version: "v1.29.1+k0s.0"        # Versión específica de k0s
```

### 2. Configurar Autenticación SSH

Asegúrate de tener acceso SSH sin contraseña a todos los nodos:

```bash
eval "$(ssh-agent -s)"
ssh-add /ruta/a/tu/clave_privada
```

## Instalación

### 1. Clonar el Repositorio

```bash
git clone <url-del-repositorio>
cd k0s-ansible
```

### 2. Verificar Conectividad

```bash
ansible all -i inventory.yaml -m ping --user root
```

## Uso

### Despliegue Completo del Cluster

```bash
ansible-playbook cluster_builder.yaml -i inventory.yaml --user root --ask-become-pass
```

### Despliegue por Etapas

Puedes ejecutar etapas específicas usando tags:

```bash
# Solo preconfiguración
ansible-playbook cluster_builder.yaml -i inventory.yaml --user root --tags preconfig

# Solo configurar el nodo maestro
ansible-playbook cluster_builder.yaml -i inventory.yaml --user root --tags master

# Solo configurar planos de control
ansible-playbook cluster_builder.yaml -i inventory.yaml --user root --tags control-planes

# Solo configurar workers
ansible-playbook cluster_builder.yaml -i inventory.yaml --user root --tags workers
```

## Arquitectura

El playbook realiza las siguientes operaciones:

### Fase de Preconfiguración (todos los nodos)
- Configuración del hostname
- Actualización del archivo `/etc/hosts`
- Desactivación del swap
- Habilitación del forwarding IPv4
- Carga de módulos del kernel necesarios (dm_crypt, br_netfilter)
- Instalación de k0s

### Fase del Nodo Maestro
- Generación de la configuración del cluster
- Inicialización del cluster k0s
- Creación de tokens para control planes y workers

### Fase de Control Planes
- Unión al cluster como nodos de control
- Configuración de alta disponibilidad

### Fase de Workers
- Unión al cluster como nodos worker
- Configuración para ejecutar cargas de trabajo

## Verificación

### Verificar el Estado del Cluster

```bash
# Desde el nodo maestro
k0s kubectl get nodes
k0s kubectl get pods -A
k0s kubectl cluster-info
```

### Obtener el Archivo kubeconfig

```bash
# Desde el nodo maestro
k0s kubeconfig admin > ~/.kube/config
```

## Solución de Problemas

### Problemas Comunes

1. **Error de conectividad SSH**
   - Verificar que las claves SSH estén configuradas correctamente
   - Comprobar que el usuario tenga privilegios sudo

2. **Fallo en la instalación de k0s**
   - Verificar conectividad a internet
   - Comprobar que el sistema cumple los requisitos mínimos

3. **Nodos no se unen al cluster**
   - Verificar que los tokens no hayan expirado
   - Comprobar conectividad de red entre nodos
   - Revisar logs: `journalctl -u k0scontroller` o `journalctl -u k0sworker`

### Logs Útiles

```bash
# Ver logs del controlador
journalctl -u k0scontroller -f

# Ver logs del worker
journalctl -u k0sworker -f

# Estado del servicio
systemctl status k0scontroller
systemctl status k0sworker
```

## Referencias

1. [Documentación oficial de k0s](https://docs.k0sproject.io/)
2. [Instalación Manual Multi-nodo](https://docs.k0sproject.io/stable/k0s-multi-node/)
3. [Opciones de Configuración](https://docs.k0sproject.io/stable/configuration/)
4. [Requisitos del Sistema](https://docs.k0sproject.io/stable/system-requirements/)
5. [Configuración de Red](https://docs.k0sproject.io/stable/networking/)
6. [Dependencias Externas](https://docs.k0sproject.io/stable/external-runtime-deps/)
7. [Documentación de Ansible](https://docs.ansible.com/)
