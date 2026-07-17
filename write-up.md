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

| **Activo** | **Tipo** | **Función** | **Riesgo o Vulnerabilidad** |
|------------|----------|-------------|------------------------------|
| **Apache** | Servicio | Publicar la aplicación web (WordPress). | Información sensible expuesta (LFI a través del plugin Mail Masta que permite leer archivos como `wp-config.php`). |
| **SSH** | Servicio | Permitir el acceso remoto para la administración del servidor. | Credenciales débiles o reutilizadas (Uso de la clave de la base de datos de WordPress para loguearse como el usuario `haber_fritz`). |
| **Usuario local (clara_immerwahr)** | Cuenta | Acceso al sistema y gestión de notas/documentos del laboratorio. | Permisos incorrectos en la configuración de sudoers (Permite ejecutar `/usr/bin/awk` con privilegios de root sin contraseña). |
| **Script de mantenimiento (backup_notes.sh)** | Archivo | Automatizar la tarea de copia de seguridad de los documentos de Clara. | Permisos globales laxos (777). Puede ser modificado por un usuario no autorizado (`haber_fritz`) para secuestrar el flujo de ejecución (Cron). |
| **Registros** | Información | Registrar accesos, acciones del sistema y errores de los servicios. | Modificación o acceso no autorizado a datos del sistema si los permisos de lectura de logs revelaran más de la cuenta. |

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



