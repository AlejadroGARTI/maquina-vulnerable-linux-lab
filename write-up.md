# Desarrollo

## 1. Descripción general del sistema

### Descripción Técnica del Sistema (Máquina: Ammonia)

- Tipo de sistema: Servidor interno basado en el sistema operativo Ubuntu Server 
- Finalidad: Alojar la aplicación web corporativa / blog de divulgación científica de la organización.

- Servicios principales instalados:
- - Servidor web lighttpd ((MD-2)) en el puerto 7664 corriendo una instancia de WordPress ((WB-1)).
- - Servidor de acceso remoto OpenSSH ((MD-1)) en el puerto estándar 22.

Usuarios relevantes:

haber_fritz: Cuenta local con bajos privilegios encargada de la gestión del entorno inicial.

clara_immerwahr: Cuenta local intermedia responsable de las tareas de automatización y almacenamiento de documentos científicos.

root: Superusuario y administrador absoluto del sistema.

Vulnerabilidades creadas (Cadena de explotación):

Inclusión de Archivos Locales (LFI): Presencia del plugin vulnerable Mail Masta en la aplicación web, lo que permite la lectura de archivos sensibles como wp-config.php y la fuga de credenciales de la base de datos.

Reutilización de Credenciales: La clave hallada en la configuración web es válida para el acceso SSH del usuario haber_fritz.

Permisos de Escritura Globales (777) en Tarea Cron: El script /opt/scripts/backup_notes.sh ((SC-1)) es ejecutado automáticamente por el usuario clara_immerwahr en intervalos de un minuto, pero permite su modificación por cualquier usuario del sistema.

Configuración Laxa de Sudoers: El usuario clara_immerwahr dispone de permisos de ejecución para el binario /usr/bin/awk bajo privilegios de root sin requerir contraseña (NOPASSWD).

Objetivo de la auditoría: El auditor deberá identificar la vulnerabilidad LFI para extraer las credenciales, consolidar acceso inicial por SSH como haber_fritz, secuestrar la tarea programada modificando el script de mantenimiento para pivotar al usuario clara_immerwahr y, finalmente, abusar de la directiva de sudo mediante la técnica de escape del binario awk (GTFOBins) para comprometer totalmente el sistema obteniendo una shell como root.

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

## 8. Write-up de resolución
- [Write-up de resolución](ruta-explotacion.md)
  
## 9. Credenciales white box
- [Credenciales white box](credenciales-white-box.md)



