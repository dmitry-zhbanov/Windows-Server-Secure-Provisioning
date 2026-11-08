# Secure Windows Server Provisioning: SDN Isolation, TPM & Secure Boot

**Rol:** Cloud Infrastructure Engineer / SysAdmin
**Tecnologías:** Proxmox VE 9, Windows Server 2025, SDN (Software Defined Networking), VirtIO, UEFI/OVMF, TPM 2.0.

## 🎯 Objetivo Arquitectónico
Aprovisionamiento de una máquina virtual Windows Server moderna cumpliendo con los estándares de seguridad perimetral y criptográfica exigidos por Microsoft. El entorno se despliega bajo una topología de red aislada mediante microsegmentación en el hipervisor (SDN) para evitar la contaminación de la red de área local (LAN) durante la fase de evaluación.

## 🛠️ Especificación de Hardware Virtual (Criptografía y Rendimiento)
Para garantizar la compatibilidad operativa y el máximo rendimiento (I/O), se aplicó la siguiente matriz de configuración estricta:

- **Arquitectura Base (`q35` + `OVMF UEFI`):** Abandono de BIOS Legacy en favor de una placa base moderna con soporte nativo PCIe.
- **Módulo de Plataforma Segura (vTPM):** Emulación de chip criptográfico, requisito estricto en Windows Server moderno para el cifrado de datos y validación de integridad.
- **Arranque Seguro (Secure Boot):** Inyección automatizada de certificados (`Pre-Enroll keys`) para habilitar el arranque seguro e impedir la ejecución de rootkits.
- **Controladora de Discos (`VirtIO SCSI single`):** Implementación de la controladora más eficiente de Proxmox para reducir la sobrecarga del hipervisor.

## ⚙️ Asignación de Recursos y Tuning del Hipervisor
Se optimizó el consumo de recursos del host físico aplicando políticas de asignación directa y prevención de cuellos de botella:

- **Procesador (CPU Type: `host`):** Se asignaron 2 cores físicos. Al seleccionar el tipo `host`, Windows utiliza directamente las instrucciones del procesador físico subyacente sin capas de emulación intermedias.
- **Memoria RAM Fija (6GB):** Se asignaron 6GB de RAM. Por decisión de diseño para garantizar la estabilidad del sistema, se descartó el uso de *Ballooning Device* (asignación dinámica), fijando la memoria para evitar latencias por reasignación.
- **Almacenamiento (80GB SATA):** Aprovisionamiento de 80GB en disco mecánico, desactivando explícitamente la emulación SSD. Se activó el flag de `Backup` a nivel de disco para empaquetar sistema operativo y configuraciones en volcados automatizados.

## 🚧 Aislamiento Perimetral (Proxmox SDN)
En lugar de conectar el servidor a la interfaz puente genérica de la red local, se implementó una zona de red definida por software para contener el tráfico:

1. **Creación de VNet (`RedNAT`):** Despliegue de una red virtual dentro de una zona aislada (Simple Zone).
2. **Subnetting (10.0.0.0/24):** Asignación de un direccionamiento privado independiente de la red de producción.
3. **Reglas de Enmascaramiento (SNAT):** Activación de NAT a nivel de hipervisor, permitiendo a la máquina virtual salir a Internet de forma enmascarada para actualizaciones sin ser enrutable desde el exterior.

## 🛡️ Despliegue de Sistema y Hardening Inicial
- Tras el despliegue de la imagen ISO con **Experiencia de Escritorio**, se estableció una política de credenciales restrictiva para la cuenta de `Administrador` local (longitud mínima de 12 caracteres, forzando el uso de mayúsculas, minúsculas, números y caracteres especiales).
- **QEMU Guest Agent:** Se activó el agente de comunicación entre hipervisor y sistema operativo invitado. Esta integración permite a Proxmox orquestar apagados seguros ordenados (evitando corrupciones del sistema de archivos NTFS) y leer el estado de la red directamente desde la consola del host.
