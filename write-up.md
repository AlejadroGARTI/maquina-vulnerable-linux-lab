# Desarrollo

## 1. Descripción general del sistema

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
# Análisis de Riesgos y Vulnerabilidades del Sistema

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



