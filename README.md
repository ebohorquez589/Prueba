# 1. Resumen Ejecutivo

El sistema de **Bibliocastia** es una solución automatizada diseñada para la gestión y sincronización de datos en un entorno **Raspberry Pi**.

Este sistema opera de manera continua y utiliza un **Orquestador (`Timer.py`)** basado en **APScheduler** para gestionar la ejecución de tareas críticas. además de esto, existen scripts auxiliares para generar reportes y preparar la configuración de WhatsApp.

El orquestador alterna la conectividad de red entre **WAN (Internet)** y **LAN (Red Local)** para ejecutar dos procesos principales:

**Tarea WAN**

- **Script:** `GlideExportBot.py`
- **Conexión:** *Wired connection 2* (D-Link)  

**Tarea LAN**

- **Script:** `ethernet_tasks.py`
- **Conexión:** *Wired connection 1* (eth0)  

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
    H --> J[Ethernet_tasks]
    
    I --> K[Notificaciones<br/>WhatsApp/Email]
    J --> K
```
## 1.4 Componentes Principales
### Timer.py

- **Tipo:** Orquestador
- **Descripción:** Coordina la ejecución de todos los procesos, gestión de red, reintentos y modo emergencia.
- **Ubicación:** `/home/rasp5/Desktop/BIBLIOCASTIA/NOOTEBOKS/timer.py`
- **Conexión:** WAN / LAN / Wi-Fi (Fallback)

### GlideExportBot.py

- **Tipo:** Bot de Exportación
- **Descripción:** Extrae datos desde Glide a través de Internet. 
- **Ubicación:** `/home/rasp5/Desktop/BIBLIOCASTIA/NOOTEBOKS/GlideExportBot.py`
- **Conexión:** WAN (Wired connection 2)

### Ethernet_tasks.py

- **Tipo:** Manipulación de Archivos
- **Descripción:** Procesa archivos en la red local/CIFS: transferir, organizar y convertir archivos de datos.
- **Ubicación:** `/home/rasp5/Desktop/BIBLIOCASTIA/NOOTEBOKS/ethernet_tasks.py`
- **Conexión:** LAN (Wired connection 1)

### Indicadores.py

- **Tipo:** Script Auxiliar
- **Descripción:** Consolida y reporta datos de mantenimiento. Convierte archivos CSV exportados en indicadores para la toma de decisiones.
- **Ubicación:** `/home/rasp5/Desktop/BIBLIOCASTIA/NOOTEBOKS/indicadores.py`
- **Conexión:** WAN / LAN

### Configwha.py

- **Tipo:** Script Auxiliar
- **Descripción:** Prepara y mantiene una sesión activa de WhatsApp Web en la máquina.
- **Ubicación:** `/home/rasp5/Desktop/BIBLIOCASTIA/NOOTEBOKS/configwha.py`
- **Conexión:** WAN

---

# 2. Orquestador principal - Timer.py

El script **timer.py** (Orquestador) es el componente central de control que se encarga de:

- Programar la ejecución automática de tareas (Glide y Ethernet) en horas predefinidas (5:00, 13:00 y 21:00).

- Gestionar la conectividad de red (WAN/LAN/Wi-Fi) de forma explícita antes de ejecutar cada tarea.

- Implementar una lógica de Modo de Emergencia para manejar fallas en las tareas o problemas de conexión a Internet.


## 2.1 Flujo de Ejecución del Ciclo

```mermaid
graph TD
    A[INICIO run_all] --> B[RESET: EMERGENCY_STATE = False]
    
    B --> C[FASE 1: WAN Internet]
    B --> D[FASE 2: LAN Red Local]
    
    %% FASE 1 - WAN
    C --> E[Conectar a Wired connection 2 D-Link]
    E --> F{¿Conexión exitosa?}
    F -->|Sí| G[Ejecutar GlideExportBot.py timeout: 300s]
    F -->|No| H[Fallback a Wi-Fi]
    H --> G
    
    G --> I{¿Ejecución exitosa?}
    I -->|Sí| J[ Éxito - Continuar]
    I -->|No| K[ MODO EMERGENCIA]
    K --> L[Cambiar a Wi-Fi únicamente]
    L --> M[Reintentar UNA VEZ]
    M --> N[Resultado: failure]
    
    J --> O[Resultado: success]
    
    %% FASE 2 - LAN
    D --> P[Conectar a Wired connection 1 eth0]
    P --> Q[Ejecutar ethernet_tasks.py timeout: 600s]
    Q --> R{¿Ejecución exitosa?}
    R -->|Sí| S[ Éxito - Continuar]
    R -->|No| T[ MODO EMERGENCIA]
    T --> U[Activar EMERGENCY_STATE = True]
    U --> V[ABORTAR resto de tareas LAN]
    V --> W[Resultado: failure]
    
    S --> X[Resultado: success]
    
    %% FASE 3 - CIERRE
    O --> Y[FASE 3: CIERRE]
    N --> Y
    X --> Y
    W --> Y
    
    Y --> Z{¿EMERGENCY_STATE activo?}
    Z -->|SÍ| AA[Dejar Wi-Fi activa]
    Z -->|NO| BB[Restaurar Internet normal D-Link/Wi-Fi]
    
    AA --> CC[FIN próximo ciclo según SCHEDULE_HOURS]
    BB --> CC

```

## 2.2 Modo de Emergencia
El Modo de Emergencia (EMERGENCY_STATE) se activa ante cualquier falla crítica.
- Fallo en tarea WAN (GlideExportBot.py)
- Fallo en tarea LAN (ethernet_tasks.py)
- Imposibilidad de conectar a LAN
- Excepción no controlada en el ciclo

**Activación (trigger_emergency):**

- Marca la bandera global EMERGENCY_STATE = True.

- Crea un archivo (/tmp/orchestrator.emergency) con la razón de la falla.

- Fuerza la conexión a Internet por Wi-Fi (ignorando D-Link y la LAN) para garantizar la comunicación mínima.

- Efecto: Una vez activo, el modo de emergencia influye en el reintento de tareas WAN, en la interrupción inmediata de tareas LAN y en el estado de red final del sistema.

## 2.2.1 **Impacto en el flujo**

### WAN (Internet)

**Sin emergencia**
- Utiliza la conexión D-Link.
- Si falla, hace fallback a Wi-Fi y realiza 1 reintento.

**Con emergencia**
- Usa únicamente Wi-Fi y también realiza 1 reintento.

### LAN (Red Local)

**Sin emergencia**
- Ejecuta todas las tareas programadas en la red local.

 **Con emergencia**
- Se abortan inmediatamente todas las tareas LAN restantes.

### Cierre del ciclo

**Sin emergencia**
- Se restablece el Internet normal (D-Link/Wi-Fi).

**Con emergencia**
- La máquina queda funcionando únicamente con Wi-Fi.

## 2.3 Inferfaz manual (GUI)

Dos modos de ejecución manual:

### **Ejecución Selectiva**
- Elige qué tareas correr (Glide y/o EthernetTasks)
- Respeta lógica de red y emergencia

### **Ejecución Completa Segura**
- Pausa el scheduler automático temporalmente
- Ejecuta ciclo completo **SIN** solaparse con automático
- Reactiva scheduler después

---
# 3. Glideexportbot.py - exportador WAN

Es un script de Python que realiza un proceso automatizado completo para extraer datos de Glide Apps, exportarlos a archivos CSV, organizarlos, comprimirlos en archivos ZIP y, finalmente, enviar esos ZIPs por correo electrónico (Gmail) y un resumen del proceso por WhatsApp Web, utilizando Selenium WebDriver para interactuar con la interfaz web.

## 3.1 Funcionalidades claves

### 3.1.1 Gestión de Red con Fallback (nmcli)

- Intenta conectarse a una conexión WAN cableada (**Wired connection 2**) varias veces.
- Si falla, prueba con las redes Wi-Fi conocidas como mecanismo de fallback.
- Preserva las conexiones PAN/Bluetooth durante los cambios de red, asegurando que no se desconecten.
- Incluye lógica para forzar servidores DNS públicos (**Cloudflare/Google**) si la resolución inicial falla.

### 3.1.2 Web Scraping y Automatización (Selenium)

- Automatiza el inicio de sesión en Glide enviando un correo al bot y extrayendo el enlace mágico (Enlace unico) del buzón de Gmail.
- Maneja el modal "This app is open in another tab" haciendo clic automáticamente en "Open app here".
- **Datos a exportar**:
**Plantas (18)**

```python
pythonplants = [
    "0002_GASCOL CENTRO", "0011_GASCOL SUR", "0021_G. LUX BOGOTÁ",
    "0079_BELLO", "0092_MEDELLÍN", "0111_MALAMBO",
    "0198_POSTOBÓN PEREIRA", "0381_G. LUX MONTERÍA", "0385_NEIVA",
    "0393_VILLAVICENCIO", "0395_G. LUX PASTO", 
    "0398_G. LUX VILLAVICENCIO", "0410_YUMBO", 
    "0422_G. LUX PIEDECUESTA", "0438_CÚCUTA",
    "0441_G. LUX DUITAMA", "0445_NGB", "0447_G. LUX VALLEDUPAR"
]
```
**Secciones por Planta (5)**
```python
pythonsections = [
    "AVISOM2",          # Avisos de mantenimiento tipo 2
    "ACTIVIDADES",      # Actividades de mantenimiento
    "INSPECCIONES",     # Inspecciones realizadas
    "AVISOM1",          # Avisos de mantenimiento tipo 1
    "Users"             # Usuarios del sistema
]
```

###  3.1.3Manejo de Archivos y Correo Electrónico (Gmail API)

- Utiliza la API de Gmail para la autenticación y el envío de correos electrónicos con archivos adjuntos.  
- Organiza los archivos .csv descargados en directorios específicos según la sección (AVISOS_M1, AVISOS_M2, ACTIVIDADES_INSPECCIONES, USERS).  
- Comprime los directorios en archivos .zip y los envía a una lista predefinida de destinatarios.

### 3.1.4 Notificación Final (WhatsApp Web)

- Genera un resumen detallado de las exportaciones exitosas y fallidas.  
- Intenta enviar este resumen a través de WhatsApp Web utilizando Selenium, pegando el contenido del portapapeles en un chat predefinido.

## 3.2 Ubicación de exportaciones

Después de la descarga, la función organizar_y_comprimir mueve y estructura estos archivos CSV en directorios basados en la fecha de ejecución y la sección del dato.

La estructura final se crea dentro del directorio EXPORTS dentro de Downloads:

Raíz de Exportación: /home/rasp5/Downloads/EXPORTS

Dentro de esa raíz, se crea una carpeta con la fecha actual (ej. 2025-11-18), y dentro de ella, los datos se separan por categoría:

| Tipo de Dato               | Directorio Final Organizado                                           |
|----------------------------|----------------------------------------------------------------------|
| ACTIVIDADES e INSPECCIONES | /home/rasp5/Downloads/EXPORTS/2025-11-18/ACTIVIDADES_INSPECCIONES   |
| AVISOM1                     | /home/rasp5/Downloads/EXPORTS/2025-11-18/AVISOS_M1                  |
| AVISOM2                     | /home/rasp5/Downloads/EXPORTS/2025-11-18/AVISOS_M2                  |
| Users                       | /home/rasp5/Downloads/EXPORTS/2025-11-18/USERS                      |


## 3.3 Flujo Completo del Proceso

```mermaid
graph TD
    A[INICIO main] --> B[PASO 1: PREPARACIÓN DE RED]
    
    B --> C[Asegurar PAN/Bluetooth ARRIBA]
    C --> D[nm_ensure_pan_up]
    
    D --> E[Conectar Internet con fallback]
    E --> F{Intentar Wired connection 2 D-Link}
    F -->|Éxito| G[Verificar DNS y conectividad IP]
    F -->|Falla| H[Probar perfiles Wi-Fi]
    H --> G
    
    G --> I{internet_by_ip y DNS resuelven?}
    I -->|Sí| J[ Red lista]
    I -->|No| K[Forzar DNS públicos 1.1.1.1, 8.8.8.8]
    K --> L[Reintentos con backoff hasta 5 intentos]
    L --> I
    
    J --> M[PASO 2: AUTENTICACIÓN GMAIL]
    M --> N{Cargar token.pickle}
    N -->|Existe y válido| O[Retorna gmail_service]
    N -->|No existe o expirado| P[OAuth flow con credentials.json]
    P --> Q[Guardar nuevo token]
    Q --> O
    
    O --> R[PASO 3: INICIAR SELENIUM]
    R --> S[Configurar Chrome Options]
    S --> T[Iniciar WebDriver]
    T --> U[WebDriverWait 30 segundos]
    
    U --> V[PASO 4: AUTENTICACIÓN EN GLIDE]
    V --> W[Navegar a https://go.glideapps.com]
    W --> X[Manejar advertencias de seguridad]
    X --> Y[Click Sign up with Email]
    Y --> Z[Ingresar SENDER_EMAIL]
    Z --> AA[Esperar correo de Glide Gmail API]
    AA --> BB[extract_link_from_email 10 intentos × 10s]
    BB --> CC[Obtener enlace mágico]
    CC --> DD[Abrir enlace en nueva pestaña]
    DD --> EE{Manejar modal Open app here?}
    EE -->|Aparece| FF[click_open_app_here_if_present]
    EE -->|No aparece| GG[Continuar]
    FF --> GG
    
    GG --> HH[PASO 5: EXPORTACIÓN POR PLANTA Loop 18 plantas]
    HH --> II[Para cada PLANTA]
    II --> JJ[Navegar a planta específica]
    JJ --> KK[Ir a pestaña Data]
    KK --> LL[Para cada SECCIÓN 5 secciones]
    
    LL --> MM[export_section driver, wait, section, plant]
    MM --> NN{Buscar panel section plant}
    NN -->|Encontrado| OO[Click botón Export]
    NN -->|No encontrado| PP[ Marcar fallo]
    OO --> QQ[Esperar descarga inicial]
    QQ --> RR[Click Select All]
    RR --> SS[Click Export final]
    SS --> TT[Esperar descarga completa]
    TT --> UU{wait_for_downloads sin .crdownload}
    UU -->|Éxito| VV[ Success]
    UU -->|Fallo| WW[Retry 2 intentos]
    WW --> OO
    
    VV --> XX[Registrar en summary_of_exports]
    PP --> XX
    XX --> YY{¿Todas las secciones?}
    YY -->|No| LL
    YY -->|Sí| ZZ[Contabilizar éxitos: success_count]
    
    ZZ --> AAA{success_count == 5?}
    AAA -->|Sí| BBB[plants_succeeded.append plant]
    AAA -->|No| CCC[plants_failed.append plant]
    
    BBB --> DDD[Volver al inicio de Glide]
    CCC --> DDD
    DDD --> EEE{¿Todas las plantas?}
    EEE -->|No| II
    EEE -->|Sí| FFF[FIN Loop de plantas]
    
    FFF --> GGG[PASO 6: ORGANIZACIÓN Y COMPRESIÓN]
    GGG --> HHH[organizar_y_comprimir gmail_svc]
    HHH --> III[Crear estructura de carpetas EXPORTS/YYYY-MM-DD/]
    III --> JJJ[Clasificar y mover CSV según nombre]
    JJJ --> KKK[Crear 4 archivos ZIP uno por carpeta]
    KKK --> LLL[Enviar correos con ZIP adjuntos]
    LLL --> MMM[_send_email_batch a cada RECIPIENT]
    MMM --> NNN[ Archivos organizados y enviados]
    
    NNN --> OOO[PASO 7: RESUMEN A WHATSAPP]
    OOO --> PPP[Preparar mensaje con resumen]
    PPP --> QQQ[plantas exitosas, fallidas, archivos]
    QQQ --> RRR[Copiar mensaje al portapapeles pyperclip]
    RRR --> SSS[Asegurar PAN/Bluetooth ARRIBA nuevamente]
    SSS --> TTT[Abrir WhatsApp Web perfil persistente]
    TTT --> UUU[Navegar a link de grupo/chat]
    UUU --> VVV[Escribir enviando datos ...]
    VVV --> WWW[Pegar mensaje Ctrl+V]
    WWW --> XXX[Enviar Enter]
    XXX --> YYY[Cerrar navegador WhatsApp]
    
    YYY --> ZZZ[FIN]
    ZZZ --> AAAA[Asegurar PAN/Bluetooth ARRIBA al finalizar]

```
---
# 4. Ethernet_tasks.py - procesador lan
Procesa los archivos CSV exportados por GlideExportBot.py, montando un recurso compartido CIFS en red local (LAN), copiando archivos, convirtiéndolos a Excel y organizándolos por fecha.

## 4.1 Funcionalidades claves

### 4.1.1 Gestión de Red

- **Bloqueo de Instancias (acquire_lock):** Asegura que solo una instancia del script se ejecute a la vez utilizando.

- **Asegurar PAN/Bluetooth:** Utiliza nmcli (NetworkManager) para asegurar que una conexión cableada (ensure_lan_for_cifs, PAN) o una conexión Bluetooth esté activa al inicio y al final, preservando dichas conexiones durante las operaciones de red.

- **Asegurar LAN (Cableada):** Llama a nm_ensure_pan_up para garantizar que el acceso remoto (Bluetooth/PAN) esté activo antes de modificar la red. Esto es crucial porque el montaje CIFS (Samba) requiere generalmente una conexión de red local y estable.

### 4.1.2 Montaje y Acceso al Servidor
Montar CIFS/Samba: Utiliza el comando sudo mount -t cifs para montar un recurso compartido de red (//sglimttoprod/compartida) en el punto de montaje local (/home/rasp5/compartida).

### 4.1.3 Procesamiento de Archivos

Copia de Directorios (copy_dirs): Recorre los directorios de exportación generados por el bot de Glide (por ejemplo, /home/rasp5/Downloads/EXPORTS/2025-11-18/AVISOS_M1) y copia los archivos a las rutas finales en el servidor:

- /home/rasp5/compartida/BIBLIOCASTIA/AVISOS M1/<fecha>

- /home/rasp5/compartida/BIBLIOCASTIA/AVISOS M2/<fecha>

- /home/rasp5/compartida/BIBLIOCASTIA/ACTIVIDADES_INSPECCIONES/<fecha>

Conversión CSV a Excel (convert_csv_to_excel): Utiliza la librería pandas para leer cada archivo .csv copiado y crear una copia idéntica en formato .xlsx dentro de la carpeta del servidor.

### 4.1.4 Limpieza y Restauración

- **Desmontar CIFS/Samba:** Una vez completada la transferencia y conversión, desmonta el recurso compartido (sudo umount /home/rasp5/compartida) para liberar el punto de montaje.

- **Restaurar Wi-Fi:** Intenta restaurar la conexión Wi-Fi con Internet activo (restore_wifi_priority) para dejar el sistema en el estado de red más común después de las tareas por cable.

### 4.2 Flujo Completo del Proceso
```mermaid
graph TD
graph TD
    A[INICIO finalize] --> B[PASO 1: PREPARACIÓN DE RED]
    
    B --> C[Asegurar PAN/Bluetooth ARRIBA]
    C --> D[nm_ensure_pan_up]
    
    D --> E[Conectar a red LAN cableada]
    E --> F[ensure_lan_for_cifs]
    F --> G[Buscar perfiles ethernet]
    G --> H[nm_discover_wired_connections]
    H --> I[Priorizar LAN_CANDIDATES]
    I --> J[Intentar cada conexión cableada]
    J --> K[nm_ensure_connection<br/>NO requiere Internet]
    K --> L{¿Conexión exitosa?}
    L -->|Sí| M[✅ Retorna nombre_conexión]
    L -->|No| N{¿Más candidatos?}
    N -->|Sí| J
    N -->|No| O[❌ None - ABORTAR RuntimeError]
    
    M --> P[PASO 2: MONTAJE DE RECURSO CIFS]
    P --> Q[mount_shared]
    Q --> R{¿Ya montado?<br/>os.path.ismount MOUNT_POINT}
    R -->|Sí| S[Desmontar forzado umount -l]
    R -->|No| T[Ejecutar comando montaje CIFS]
    S --> T
    
    T --> U{¿Montaje exitoso?}
    U -->|Sí| V[✅ Recurso compartido disponible]
    U -->|No| W[Reintentos con backoff exponencial]
    W --> X{¿Intentos < 5?}
    X -->|Sí| T
    X -->|No| Y[❌ RuntimeError]
    
    V --> Z[PASO 3: COPIA DE ARCHIVOS]
    Z --> AA[copy_dirs]
    AA --> BB[Obtener fecha de hoy: YYYY-MM-DD]
    BB --> CC[Para cada categoría en DIRS]
    
    CC --> DD{¿Existe origen?<br/>EXPORTS/fecha/categoría/}
    DD -->|No| EE[Saltar categoría]
    DD -->|Sí| FF[Crear carpeta destino]
    FF --> GG[Recorrer archivos en origen os.walk]
    GG --> HH{Filtrar: ¿archivo.endswith '.zip'?}
    HH -->|Sí| II[Omitir archivo ZIP]
    HH -->|No| JJ[shutil.copy2 origen, destino]
    II --> KK{¿Más archivos?}
    JJ --> KK
    
    KK -->|Sí| GG
    KK -->|No| LL{¿Más categorías?}
    LL -->|Sí| CC
    LL -->|No| MM[✅ Archivos CSV copiados a red]
    
    MM --> NN[PASO 4: CONVERSIÓN CSV → XLSX]
    NN --> OO[convert_csv_to_excel]
    OO --> PP[Para cada categoría en DIRS]
    PP --> QQ[Listar archivos .csv en ruta]
    QQ --> RR[Para cada archivo CSV]
    RR --> SS[Leer CSV con pandas pd.read_csv]
    SS --> TT[Extraer nombre sin extensión]
    TT --> UU[Crear archivo Excel df.to_excel]
    UU --> VV{¿Más archivos CSV?}
    VV -->|Sí| RR
    VV -->|No| WW{¿Más categorías?}
    WW -->|Sí| PP
    WW -->|No| XX[✅ Conversión completada]
    
    XX --> YY[PASO 5: DESMONTAJE SEGURO finally]
    YY --> ZZ[unmount_shared<br/>sudo umount /home/rasp5/compartida]
    
    ZZ --> AAA[PASO 6: RESTAURAR CONECTIVIDAD WI-FI]
    AAA --> BBB[restore_wifi_priority]
    BBB --> CCC[Buscar perfiles Wi-Fi]
    CCC --> DDD[nm_discover_wifi_connections]
    DDD --> EEE[Intentar cada Wi-Fi con Internet]
    EEE --> FFF[nm_ensure_connection<br/>REQUIERE Internet]
    FFF --> GGG{¿Wi-Fi con Internet?}
    GGG -->|Sí| HHH[Bajar conexiones excepto Wi-Fi activa]
    GGG -->|No| III[Mantener conexión actual]
    HHH --> JJJ[nm_deactivate_all_except<br/>Preserva PAN/Bluetooth]
    JJJ --> KKK[✅ Wi-Fi prioritaria con Internet]
    III --> KKK
    
    KKK --> LLL[PASO 7: ASEGURAR PAN/BLUETOOTH]
    LLL --> MMM[nm_ensure_pan_up]
    MMM --> NNN[FIN ✅<br/>Sistema listo para próximo ciclo]


```

