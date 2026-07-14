### Diagrama de la arquitectura del sistema

```bash
                                                ┌────────┐     ┌────────┐
                                                │ WEB 1  │     │ WEB 2  │
                                       ((WB-1)) │ maint. │     │TeamCity│ ((WB-2))
                                                └───┬────┘     └───┬────┘ 
                                                    │              │
                                                ┌───▼────┐     ┌───▼────┐
                                       ((MD-2)) │ APACHE │     │ TOMCAT │ ((MD-3))  
                                ┌─────────┐     └───┬────┘     └───┬────┘ 
                       ((MD-1)) │ OpenSSH │         │ pt. 80       │ pt. 50000
                                └────┬────┘         └──────┬───────┘
                                     │ pt. 22              │ 
                                ┌────▼────┐           ┌────▼────┐               
                      ((COM-1)) │   SSH   │           │  HTTP   │ ((COM-2))          
                                └─────────┘           └─────────┘      
                                     │                     │      
                                     └──────────┼──────────┘
                                                │
                                        ┌───────▼───────┐
                         ((AC-1)) ──────│ Ubuntu Server │ ((SO-1)) 
                                        └───────────────┘
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
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'amoníaco!(NH3)';
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






