# agents.md — Naturkeller (Codex Project Template, AM2301A sensors)

> **Goal**: Template to define the participating agents/services for the Naturkeller ventilation project based on Shelly Plus 1PM Gen3 and AM2301A (DHT21) sensors. This document serves as a starting point for a Codex/repo setup (monorepo or multi-repo) and describes roles, interfaces, configuration, runtime behavior, and operational processes.

**Architecture**
Use different timer loops for a heartbeat management with a configurable log, duty cycle handling for long switch on periods, which can be configured.

---

## 1. Overview

- **Problem**: Humidity and temperature-controlled ventilation of a natural cellar (beverages/potatoes), avoiding condensation/mold, protecting against overdrying at low temperatures.
- **Approach**: Two-sensor strategy with **indoor AM2301A** and **outdoor AM2301A**. Decision based on relative humidity, temperature difference, and safety conditions (frost limit, potato protection). Fallback window if no outdoor data is available.
- **Topology**:
  - **Outdoor Shelly** (with Add-on + AM2301A, provides outdoor values via `Shelly.GetStatus`)
  - **Cellar Shelly (controller)** (with Add-on + AM2301A, controls fan/relay, reads indoor values locally, fetches outdoor values from the outdoor Shelly)

---

## 2. Agents & Responsibilities

### 2.1 Agent: Outdoor Sensor Shelly
- **Role**: Provides outdoor temperature/humidity values via RPC.
- **Inputs**: AM2301A sensor (Add-on).
- **Outputs**:  
  - `temperature:* { tC }`  
  - `humidity:* { rh }`  
- **Non-goals**: No decisions, no relay switching.

### 2.2 Agent: Cellar Controller Shelly
- **Role**: Core control logic; switches the relay (fan) based on indoor and outdoor state.
- **Inputs**:
  - Indoor: AM2301A values (`temperature:*`, `humidity:*`) from local Add-on
  - Outdoor: HTTP GET to Outdoor Shelly `/rpc/Shelly.GetStatus`
- **Outputs**: `Switch.Set` (relay ON/OFF), logs.
- **Safeguards**: Hysteresis, ΔT condition, frost limit, potato protection, Min-Off/Max-On, PowerGuard.

### 2.3 Agent: Diagnostics (optional)
- **Role**: Helper scripts for debugging and testing (e.g. show both indoor and outdoor readings, decision log).
- **Inputs/Outputs**: read-only; writes logs.

---

## 3. Interfaces

### 3.1 RPC / HTTP
- **Outdoor Status**:  
  `GET http://<outdoorShellyHost>/rpc/Shelly.GetStatus`  
  - **Relevant Schema**:
    ```json
    {
      "temperature:<id>": { "tC": <number> },
      "humidity:<id>":    { "rh": <number> }
    }
    ```
- **Indoor Status**:  
  `Shelly.GetStatus` on the cellar Shelly (local Add-on AM2301A).
- **Relay Control**:  
  `Switch.Set { id: <int>, on: <bool> }`

### 3.2 Normalized Payloads
- **Outdoor values**:
  ```json
  {
    "Tout": 21.3,
    "RHout": 68.0,
    "ABSout": 12.67,
    "fetchTs": 1755904181
  }
