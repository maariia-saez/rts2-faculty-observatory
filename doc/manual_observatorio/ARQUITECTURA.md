# Arquitectura del Sistema RTS2 — Observatorio ETSIINF (UPM)

Este documento describe visualmente la arquitectura completa del sistema RTS2 y las conexiones físicas con el hardware del observatorio.

---

## 1. Arquitectura General de Demonios

Todos los módulos de RTS2 son procesos independientes que se comunican entre sí a través del **demonio central** (`centrald`), que actúa como enrutador de mensajes en el puerto 8889.

```mermaid
graph TD
    subgraph PC_Observatorio["PC del Observatorio (Linux)"]
        CENTRALD["centrald\n(Puerto 8889)\nEnrutador central"]

        subgraph Dispositivos["Demonios de Hardware"]
            TELD["rts2-teld-lx200\nTelescopio"]
            DOME["rts2-dome\nCúpula"]
            CAMD["rts2-camd\nCámara CCD"]
            FOCUSD["rts2-focusd\nFocalizador"]
            FILTERD["rts2-filterd\nRueda de filtros"]
            SENSORD["rts2-sensord\nSensores meteo"]
        end

        subgraph Automatizacion["Demonios de Automatización"]
            EXECD["execd\nEjecución de planes"]
            SCHEDULER["rts2-scheduler\nPlanificador"]
            MOODD["moodd\nEstado del sistema"]
        end

        subgraph Acceso["Interfaces de Acceso"]
            HTTPD["rts2-httpd\nAPI HTTP/JSON"]
            LOGGER["rts2-logger\nRegistro de eventos"]
        end

        DB[("PostgreSQL\nBase de datos")]
    end

    USER_MON["Operador\n(rts2-mon)"]
    USER_WEB["Cliente Web\n(navegador)"]

    TELESCOPE["Meade LX200\n/dev/ttyUSB0"]
    DOME_HW["Talon 6\n/dev/ttyS0"]
    CAMERA_HW["Cámara\n(pendiente)"]

    USER_MON -->|"ncurses TCP"| CENTRALD
    USER_WEB -->|"HTTP / JSON-RPC"| HTTPD
    HTTPD <-->|"Mensajes internos"| CENTRALD

    CENTRALD <-->|"TCP interno"| TELD
    CENTRALD <-->|"TCP interno"| DOME
    CENTRALD <-->|"TCP interno"| CAMD
    CENTRALD <-->|"TCP interno"| FOCUSD
    CENTRALD <-->|"TCP interno"| FILTERD
    CENTRALD <-->|"TCP interno"| SENSORD
    CENTRALD <-->|"TCP interno"| EXECD
    CENTRALD <-->|"TCP interno"| SCHEDULER
    CENTRALD <-->|"TCP interno"| MOODD
    CENTRALD <-->|"TCP interno"| LOGGER

    EXECD -->|"Planes de observación"| CENTRALD
    SCHEDULER -->|"Colas de targets"| DB
    CENTRALD --> DB
    LOGGER --> DB

    TELD -->|"LX200 RS232"| TELESCOPE
    DOME -->|"Protocolo serie"| DOME_HW
    CAMD -.->|"Pendiente"| CAMERA_HW
```

---

## 2. Capas de Software (Arquitectura en Capas)

```mermaid
graph BT
    subgraph HW["Capa Hardware"]
        H1["Telescopio\n(LX200)"]
        H2["Cúpula\n(Talon 6)"]
        H3["Cámara CCD"]
        H4["Focalizador"]
        H5["Sensores"]
    end

    subgraph DRV["Capa Demonios de Dispositivo"]
        D1["rts2-teld-lx200"]
        D2["rts2-dome"]
        D3["rts2-camd-*"]
        D4["rts2-focusd-*"]
        D5["rts2-sensord-*"]
    end

    subgraph CORE["Capa Central (centrald)"]
        C1["centrald\nEnrutamiento de mensajes\nGestión de estado global\nControl de modos"]
    end

    subgraph AUTO["Capa Automatización"]
        A1["execd\nEjecución"]
        A2["rts2-scheduler\nPlanificación"]
        A3["moodd\nEstado noche"]
    end

    subgraph LIBS["Bibliotecas C++ (lib/)"]
        L1["librts2\nClases base"]
        L2["librts2db\nAcceso BD"]
        L3["librts2tel\nAstrometría"]
        L4["librts2script\nScripting"]
        L5["librts2fits\nFITS I/O"]
    end

    subgraph ACCESO["Capa de Acceso"]
        I1["rts2-mon\nTUI ncurses"]
        I2["rts2-httpd\nAPI HTTP/JSON"]
        I3["Python3 API\nrts2.py"]
    end

    HW --> DRV
    DRV --> CORE
    AUTO --> CORE
    LIBS --> DRV
    LIBS --> AUTO
    LIBS --> CORE
    ACCESO --> CORE
```

---

## 3. Conexiones Físicas del Hardware

```mermaid
graph LR
    subgraph PC["PC Linux"]
        USB["Puerto USB"]
        COM1["Puerto Serie\n/dev/ttyS0"]
        ETH["Red Ethernet"]
    end

    subgraph Adaptadores["Adaptadores"]
        CONV["Adaptador\nUSB → RS232\n(/dev/ttyUSB0)"]
    end

    subgraph Instrumentos["Hardware del Observatorio"]
        LX200["Meade LX200\n(Telescopio)"]
        TALON["Talon 6\n(Control Cúpula)"]
        CUPULA["Cúpula motorizada"]
        CAM["Cámara CCD\n(pendiente)"]
    end

    USB --> CONV
    CONV -->|"RS232\n9600 bps"| LX200
    COM1 -->|"Serie\n/dev/ttyS0"| TALON
    TALON --> CUPULA
    ETH -.->|"USB/FireWire\n(futuro)"| CAM
```

---

## 4. Flujo de una Observación Automática

```mermaid
sequenceDiagram
    participant S as rts2-scheduler
    participant DB as PostgreSQL
    participant E as execd
    participant C as centrald
    participant T as rts2-teld
    participant D as rts2-dome
    participant CAM as rts2-camd

    S->>DB: Consulta targets disponibles
    DB-->>S: Lista de targets ordenados
    S->>E: Asigna siguiente target
    E->>C: Solicita mover telescopio
    C->>T: SET ra/dec del target
    T->>T: Mueve montura (LX200)
    T-->>C: Estado: tracking

    E->>C: Solicita abrir cúpula
    C->>D: CMD open
    D->>D: Abre Talon 6
    D-->>C: Estado: open

    E->>C: Solicita exposición
    C->>CAM: EXPOSE tiempo/filtro
    CAM->>CAM: Captura imagen
    CAM-->>C: FITS guardado

    C->>DB: Registro de observación
    E->>S: Target completado
```

---

## 5. Secuencia de Arranque del Sistema

El orden en que deben arrancarse los demonios es estricto: primero el enrutador, luego los dispositivos.

```mermaid
sequenceDiagram
    participant Op as Operador
    participant C as centrald
    participant T as rts2-teld-lx200
    participant D as rts2-dome
    participant E as execd
    participant DB as PostgreSQL

    Op->>DB: 1. systemctl start postgresql
    DB-->>Op: Base de datos lista

    Op->>C: 2. rts2-centrald
    C-->>Op: Escuchando en puerto 8889

    Op->>T: 3. rts2-teld-lx200 -d /dev/ttyUSB0
    T->>C: Registro en centrald
    C-->>T: Conexión establecida

    Op->>D: 4. rts2-dome -d /dev/ttyS0
    D->>C: Registro en centrald
    C-->>D: Conexión establecida

    Op->>E: 5. rts2-executor (opcional)
    E->>C: Registro en centrald
    C-->>E: Conexión establecida

    Op->>C: 6. rts2-mon (interfaz de control)
    C-->>Op: Sistema listo para operar
```

---

## 6. Modos de Operación del Sistema

`moodd` controla el estado global que regula qué acciones están permitidas.

```mermaid
stateDiagram-v2
    [*] --> DAY : Arranque del sistema
    DAY --> DUSK : Puesta de sol
    DUSK --> NIGHT : Oscuridad astronómica
    NIGHT --> DAWN : Amanecer astronómico
    DAWN --> DAY : Salida del sol

    NIGHT --> STANDBY : Mal tiempo / lluvia
    STANDBY --> NIGHT : Tiempo despejado

    DAY : DAY\nOperación diurna\nCúpula cerrada
    DUSK : DUSK\nPreparación\nCalibración
    NIGHT : NIGHT\nObservación automática\nEjecución de planes
    DAWN : DAWN\nFin de observación\nFlats de amanecer
    STANDBY : STANDBY\nEmergencia meteorológica\nCierre de seguridad
```

---

## 7. Estado Actual del Hardware

| Componente | Modelo | Puerto (Windows) | Puerto (Linux) | Estado |
|---|---|---|---|---|
| Telescopio | Meade LX200 | `COM3` | `/dev/ttyUSB0` | Conectado |
| Cúpula | Talon 6 | `COM1` | `/dev/ttyS0` | Conectado |
| Cámara | Por determinar | — | — | Pendiente de instalación |
| Focalizador | Por determinar | — | — | Pendiente |
| Sensor meteo | Por determinar | — | — | Pendiente |
