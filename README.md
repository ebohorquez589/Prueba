# 1. Resumen Ejecutivo

El sistema de **Bibliocastia** es una solución automatizada diseñada para la gestión y sincronización de datos en un entorno **Raspberry Pi**.

Este sistema opera de manera continua y utiliza un **Orquestador (`Timer.py`)** basado en **APScheduler** para gestionar la ejecución de tareas críticas.  
El orquestador alterna la conectividad de red entre **WAN (Internet)** y **LAN (Red Local)** para ejecutar dos procesos principales:

**🌐 Tarea WAN**

**Script:** `GlideExportBot.py`

**🖧 Tarea LAN**

**Script:** `ethernet_tasks.py`

---

## 1.1 Programación de Ejecución

Las tareas se ejecutan automáticamente en ciclos **tres veces al día**, según la programación definida en la variable `SCHEDULE_HOURS`.

## 1.2 Alta Disponibilidad y Resiliencia

Además de la programación, el sistema incorpora lógica de alta disponibilidad y resiliencia, incluyendo:

- Mecanismos de reintentos de conexión  
- **Fallback** a Wi-Fi  
- Un **Modo de Emergencia** que se activa automáticamente ante cualquier fallo en las tareas críticas  

Esto asegura la **continuidad de la operación** o deja el sistema en un **estado conocido y seguro**.

## Nota

> **APScheduler (Advanced Python Scheduler)** es una biblioteca de Python que permite programar la ejecución de funciones (tareas o "jobs") para que se ejecuten más tarde, ya sea una sola vez o de forma periódica.  
>  
> Es ideal para automatizar tareas dentro de aplicaciones Python existentes, como el código del Orquestador, sin depender de herramientas externas del sistema operativo como **cron**.

---

## 1.3 Diagrama de Arquitectura General
```mermaid
graph TD
    B[APScheduler<br/>3 ejecuciones diarias<br/>Horarios: 05:00, 13:00, 21:00] --> A[TIMER.PY <br/>Orquestador]
    A --> C[FASE WAN<br/>Internet]
    A --> D[FASE LAN<br/>Red Local]
    
    C --> E[Wired Connection 2<br/>D-Link]
    D --> F[Wired Connection 1<br/>eth0]
    
    E --> G[INTERNET]
    F --> H[RED LOCAL <br/>CIFS]
    
    G --> I[Glide Export Bot]
    H --> J[Archivos CSV]
    
    I --> K[Notificaciones<br/>WhatsApp/Email]
    J --> K

## 1.4 Componentes Principales
| Componente            | Tipo                 | Descripción                                                                 | Ubicación                                                                     | Tipo de Conexión               |
|-----------------------|----------------------|------------------------------------------------------------------------------|-------------------------------------------------------------------------------|--------------------------------|
| Orquestador Principal | Script Python        | Lógica de programación, gestión de red, reintentos y modo emergencia.       | /home/rasp5/Desktop/BIBLIOCASTIA/NOOTEBOKS/timer.py                           | WAN / LAN / Wi-Fi (Fallback)  |
| Tarea Internet        | Script Python        | Ejecuta la lógica de exportación/bot que requiere acceso a Internet.        | SCRIPT_PATH (/home/rasp5/Desktop/BIBLIOCASTIA/NOOTEBOKS/GlideExportBot.py)    | WAN (Wired connection 2)       |
| Tarea Red Local       | Script Python        | Ejecuta tareas que requieren acceso a la Red Local.                         | ETHERNET_TASKS_SCRIPT (/home/rasp5/Desktop/BIBLIOCASTIA/NOOTEBOKS/ethernet_tasks.py) | LAN (Wired connection 1)       |
| Conexión WAN          | Perfil de Red (NM)   | Conexión cableada principal para acceso a Internet (p. ej., módem 3G/4G).   | Configuración de NetworkManager (NM)                                           | WAN (Wired connection 2)       |
| Conexión LAN          | Perfil de Red (NM)   | Conexión cableada principal para acceso a la Red Local.                     | Configuración de NetworkManager (NM)                                           | LAN (Wired connection 1)       |
| Wi-Fi (Fallback)      | Perfiles de Red (NM) | Conexiones inalámbricas almacenadas, usadas automáticamente en EMERGENCIA. | Perfiles de NetworkManager (NM)                                                | WAN (Wi-Fi)                    |
| APScheduler           | Librería Python      | Programación de la función run_all 3 veces al día.                          | Importada en timer.py                                                          | N/A (Scheduler)                |

