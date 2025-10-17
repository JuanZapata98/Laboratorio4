# Laboratorio 4

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
