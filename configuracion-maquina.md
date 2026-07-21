# Configuración del servidor, de la arquitectura y de la configuración del sistema

## Diagrama de la arquitectura del sistema
```bash
                                               ┌────────┐
                                               │  WEB   │
                                       ((WB-1))│  Blog  │ (WordPress)
                                               └───┬────┘ 
                                                   │ 
                                               ┌───▼────┐
                                               │lighttpd│ ((MD-2)) 
                                ┌─────────┐    └───┬────┘ 
                        ((MD-1))│ OpenSSH │        │ pt. 7664
                                └────┬────┘        │
                                     │ pt. 22      │ 
                                ┌────▼────┐    ┌───▼─────┐               
                      ((COM-1)) │   SSH   │    │  HTTP   │ ((COM-2))          
                                └─────────┘    └─────────┘      
                                     │              │      
                                     └──────┬───────┘
                                            │
                                    ┌───────▼───────┐
                     ((AC-1)) ──────│ Ubuntu Server │ ((SO-1)) 
                                    └───────┬───────┘
                                            │
                     ┌──────────────────────┴──────────────────────┐
                     ▼                                             ▼
            [ Movimiento Lateral ]                      [ Escalada a Root ]
         ┌───────────────────────┐                  ┌───────────────────────┐
         │     haber_fritz       │                  │    clara_immerwahr    │
         │                       │                  │                       │
         │ - Crontab modificable │ ─(Cron Ejecuta)─►│ - Permisos de Sudo:   │
         │   (/opt/scripts/...)  │                  │   /usr/bin/awk        │
         └───────────────────────┘                  └───────────┬───────────┘
                                                                │ 
                                                                ▼ 
                                                          ┌───────────┐
                                                          │   ROOT    │
                                                          └───────────┘
```

## Especificaciones del Sistema
```bash
haber_fritz@ammonia:~$ lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 22.04.5 LTS
Release:        22.04
Codename:       jammy
```
##  Credenciales de Acceso
```bash
User: haber_fritz
Pass: 9/12/!1868_Br3sl$a5ia!
```

## Instalación y Actualización de Paquetes y Servicios Web
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install lighttpd mariadb-server php-cgi php-mysql php-curl php-gd php-xml php-mbstring unzip wget -y
```
Habilitación de los módulos FastCGI y FastCGI-PHP en Lighttpd, seguido del reinicio del servicio para aplicar los cambios y permitir que el servidor web procese archivos PHP correctamente.
```bash
sudo lighty-enable-mod fastcgi
sudo lighty-enable-mod fastcgi-php
sudo systemctl restart lighttpd
```
## Configuración del puerto de WordPress. El número CAS del amoníaco (NH₃) es 7664-41-7
```bash
sudo nano /etc/lighttpd/lighttpd.conf

server.modules = (
        "mod_indexfile",
        "mod_access",
        "mod_alias",
        "mod_redirect",
)

server.document-root        = "/var/www/html"
server.upload-dirs          = ( "/var/cache/lighttpd/uploads" )
server.errorlog             = "/var/log/lighttpd/error.log"
server.pid-file             = "/run/lighttpd.pid"
server.username             = "www-data"
server.groupname            = "www-data"
server.port                 = 7664
```
## Creación y configuración de la base de datos.
```bash
sudo mysql -u root

CREATE DATABASE wordpress DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci;
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY '9/12/!1868_Br3sl$a5ia!';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wp_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```
## Instalación de WordPress
```bash
# Limpiar el index por defecto de lighttpd
sudo rm -f /var/www/html/index.lighttpd.html

# Descargar y descomprimir WordPress
cd /tmp
wget https://wordpress.org/latest.tar.gz
tar -xzvf latest.tar.gz

# Mover los archivos al directorio web
sudo cp -r wordpress/* /var/www/html/

# Ajustar los permisos para que Lighttpd 
sudo chown -R www-data:www-data /var/www/html/
sudo chmod -R 755 /var/www/html/

# Contraseña: crU^A5rRlwI3NdP7ff
```
![](Evidencias_Visuales/wordpressconfig)
![](Evidencias_Visuales/wordpresscredentials)

## Instalación del plugin MailMasta 1.0
```bash
cd /var/www/html/wp-content/plugins/

sudo wget https://downloads.wordpress.org/plugin/mail-masta.zip

sudo unzip mail-masta.zip

sudo rm mail-masta.zip

sudo chown -R www-data:www-data /var/www/html/wp-content/plugins/mail-masta
```
![](Evidencias_Visuales/mailmasta)

## Creación de flag, directorios y archivos para despistar al atacante.

```bash
haber_fritz@ammonia:~$ zip -e proceso_haber_bosch.zip proceso_haber_bosch.txt proceso_hab
er.txt agricultura_1918.txt guerra_quimica.txt amoniaco_sintesis.txt nobel_1918.txt
Enter password:
Verify password:
        zip warning: name not matched: proceso_hab
  adding: proceso_haber_bosch.txt (deflated 11%)
er.txt: command not found
haber_fritz@ammonia:~$ zip -e proceso_haber_bosch.zip "proceso_haber_bosch.txt" "proceso_haber.txt" "agricultura_1918.txt" "guerra_quimica.txt" "amoniaco_sintesis.txt" "nobel_1918.txt"
Enter password:
Verify password:
updating: proceso_haber_bosch.txt (deflated 11%)
  adding: proceso_haber.txt (stored 0%)
  adding: agricultura_1918.txt (stored 0%)
  adding: guerra_quimica.txt (stored 0%)
  adding: amoniaco_sintesis.txt (stored 0%)
  adding: nobel_1918.txt (stored 0%)
haber_fritz@ammonia:~$ echo "Lista de compras" > lista_compras.txt
etc.....

```
---
---

## Creación de Segundo Usuario

Creación del usuario clara_immerwahr con contraseña pacifista1915chemsitry!, estableciendo una segunda cuenta en el sistema para ampliar la superficie de ataque y permitir vectores de movimiento lateral.

```bash
sudo adduser clara_immerwahr
pacifista1915chemsitry!
```

## Creación de Script de Backup
```bash
sudo mkdir -p /opt/scripts
sudo nano /opt/scripts/backup_notes.sh
```

## Configuración de Script de Backup y Tarea Programada
```bash
#!/bin/bash
# Backup de las notas científicas de Clara
tar -czf /home/clara_immerwahr/notes_backup.tar.gz /home/clara_immerwahr/documents/ 2>/dev/null

sudo chmod 777 /opt/scripts/backup_notes.sh
sudo nano /etc/crontab
```
## Programación de Tarea Cron

onfiguración de una tarea programada en el sistema que ejecutará el script /opt/scripts/backup_notes.sh cada minuto con los privilegios de la usuaria clara_immerwahr, introduciendo así una vulnerabilidad de escalada de privilegios al permitir la ejecución periódica de un script con permisos 777 que puede ser modificado por cualquier usuario del sistema.

```bash
  GNU nano 6.2                                                                                       /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
# You can also override PATH, but by default, newer versions inherit it from the environment
#PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#
* *     * * *   clara_immerwahr /opt/scripts/backup_notes.sh
```

## Configuración de Sudoers para Clara

Adición de un privilegio en el archivo /etc/sudoers que permite a la usuaria clara_immerwahr ejecutar el comando /usr/bin/awk con permisos de superusuario sin necesidad de proporcionar contraseña, creando así un vector de escalada de privilegios mediante la explotación de la funcionalidad de ejecución de comandos de awk.

```bash
sudo visudo

@includedir /etc/sudoers.d

clara_immerwahr ALL=(ALL) NOPASSWD: /usr/bin/awk
```

## Eliminación de privilegios de haber_fritz
```bash
gpasswd -d haber_fritz sudo
```

## Configuración del firewall

```bash
root@ammonia:~# sudo ufw status
Status: active

To         Action   From
---        ---      ---
22/tcp     ALLOW    Anywhere
7664/tcp   ALLOW    Anywhere
22/tcp (v6) ALLOW   Anywhere (v6)
7664/tcp (v6) ALLOW Anywhere (v6)
```

## Eliminación del historial y archivos temporales

Se ejecutaron los siguientes comandos en cada usuario para borrar cualquier registro de la línea de comandos

```bash
apt-get clean && apt-get autoremove -y

rm -rf /tmp/* /var/tmp/*

find /var/log -type f -exec truncate -s 0 {} \;

cat /dev/null > /root/.bash_history
cat /dev/null > /home/clara_immerwahr/.bash_history 2>/dev/null
cat /dev/null > /home/haber_fritz/.bash_history 2>/dev/null

unset HISTFILE
history -c
poweroff
```
