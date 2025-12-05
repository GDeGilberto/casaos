# CasaOS en Raspberry Pi — Guía de instalación desde cero
(Con Docker y almacenamiento USB)

Este README explica, paso a paso y en castellano, cómo preparar una Raspberry Pi para ejecutar CasaOS desde cero, cómo instalar y fijar una versión de Docker compatible, cómo montar una USB para el almacenamiento persistente y un ejemplo/plantilla para desplegar aplicaciones (ejemplo: Odoo). Está pensado para Raspberry Pi con Raspberry Pi OS (32/64-bit) o una distro Debian/Ubuntu compatible.

## Índice
- Requisitos
- Preparar la Raspberry Pi
- Instalar y fijar la versión de Docker recomendada
- Instalar CasaOS
- Preparar y montar la USB para almacenamiento persistente
- Plantilla / ejemplo: desplegar Odoo (Docker Compose)
- Backup y buenas prácticas
- Verificación y resolución de problemas
- Anexos: resumen rápido

---

## 1) Requisitos
- Raspberry Pi 3/4/400/Zero 2 W (recomendada Pi 4 o superior para producción).
- 4 GB+ RAM recomendado para Odoo / apps de producción.
- Raspberry Pi OS (preferible Debian-based) o Ubuntu Server ARM.
- Conexión a internet.
- Una unidad USB (pendrive o disco USB) para datos / backups.
- Acceso SSH o terminal local con privilegios `sudo`.

---

## 2) Preparar la Raspberry Pi (sistema base)
- Flashear Raspberry Pi OS / Ubuntu Server con Raspberry Pi Imager o balenaEtcher.
- Actualizar el sistema:
```bash
sudo apt update && sudo apt upgrade -y
```
- Instalar paquetes útiles:
```bash
sudo apt install -y ca-certificates curl gnupg lsb-release software-properties-common
```

---

## 3) Instalar y fijar la versión de Docker (recomendación y procedimiento)
CasaOS funciona con Docker. Algunas versiones de Docker recientes pueden provocar incompatibilidades, por eso se recomienda una versión estable y probada (por ejemplo Docker 20.10.x). Si prefieres usar la versión más reciente, pruébala primero en un entorno de test.

### A) Añadir repositorio oficial de Docker
```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
```

### B) Buscar versiones disponibles y elegir una
```bash
apt-cache madison docker-ce
```
Selecciona una versión 20.10.x que aparezca en la lista (el formato puede variar según la distro).

### C) Instalar la versión elegida (usar la versión exacta mostrada por apt-cache)
```bash
VERSION="<la-version-que-elegiste>"
sudo apt install -y docker-ce=$VERSION docker-ce-cli=$VERSION containerd.io
```
Ejemplo (ajusta la variable `VERSION` según la salida de `apt-cache`):
```bash
sudo apt install -y docker-ce=5:20.10.17~3-0~debian-buster docker-ce-cli=5:20.10.17~3-0~debian-buster containerd.io
```

### D) Fijar la versión para evitar actualizaciones automáticas
```bash
sudo apt-mark hold docker-ce docker-ce-cli containerd.io
```

### E) Verificar instalación
```bash
sudo systemctl enable --now docker
docker --version
sudo docker run --rm hello-world
```

> Nota: Si no aparece la versión que quieres en los repositorios, consulta la documentación oficial de Docker para obtener el paquete `.deb` correcto o usa el script de instalación con precaución. Para Raspberry Pi OS / Debian armhf/arm64, usa las builds oficiales para tu arquitectura.

---

## 4) Instalar CasaOS
La instalación oficial (comprueba siempre el script antes de ejecutarlo):
```bash
curl -fsSL https://get.casaos.io | sudo bash
```

Opcional recomendado:
- Cambiar el puerto de CasaOS a 90 si quieres dejar el puerto 80 libre para otros servicios.

Si quieres que el almacenamiento de CasaOS apunte a la USB, crea los volúmenes o directorios en la USB y, si es necesario, ajusta las rutas de datos en la configuración de CasaOS o crea symlinks (ver sección montaje USB).

---

## 5) Preparar y montar la USB para almacenamiento persistente
Se recomienda formatear la USB a `ext4` para mayor estabilidad en Linux. Si necesitas compatibilidad con Windows, usa `exFAT` y ten en cuenta permisos.

### A) Identificar la unidad
```bash
lsblk -f
# Busca tu dispositivo, p.ej. /dev/sda1 o /dev/sdb1
```

### B) (Opcional) Formatear a ext4
```bash
sudo umount /dev/sda1
sudo mkfs.ext4 -L CASAOS_DATA /dev/sda1
```

### C) Crear punto de montaje y montar
```bash
sudo mkdir -p /mnt/usb
sudo mount /dev/sda1 /mnt/usb
```

### D) Obtener UUID y añadir en /etc/fstab para montaje automático
```bash
sudo blkid /dev/sda1
# Copia el UUID y añade a /etc/fstab:
# UUID=xxxxxxxx /mnt/usb ext4 defaults,noatime 0 2
sudo nano /etc/fstab
sudo mount -a
```

### E) Ajustar permisos (ejemplo usando usuario `pi` o el usuario que vayas a usar)
```bash
sudo chown -R pi:pi /mnt/usb
sudo chmod -R 755 /mnt/usb
```

### F) Organización sugerida dentro de la USB
```
/mnt/usb/
  casaos/         # datos de CasaOS o volúmenes mapeados
  backups/        # backups periódicos
  odoo/           # datos persistentes para Odoo (db, filestore, addons)
  docker-volumes/ # si usas bind mounts para contenedores
```

---

## 6) Docker Compose de las aplicaciones instaladas con CasaOS y su configuración para respaldar información en USB

A continuación se muestra una plantilla (adaptada) para Odoo + PostgreSQL usada en CasaOS. Asegúrate de ajustar rutas, contraseñas y permisos según tu entorno.

```yaml
name: big-bear-odoo

services:
  big-bear-odoo:
    cpu_shares: 90
    command: []
    container_name: odoo
    deploy:
      resources:
        limits:
          memory: 4096M
    environment:
      - HOST=big-bear-odoo-db
      - PASSWORD=password_aqui
      - USER=odoo
    hostname: odoo
    image: odoo:19
    labels:
      icon: https://cdn.jsdelivr.net/gh/bigbeartechworld/big-bear-universal-apps/apps/odoo/logo.jpg
    ports:
      - mode: ingress
        target: 8069
        published: "8069"
        protocol: tcp
    restart: unless-stopped
    user: root
    volumes:
      - type: bind
        source: /media/devmon/sda1-usb-_USB_DISK_2.0_90/Odoo/data
        target: /var/lib/odoo
        bind:
          create_host_path: true
      - type: bind
        source: /media/devmon/sda1-usb-_USB_DISK_2.0_90/Odoo/config
        target: /etc/odoo
        bind:
          create_host_path: true
    networks:
      - big_bear_odoo_network
    privileged: false

  big-bear-odoo-db:
    cpu_shares: 90
    command: []
    container_name: odoo-db
    deploy:
      resources:
        limits:
          memory: 2048M
    environment:
      - POSTGRES_DB=postgres
      - POSTGRES_PASSWORD=password_aqui
      - POSTGRES_USER=odoo
    hostname: odoo-db
    image: postgres:15
    labels:
      icon: https://cdn.jsdelivr.net/gh/bigbeartechworld/big-bear-universal-apps/apps/odoo/logo.jpg
    restart: unless-stopped
    user: root
    volumes:
      - type: bind
        source: /media/devmon/sda1-usb-_USB_DISK_2.0_90/PostgreSQL/data
        target: /var/lib/postgresql/data
        bind:
          create_host_path: true
    networks:
      - big_bear_odoo_network
    privileged: false

networks:
  big_bear_odoo_network:
    name: big-bear-odoo_big_bear_odoo_network
    driver: bridge

x-casaos:
  architectures:
    - amd64
    - arm64
  author: BigBearTechWorld
  category: BigBearCasaOS
  description:
    en_us: Open-source business management software suite designed to streamline
      various aspects of business operations. With its modular structure, users
      can choose and integrate specific applications such as accounting,
      inventory, and sales. Odoo provides a user-friendly interface, scalability
      for businesses of all sizes, and is available in both community (free) and
      enterprise editions. Its integrated approach and customization options
      make it a popular choice for comprehensive ERP (Enterprise Resource
      Planning) solutions.
  developer: odoo
  icon: https://cdn.jsdelivr.net/gh/bigbeartechworld/big-bear-universal-apps/apps/odoo/logo.jpg
  index: /
  is_uncontrolled: false
  main: big-bear-odoo
  port_map: "8069"
  scheme: http
  store_app_id: big-bear-odoo
  tagline:
    en_us: Open-source business management software with modular applications for
      streamlined operations.
  tips:
    before_install:
      en_us: >
        Documentation:
        https://community.bigbeartechworld.com/t/added-odoo-to-bigbearcasaos/1115?u=dragonfire1119
  title:
    en_us: Odoo
```

- Guardar como `/home/pi/odoo/docker-compose.yml` (o directamente en CasaOS si usas un editor de apps que acepte Compose).
- Crear directorios locales en la USB antes de arrancar:
```bash
sudo mkdir -p /mnt/usb/odoo/postgres /mnt/usb/odoo/odoo-data /mnt/usb/odoo/odoo-addons
sudo chown -R 1000:1000 /mnt/usb/odoo   # Postgres y Odoo suelen usar UID 1000; ajusta si es necesario
```

- Levantar los servicios:
```bash
docker compose -f /home/pi/odoo/docker-compose.yml up -d
```

- Acceder a Odoo en: `http://<ip-raspberry>:8069`

> Nota sobre CasaOS: Si prefieres instalar a través del gestor de aplicaciones de CasaOS, puedes crear una aplicación personalizada y pegar este Docker Compose (o los campos equivalentes). Asegúrate de mapear rutas de datos a `/mnt/usb/...` para persistencia.

---

## 7) Backup y estrategia de respaldo (USB)
- Backups locales: usa `rsync` para copiar datos críticos a otra unidad o a la nube:
```bash
rsync -a --delete /mnt/usb/casaos/ /path/to/backup/location/
```

- Ejemplo de script de backup simple (`/usr/local/bin/backup-odoo.sh`):
```bash
#!/bin/bash
TIMESTAMP=$(date +%F-%H%M)
BACKUP_DIR="/mnt/usb/backups/odoo/$TIMESTAMP"
mkdir -p "$BACKUP_DIR"
# Dump de postgres (desde contenedor)
docker exec -t odoo-db pg_dump -U odoo odoo > "$BACKUP_DIR/odoo.sql"
rsync -a /mnt/usb/odoo/ "$BACKUP_DIR/odoo-files/"
```
- Programar con `cron` o `systemd-timer`.
- Para restauración, restaurar archivos y cargar la base de datos con `psql`.

---

## 8) Verificación y comandos útiles
- Comprobar Docker:
```bash
docker --version
sudo systemctl status docker
docker ps -a
```
- Comprobar CasaOS:
```bash
sudo systemctl status casaos
# UI: http://<ip-raspberry>/ (puerto por defecto 80, o el que hayas configurado)
```
- Logs:
```bash
sudo journalctl -u docker -f
sudo journalctl -u casaos -f
```

---

## 9) Buenas prácticas y consideraciones
- Usa contraseñas seguras y gestiona credenciales con variables de entorno o vaults.
- Mantén copias periódicas fuera del dispositivo (nube, otra USB, NAS).
- Antes de actualizar Docker o CasaOS, prueba en entorno de staging.
- Si la USB es lenta, coloca solo datos críticos; considera usar una SD/SSD para el sistema.
- Considera usar un SSD vía USB para mayor fiabilidad en producción.

---

## 10) Resolución rápida de problemas
- Docker no inicia: revisar `sudo journalctl -u docker` y liberar espacio en disco.
- CasaOS no responde: revisar logs y reiniciar servicio `sudo systemctl restart casaos`.
- Permisos en la USB: ajustar owner/uid/gid.

---

## Anexos: Resumen paso a paso (rápido)
1. Flashea Raspberry Pi OS y arranca.  
2. `sudo apt update && sudo apt upgrade -y`  
3. Instalar Docker desde repositorio oficial y fijar versión (ver sección 3).  
4. Montar USB en `/mnt/usb` (ver sección 5) y preparar carpetas.  
5. Instalar CasaOS (`curl -fsSL https://get.casaos.io | sudo bash`) o desplegar manualmente los contenedores.  
6. Desplegar Odoo con `docker compose` apuntando a `/mnt/usb/odoo` para datos.  
7. Configurar backups regulares con `rsync` o dump de PostgreSQL.

---

Si quieres, puedo:
- Generar un `docker-compose.yml` listo con variables en `.env` para Odoo.
- Crear un script de montaje/formatado de la USB que incluya `fstab` con UUID.
- Preparar un script de instalación automatizada que instale Docker (versión exacta) y CasaOS.

Dime cuál prefieres y lo preparo.
