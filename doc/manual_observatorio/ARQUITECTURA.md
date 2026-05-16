# Arquitectura del Sistema RTS2 — Observatorio ETSIINF (UPM)

Este documento describe visualmente la arquitectura de demonios del sistema RTS2 y las conexiones físicas con el hardware del observatorio.

---

## 1. Arquitectura de Demonios (Software)

Todos los módulos de RTS2 son procesos independientes que se comunican entre sí a través del **demonio central** (`centrald`), que actúa como enrutador de mensajes en el puerto 8889.

```mermaid
graph TD
    subgraph PC_Observatorio["PC del Observatorio (Linux)"]
        CENTRALD["centrald\n(Puerto 8889)\nCerebro del sistema"]
        TELD["rts2-teld-lx200\nDemonio Telescopio"]
        DOME["rts2-dome\nDemonio Cúpula"]
        EXECD["execd\nDemonio de Ejecución"]
        DB[("PostgreSQL\nBase de datos")]
    end

    USER["Operador\n(rts2-mon)"]

    TELESCOPE["Meade LX200\n/dev/ttyUSB0\n(USB → RS232)"]
    DOME_HW["Talon 6\n/dev/ttyS0\n(Serie COM1)"]

    USER -->|"Interfaz ncurses"| CENTRALD
    CENTRALD <-->|"Mensajes internos"| TELD
    CENTRALD <-->|"Mensajes internos"| DOME
    CENTRALD <-->|"Mensajes internos"| EXECD
    EXECD -->|"Planes de observación"| CENTRALD
    CENTRALD --> DB

    TELD -->|"Comandos LX200 (RS232)"| TELESCOPE
    DOME -->|"Comandos Talon 6 (Serie)"| DOME_HW
```

---

## 2. Conexiones Físicas del Hardware

Diagrama de cómo el PC se conecta físicamente con los instrumentos del observatorio.

```mermaid
graph LR
    subgraph PC["PC Linux"]
        USB["Puerto USB"]
        COM1["Puerto Serie\n/dev/ttyS0"]
    end

    subgraph Adaptador["Adaptador"]
        CONV["USB → RS232"]
    end

    subgraph Instrumentos["Hardware"]
        LX200["Meade LX200\n(Telescopio)"]
        TALON["Talon 6\n(Control Cúpula)"]
        CUPULA["Cúpula motorizada"]
    end

    USB --> CONV
    CONV -->|"RS232 /dev/ttyUSB0"| LX200
    COM1 -->|"/dev/ttyS0"| TALON
    TALON --> CUPULA
```

---

## 3. Secuencia de Arranque del Sistema

El orden en que deben arrancarse los demonios es estricto: primero el enrutador, luego los dispositivos.

```mermaid
sequenceDiagram
    participant Op as Operador
    participant C as centrald
    participant T as rts2-teld-lx200
    participant D as rts2-dome
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

    Op->>C: 5. rts2-mon (interfaz de control)
    C-->>Op: Sistema listo para operar
```

---

## 4. Estado Actual del Hardware

| Componente | Modelo | Puerto (Windows) | Puerto (Linux) | Estado |
|---|---|---|---|---|
| Telescopio | Meade LX200 | `COM3` | `/dev/ttyUSB0` | Conectado |
| Cúpula | Talon 6 | `COM1` | `/dev/ttyS0` | Conectado |
| Cámara | Por determinar | — | — | Pendiente de instalación |
