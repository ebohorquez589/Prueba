## 1. 📑 Resumen Ejecutivo

El sistema de **Bibliocastia** es una solución automatizada diseñada para la gestión y sincronización de datos en un entorno **Raspberry Pi**.

Este sistema opera de manera continua y utiliza un **Orquestador (`Timer.py`)** basado en **APScheduler** para gestionar la ejecución de tareas críticas.  
El orquestador alterna la conectividad de red entre **WAN (Internet)** y **LAN (Red Local)** para ejecutar dos procesos principales:

**🌐 Tarea WAN**

**Script:** `GlideExportBot.py`

**🖧 Tarea LAN**

**Script:** `ethernet_tasks.py`

---

## 1.1 🕒 Programación de Ejecución

Las tareas se ejecutan automáticamente en ciclos **tres veces al día**, según la programación definida en la variable `SCHEDULE_HOURS`.

## 1.2 🔁 Alta Disponibilidad y Resiliencia

Además de la programación, el sistema incorpora lógica de alta disponibilidad y resiliencia, incluyendo:

- Mecanismos de reintentos de conexión  
- **Fallback** a Wi-Fi  
- Un **Modo de Emergencia** que se activa automáticamente ante cualquier fallo en las tareas críticas  

Esto asegura la **continuidad de la operación** o deja el sistema en un **estado conocido y seguro**.

## 🧩 Nota

> **APScheduler (Advanced Python Scheduler)** es una biblioteca de Python que permite programar la ejecución de funciones (tareas o "jobs") para que se ejecuten más tarde, ya sea una sola vez o de forma periódica.  
>  
> Es ideal para automatizar tareas dentro de aplicaciones Python existentes, como el código del Orquestador, sin depender de herramientas externas del sistema operativo como **cron**.

---

## 1.3 🏗️ Diagrama de Arquitectura General
graph TD
    A[TIMER.PY (Orquestador)] --> B[APScheduler<br/>3 ejecuciones diarias<br/>Horarios: 05:00, 13:00, 21:00]
    B --> C{FASE WAN<br/>(Internet)}
    B --> D{FASE LAN<br/>(Red Local)}
    
    C --> E[Wired Connection 2<br/>(D-Link)]
    D --> F[Wired Connection 1<br/>(eth0)]
    
    E --> G[INTERNET]
    F --> H[RED LOCAL (CIFS)]
    
    G --> I[Glide Export Bot]
    H --> J[Archivos CSV]
    
    I --> K[Notificaciones<br/>WhatsApp/Email]
    J --> K

    
