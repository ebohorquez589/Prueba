# 📚 BIBLIOCASTIA

## 1. RESUMEN EJECUTIVO

**BIBLIOCASTIA** es un sistema automatizado de gestión y sincronización de datos bibliotecarios que opera en un entorno **Raspberry Pi**.  
El sistema alterna entre operaciones de red **WAN (Internet)** y **LAN (red local)** mediante un **orquestador central**, ejecutando tareas programadas **tres veces al día**.

---

## 1.1 COMPONENTES PRINCIPALES

### 🕒 **Timer.py**
**Orquestador central del sistema**

---

### 🌐 **GlideExportBot.py**
**Bot de exportación de datos desde Glide (WAN)**

---

### 🖧 **ethernet_tasks.py**
**Procesador de archivos en red local (LAN)**

---

### 🧩 **Scripts auxiliares**
- **INDICADORES.py**
- **configwha.py**
