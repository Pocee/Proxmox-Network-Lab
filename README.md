# Proxmox Network Lab + Hardening

Despliegue de servicios corporativos esenciales sobre Proxmox VE con hardening y monitorización básica. Entorno de laboratorio SOC sobre LXC Debian.

---

## Índice

1. [Proxmox: Instalación y configuración inicial](#1-proxmox-instalación-y-configuración-inicial)
2. [LXC Debian: Creación y configuración base](#2-lxc-debian-creación-y-configuración-base)
3. [SSH: Hardening y autenticación por clave](#3-ssh-hardening-y-autenticación-por-clave)
4. [Fail2ban: Protección contra fuerza bruta SSH](#4-fail2ban-protección-contra-fuerza-bruta-ssh)
5. [SMB Server: Samba con hardening](#5-smb-server-samba-con-hardening)

---

## 1. Proxmox: Instalación y configuración inicial

### Adaptador de red bridge (en host Debian con nmcli)

Si Proxmox se instala sobre Debian, hay que crear un bridge de red para que las VMs tengan acceso directo a la red física.

```bash
# Identificar la interfaz principal (ej: eno1)
ip a

# Crear bridge con DHCP automático
sudo nmcli con add type bridge ifname br0 con-name br0 ipv4.method auto

# Añadir la interfaz física como puerto del bridge
sudo nmcli con add type bridge-slave ifname eno1 master br0

# Bajar la interfaz original (se pierde conexión momentáneamente)
sudo nmcli con down "Wired connection 1"

# Levantar el bridge
sudo nmcli con up br0

# Eliminar la conexión antigua para evitar conflicto en el arranque
sudo nmcli con del "Wired connection 1"
```

Al crear VMs en virt-manager, seleccionar la conexión `br0`.

### Repositorios: deshabilitar enterprise, habilitar no-subscription

En `Datacenter > Updates > Repositories`:

- Deshabilitar `pve-enterprise` y `ceph enterprise` (requieren suscripción)
- Añadir `pve-no-subscription` y `ceph no-subscription`

Luego desde la shell del nodo: `Datacenter > Shell > Upgrade`.

### Ampliar disco: eliminar LVM-thin y recuperar espacio

Por defecto Proxmox reserva espacio en un volumen `pve/data` (LVM-thin) que no es necesario en un lab. Se puede recuperar ese espacio para el sistema raíz.

En `Datacenter > Storage`: añadir contenido `Backup` y `Container` al almacenamiento `local`, y eliminar `local-lvm`.

Luego ejecutar en la shell, uno a uno:

```bash
lvremove /dev/pve/data
lvresize -l +100%FREE /dev/pve/root
resize2fs /dev/mapper/pve-root
```

---

## 2. LXC Debian: Creación y configuración base

### Crear contenedor

1. Ir a `local (pve) > CT Templates > Templates`
2. Descargar la plantilla deseada (en este lab: Debian 13.1)
3. Pulsar `Create CT` (arriba a la derecha)
4. Asignar hostname, contraseña e ID
5. Seleccionar template, disco, CPU cores y memoria
6. En `Network`, dejar IPv4 en `DHCP`
7. DNS por defecto y confirmar

Si tras arrancar no hay ping ni `apt`, volver a Network y reconfirmar DHCP.

```bash
# Actualizar el sistema dentro del contenedor
apt update && apt upgrade -y
```

---

## 3. SSH: Hardening y autenticación por clave

### Instalación

```bash
sudo apt update && sudo apt install openssh-server -y

# Verificar que el servicio está activo
sudo systemctl status ssh

# Verificar puerto de escucha
sudo ss -tlnp | grep :22
```

### Crear usuario y autenticación por clave

```bash
# En el servidor: crear usuario con sudo
adduser usuario
usermod -aG sudo usuario

# En el cliente: generar par de claves ed25519
ssh-keygen -t ed25519 -C "proxmox-debiantest"

# Copiar la clave pública al servidor
ssh-copy-id -i ~/.ssh/id_ed25519.pub usuario@192.168.100.50
```

Tras esto el login SSH no pedirá contraseña.

### Hardening de sshd_config

Editar `/etc/ssh/sshd_config`:

```
PermitRootLogin no
LogLevel VERBOSE
ClientAliveInterval 600
ClientAliveCountMax 3
MaxAuthTries 3
```

| Directiva | Motivo |
|---|---|
| `PermitRootLogin no` | Obliga a usar un usuario con sudo, eliminando el vector de ataque directo sobre root |
| `LogLevel VERBOSE` | Registra fingerprint de clave en cada autenticación, útil para auditoría y detección de accesos no autorizados |
| `ClientAliveInterval 600` | Cierra sesiones inactivas tras ~30 min (600s × 3), libera recursos y reduce ventana de acceso no atendido |
| `MaxAuthTries 3` | Limita intentos por conexión antes de cortar, complementa a Fail2ban |

```bash
sudo systemctl restart ssh
```

### Logging con rsyslog

Por defecto en Debian el logging SSH va a `journald`. Se instala `rsyslog` para tener un archivo `auth.log` persistente y compatible con Fail2ban.

```bash
apt install rsyslog -y
systemctl enable --now rsyslog
systemctl restart ssh

# Verificar que auth.log se genera con tráfico
tail -f /var/log/auth.log
```

Comprobar que logrotate está configurado para evitar que `auth.log` llene el disco:

```bash
cat /etc/logrotate.d/rsyslog
# Debe incluir: rotate 4, weekly, compress, delaycompress
```

---

## 4. Fail2ban: Protección contra fuerza bruta SSH

```bash
apt install fail2ban -y
```

Crear `/etc/fail2ban/jail.local`:

```ini
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 3
banaction = iptables-multiport
backend = polling

[sshd]
enabled  = true
port     = ssh
filter   = sshd
logpath  = /var/log/auth.log
mode     = aggressive
maxretry = 3
bantime  = 1h
findtime = 10m
```

```bash
systemctl restart fail2ban

# Verificar que la jaula SSH está activa
fail2ban-client status sshd
```

**Verificación de ban:** forzar autenticación por contraseña con usuario inexistente y comprobar que banea en el 3er intento:

```bash
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no usuariofake@192.168.100.50
```

`fail2ban-client status sshd` debe mostrar la IP baneada en `Banned IP list`.

---

## 5. SMB Server: Samba con hardening

### Instalación y preparación

```bash
sudo apt install samba -y

# Crear directorio del share
sudo mkdir -p /srv/samba/secure-share
sudo chown -R poce:poce /srv/samba/secure-share
sudo chmod 755 /srv/samba/secure-share

# Archivo de prueba
echo "SOC Lab - Samba Share - $(date)" | sudo tee /srv/samba/secure-share/README.txt
```

### Configuración: /etc/samba/smb.conf

```ini
[global]
   workgroup = WORKGROUP
   server string = Lab File Server
   netbios name = DEBIANTEST

   # Hardening de protocolo
   server min protocol = SMB2_10
   server max protocol = SMB3
   server signing = mandatory
   smb encrypt = required
   map to guest = never
   security = user

   # Logging por cliente
   logging = file
   log level = 2
   log file = /var/log/samba/log.%m
   max log size = 10000

   # Reducción de superficie
   load printers = no
   disable spoolss = yes
   usershare allow guests = no
   idmap config * : backend = tdb

[secure-share]
   path = /srv/samba/secure-share
   comment = Shared folder for SOC Lab
   read only = yes
   valid users = poce
   hosts allow = 192.168.100.
   create mask = 0644
   directory mask = 0755
```

### Por qué cada opción de hardening

| Directiva | Motivo |
|---|---|
| `server min protocol = SMB2_10` | Deshabilita SMB1, vulnerable a EternalBlue y otros exploits críticos |
| `server signing = mandatory` | Obliga firma criptográfica en todos los paquetes SMB, previene ataques MITM |
| `smb encrypt = required` | Fuerza cifrado SMB3 extremo a extremo, protege datos en tránsito aunque la red esté comprometida |
| `map to guest = never` | Bloquea cualquier acceso anónimo o invitado sin excepción |
| `logging = file` | Fuerza el backend de archivo; sin esto Samba 4.x puede priorizar journald e ignorar `log level` |
| `log file = /var/log/samba/log.%m` | Un archivo de log por cliente (`%m` = nombre NetBIOS). Facilita análisis forense y detección de fuerza bruta por origen |
| `load printers = no` / `disable spoolss = yes` | Elimina el subsistema de impresión, vector histórico de exploits (PrintNightmare) |
| `hosts allow = 192.168.100.` | Restringe acceso a la subred del lab. `hosts deny` es redundante cuando existe `hosts allow` |

### Activar usuario Samba

```bash
sudo smbpasswd -a poce
sudo smbpasswd -e poce

# Verificar
pdbedit -L | grep poce
```

### Verificación y arranque

```bash
# Validar sintaxis
testparm -s

# Reiniciar servicios
sudo systemctl restart smbd nmbd

# Comprobar nivel de log activo
testparm -v | grep "log level"

# Revisar logs al conectar un cliente
ls -lh /var/log/samba/
tail -f /var/log/samba/log.<NOMBRE_CLIENTE>
```

### Conexión desde cliente

```bash
# Montar el share
sudo mkdir -p /mnt/smb-test
sudo mount -t cifs //192.168.100.50/secure-share /mnt/smb-test \
  -o username=poce,password=TU_CONTRASEÑA,vers=3.0,ro

ls -la /mnt/smb-test

# Verificar negociación SMB3
smbclient //192.168.100.50/secure-share -U poce -m SMB3
```
## Authors

- [@Pocee](https://www.github.com/Pocee)
