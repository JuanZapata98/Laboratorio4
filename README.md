# Laboratorio 4
# Punto 1
# Conexión por consola a un switch Cisco 2960

## 1. Introducción  
Este documento describe el procedimiento para conectar por consola un switch Cisco Catalyst 2960 desde Ubuntu. Incluye instalación de herramientas útiles, identificación del puerto, conexión mediante `screen`, comandos de configuración básicos y verificación.

## 2. Herramientas útiles (instalación en Ubuntu)  
Antes de conectar por consola, instalar en Ubuntu:

```bash
sudo apt update  
sudo apt install usbutils screen -y  
# Opcionales:  
sudo apt install picocom minicom -y
```

Comandos para verificar el adaptador y el dispositivo serie:

```bash
sudo dmesg | grep tty  
ls /dev/ttyUSB* /dev/ttyACM* 2>/dev/null  
lsusb
```

> `usbutils` aporta `lsusb`, útil para identificar el adaptador. `screen` permite abrir la sesión serie.

## 3. Material y software  
- Switch: Cisco Catalyst 2960 (nombre de host: `sw-lab`)  
- Cable de consola: RJ45 al switch → adaptador USB al equipo Ubuntu  
- Dispositivo1: ordenador cliente conectado al puerto 1  
- Dispositivo2: Raspberry Pi 3, conectado al puerto 2  
- Dispositivo3: otro computador, conectado al puerto 3  
- Ubuntu como estación de administración  
- Privilegios sudo para ejecutar comandos de sistema

## 4. Procedimiento  

### 4.1 Conexión física  
1. Conecta RJ45 al puerto **CONSOLE** del switch.  
2. Conecta el adaptador USB al equipo Ubuntu.  
3. Enciende el switch (si no estuviera encendido).

### 4.2 Identificar el puerto serie en Ubuntu  
Ejecutar:

```bash
sudo dmesg | grep tty  
ls /dev/ttyUSB* /dev/ttyACM* 2>/dev/null
```

Se espera ver algo como `/dev/ttyUSB0`. Si el comando `dmesg` da error de permisos, usar `sudo`.

### 4.3 Abrir sesión por consola usando `screen`  
Con el puerto identificado:

```bash
sudo screen /dev/ttyUSB0 9600
```

Parámetros del puerto: 9600 baudios, 8 bits, sin paridad, 1 bit de parada.  
Para salir de `screen`: `Ctrl + A`, luego `K`, confirmar con `Y`.

### 4.4 Interacción inicial con el switch  
Al conectar se debe ver el prompt:

```
sw-lab>
```

Para pasar a modo privilegiado y luego al modo de configuración:

```
sw-lab> enable  
sw-lab# configure terminal  
sw-lab(config)#  
```

### 4.5 Bloque de configuración básica  
Ejemplo de comandos para administración y preparar puertos:

```
no ip domain-lookup  
enable secret TuPasswordEnable  
username admin privilege 15 secret TuPasswordAdmin  

line console 0  
 exec-timeout 5 0  
 password consolaPass  
 login  
 exit  

line vty 0 4  
 login local  
 transport input ssh  
 exit  

interface vlan 1  
 ip address 192.168.1.2 255.255.255.0  
 no shutdown  
exit  

interface range GigabitEthernet1/0/1 - 3  
 description Dispositivo1_Dispositivo2_Dispositivo3  
 switchport mode access  
 switchport access vlan 1  
 spanning-tree portfast  
 exit  

copy running-config startup-config
```

### 4.6 Verificación y diagnóstico  
Comandos útiles desde el switch:

```
show interfaces status  
show mac address-table dynamic  
show vlan brief  
show ip interface brief  
ping 192.168.1.10   # ejemplo: probar conectividad con Dispositivo1
```

### 4.7 Configuración de la Raspberry Pi 3 (Dispositivo2)  
Pasos comunes para preparar la Raspberry Pi:

- Habilitar SSH:

```bash
sudo raspi-config   # Interfacing Options → SSH → Enable
sudo systemctl enable ssh
sudo systemctl start ssh
```

- Asignar IP estática (editar `/etc/dhcpcd.conf`):

```bash
sudo nano /etc/dhcpcd.conf
```

Añadir al final:

```
interface eth0
static ip_address=192.168.1.11/24
static routers=192.168.1.1
static domain_name_servers=192.168.1.1 8.8.8.8
```

Reiniciar la interfaz o la Raspberry:

```bash
sudo systemctl restart dhcpcd
sudo reboot
```

- Verificar conectividad:

```bash
Ping 192.168.1.11
ssh pi@192.168.1.11
```

Usar `show mac address-table` en el switch para confirmar el puerto asociado.

### 4.8 Actividades prácticas — Paso a paso
A continuación los pasos concretos, listos para ejecutar.

#### Actividad 1 — Probar ping entre dispositivos
1. Desde el switch (modo privilegiado):
```text
sw-lab# ping 192.168.1.10   # Dispositivo1
sw-lab# ping 192.168.1.11   # Dispositivo2 (Raspberry Pi)
sw-lab# ping 192.168.1.12   # Dispositivo3
```
2. Desde Dispositivo1/Dispositivo3/Raspberry (en terminal):
```bash
ping -c 4 192.168.1.2    # ping al switch
ping -c 4 192.168.1.11   # ping entre hosts
```
3. Si no responde, comprobar `show interfaces status` y `show mac address-table dynamic` en el switch, y `ip addr` en el host.

#### Actividad 2 — Explorar con nmap
1. Escaneo de hosts activos (ping sweep):
```bash
sudo nmap -sn 192.168.1.0/24
```
2. Identificar servicios en un host específico (ej. Raspberry):
```bash
sudo nmap -sV 192.168.1.11
```
3. Escaneo rápido de puertos comunes en la red:
```bash
sudo nmap --top-ports 20 192.168.1.0/24
```

#### Actividad 3 — Revisar IP, puerta de enlace, máscara, VLAN y CIDR
1. En hosts Linux (Ubuntu / Raspberry):
```bash
ip addr show
ip route show
```
2. En el switch, revisar VLAN y puerto:
```text
sw-lab# show vlan brief
sw-lab# show interfaces GigabitEthernet1/0/2 switchport
```
3. Interpretación rápida: si el host muestra `inet 192.168.1.11/24`, su red es `192.168.1.0/24`, máscara `255.255.255.0` y la puerta de enlace suele ser `192.168.1.1`.

#### Actividad 4 — Transferir archivos con `scp`
1. Asegurar que SSH está activo en el destino (Dispositivo3 o Raspberry):
```bash
sudo systemctl status ssh
```
2. Enviar archivo desde el monitor (Ubuntu) al destino:
```bash
scp /ruta/local/archivo.txt usuario@192.168.1.12:/ruta/remota/
# Ejemplo:
scp ~/prueba.txt pi@192.168.1.11:/home/pi/
```
3. Verificar en destino:
```bash
ssh usuario@192.168.1.12 'ls -l /ruta/remota/archivo.txt && md5sum /ruta/remota/archivo.txt'
md5sum ~/prueba.txt  # en origen
```
4. Solución de problemas comunes: comprobar `ping`, `nmap -p 22`, y estado del servicio SSH; revisar permisos y rutas.


## 5. Problemas encontrados  
- Lectura de `dmesg` sin permisos → usar `sudo`.  
- Pantalla negra al abrir consola → presionar `Enter`.  
- Ping con tasa de éxito 0% → verificar VLAN, enlace físico, IP, máscara o firewall.

## 6. Conclusiones y recomendaciones  
1. Verificar siempre el dispositivo serie en Ubuntu con permisos adecuados.  
2. Usar parámetros correctos (9600 8N1) para la consola del switch.  
3. Confirmar estado físico y tabla MAC antes de diagnosticar IP.  
4. Guardar configuración con `copy running-config startup-config`.

## 7. Apéndice: Bloque de comandos completo  
```
# En Ubuntu: instalar herramientas útiles
sudo apt update
sudo apt install usbutils screen nmap -y
# (opcional) utilidades alternativas para consolas
sudo apt install picocom minicom -y

# Identificación del adaptador USB/consola
sudo dmesg | grep tty
ls /dev/ttyUSB* /dev/ttyACM* 2>/dev/null
lsusb

# Escaneo rápido en la red
sudo nmap -sn 192.168.1.0/24

# Abrir sesión consola con screen
sudo screen /dev/ttyUSB0 9600

# Raspberry Pi 3: habilitar SSH y asignar IP estática
sudo raspi-config
sudo systemctl enable ssh
sudo systemctl start ssh
# Configurar IP estática en /etc/dhcpcd.conf y reiniciar

# En el switch
enable
configure terminal
no ip domain-lookup
enable secret TuPasswordEnable
username admin privilege 15 secret TuPasswordAdmin

line console 0
 exec-timeout 5 0
 password consolaPass
 login
 exit

line vty 0 4
 login local
 transport input ssh
 exit

interface vlan 1
 ip address 192.168.1.2 255.255.255.0
 no shutdown
 exit

interface range GigabitEthernet1/0/1 - 3
 description Dispositivo1_Dispositivo2_Dispositivo3
 switchport mode access
 switchport access vlan 1
 spanning-tree portfast
 exit

copy running-config startup-config

show interfaces status
show mac address-table dynamic
show vlan brief
show ip interface brief
ping 192.168.1.10
```

# Punto 2
## 1. Introducción

QEMU (Quick Emulator) es una herramienta de virtualización y emulación de hardware ampliamente utilizada en entornos académicos, de investigación y administración de sistemas. Ubuntu 24.04.3 LTS incorpora QEMU versión 8.2.2, la cual ofrece compatibilidad con arquitecturas múltiples y soporte para aceleración mediante KVM.
Este documento describe el proceso completo para habilitar los repositorios correctos, instalar QEMU y crear máquinas virtuales para distintos sistemas operativos.

## 2. Activación de repositorios en Ubuntu 24.04

Ubuntu 24.04 reemplaza el archivo clásico /etc/apt/sources.list por el archivo moderno:

```
/etc/apt/sources.list.d/ubuntu.sources
```

Fue necesario habilitar los componentes donde reside QEMU. Se editó el archivo para asegurar que la línea Components incluyera:
```makefile
Components: main universe multiverse restricted
```

Posteriormente se actualizó la lista de paquetes con:
```bash
sudo apt update
```
## 3. Instalación de QEMU, KVM y herramientas adicionales

Se instalaron los paquetes principales mediante:
```bash
sudo apt install qemu-system qemu-utils qemu-kvm virt-manager
```

La instalación fue exitosa y el sistema confirmó que se utilizaba la versión:
```
QEMU emulator version 8.2.2
```

Opcionalmente se recomendó verificar módulos del kernel:
```bash
lsmod | grep kvm
```

Y añadir al usuario al grupo libvirt si se pretende utilizar herramientas gráficas:
```bash
sudo usermod -aG libvirt $USER
```
## 4. Preparación del entorno para máquinas virtuales

Se creó un directorio dedicado para las máquinas:
```bash
mkdir -p ~/vms
cd ~/vms
```

Se definió una plantilla base aplicable a distintos sistemas operativos:
```bash
qemu-system-x86_64 \
  -enable-kvm \
  -m 2048 \
  -smp 2 \
  -cpu host \
  -hda DISCO.qcow2 \
  -cdrom ISO_DEL_SISTEMA.iso \
  -boot d
```

Los parámetros permiten asignar memoria (RAM), núcleos de CPU, disco en formato QCOW2 y una imagen ISO de instalación.

## 5. Creación de máquinas virtuales específicas
### 5.1 Ubuntu
```bash
qemu-img create -f qcow2 ubuntu.qcow2 20G

qemu-system-x86_64 \
  -enable-kvm \
  -m 2048 \
  -smp 2 \
  -cpu host \
  -hda ubuntu.qcow2 \
  -cdrom ubuntu-24.04-live-server-amd64.iso \
  -boot d
```
### 5.2 CentOS (o derivados como Rocky/AlmaLinux)
```bash
qemu-img create -f qcow2 centos.qcow2 20G

qemu-system-x86_64 \
  -enable-kvm \
  -m 2048 \
  -smp 2 \
  -cpu host \
  -hda centos.qcow2 \
  -cdrom CentOS-Stream-9-latest-x86_64-dvd1.iso \
  -boot d
```
### 5.3 Alpine Linux
```bash
qemu-img create -f qcow2 alpine.qcow2 4G

qemu-system-x86_64 \
  -enable-kvm \
  -m 512 \
  -smp 1 \
  -cpu host \
  -hda alpine.qcow2 \
  -cdrom alpine-standard-3.20.0-x86_64.iso \
  -boot d
```
### 5.4 Scientific Linux
```bash
qemu-img create -f qcow2 scientific.qcow2 20G

qemu-system-x86_64 \
  -enable-kvm \
  -m 2048 \
  -smp 2 \
  -cpu host \
  -hda scientific.qcow2 \
  -cdrom SL-7.9-x86_64-DVD.iso \
  -boot d
```
## 6. Conclusión

A partir de la habilitación correcta de repositorios en Ubuntu 24.04.3 LTS fue posible instalar QEMU 8.2.2 junto con KVM y herramientas asociadas. Los procedimientos descritos permiten la creación y ejecución de máquinas virtuales para múltiples sistemas operativos de forma directa, flexible y reproducible.

Este entorno es adecuado para fines académicos, pruebas de laboratorio, investigación y virtualización ligera sin depender necesariamente de herramientas gráficas.
