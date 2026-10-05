# Parte a – Dron simulado controlado desde una ESP32 (real-to-sim)

Una **ESP32** actúa como control remoto: decide a qué punto debe volar un cuadricóptero y se lo comunica por **UART (USB)** a un script en **Python**, que lo entrega como objetivo al controlador **PID** del simulador [`gym-pybullet-drones`](https://github.com/utiasDSL/gym-pybullet-drones).

> **Enunciado (a):** mover los drones de un lugar A a un lugar B y a un lugar C, teniendo presente que el control se gestionará desde la ESP32.

---

## 1. Objetivo

- Instalar y validar un entorno de simulación de drones basado en PyBullet.
- Establecer una comunicación serial bidireccional entre la ESP32 y Python.
- Recorrer en vuelo los puntos A → B → C, con la secuencia gobernada por eventos generados en la ESP32.

## 2. Descripción del sistema

| Punto | x (m) | y (m) | z (m) |
|---|---|---|---|
| A | 0.0 | 0.0 | 1.0 |
| B | 1.5 | 1.0 | 1.2 |
| C | 2.5 | −0.5 | 0.8 |

Controles en la ESP32 (botón **BOOT**, GPIO0):

| Acción | Resultado |
|---|---|
| Pulsación corta | Envía el siguiente punto de la ruta (A → B → C → A…) |
| Pulsación larga (≥ 1 s) | Regresa al punto A |
| Llega un mensaje `LLEGO <punto>` del PC | El LED integrado parpadea 3 veces |

## 3. Hardware

- 1 × ESP32 DevKit y cable USB.
- No requiere componentes externos: se usan el botón **BOOT** (GPIO0) y el LED azul integrado (GPIO2).

## 4. Arquitectura

```
ESP32                           PC (Python)                      Simulador
┌──────────────┐  GOTO B 1.5 1.0 1.2   ┌────────────────────┐    ┌──────────────┐
│ Botón BOOT   │ ────────────────────► │ control_dron_esp32 │───►│ Dron CF2X    │
│ (elige punto)│  UART 115200          │ - suaviza objetivo │PID │ gym-pybullet │
│ LED          │ ◄──────────────────── │ - detecta llegada  │    │ -drones      │
└──────────────┘     LLEGO B           └────────────────────┘    └──────────────┘
```

**Protocolo (texto plano, una línea por mensaje)**

| Sentido | Formato | Ejemplo |
|---|---|---|
| ESP32 → PC | `GOTO <nombre> <x> <y> <z>` | `GOTO B 1.5 1.0 1.2` |
| PC → ESP32 | `LLEGO <nombre>` | `LLEGO B` |

## 5. Estructura del proyecto

```
parte_a_dron/
├── protocolo_dron.py            # Interpretación de mensajes y trayectoria suave
├── simulacion_sin_esp32.py      # Valida el simulador: A → B → C automático
├── control_dron_esp32.py        # Puente ESP32 → simulador en tiempo real
├── firmware/
│   └── dron_waypoints/dron_waypoints.ino
├── evidencias/
└── README.md
```

## 6. Cómo funciona el código

### 6.1 Firmware (`dron_waypoints.ino`)
1. Lee el botón BOOT con antirrebote de 40 ms; al soltar tras una pulsación corta envía el siguiente punto.
2. Si el botón se mantiene 1 s, envía el punto A y reinicia la secuencia.
3. Lee el serial sin bloquear; al recibir `LLEGO` hace parpadear el LED con `millis()` (sin `delay()`).

### 6.2 Python
- **`protocolo_dron.py`**
  - `interpretar_linea()` valida el formato y **recorta** el destino a límites de seguridad (|x|, |y| ≤ 3 m; 0.2 ≤ z ≤ 2.5 m).
  - `avanzar_objetivo()` desplaza la referencia hacia el destino a velocidad limitada (0.6 m/s), para que el PID no reciba saltos bruscos de varios metros.
- **`simulacion_sin_esp32.py`:** recorre A, B y C solo, esperando 2 s en cada punto. Sirve para comprobar la instalación antes de conectar la placa.
- **`control_dron_esp32.py`:**
  1. Detecta el puerto de la ESP32 por la descripción del chip USB-serial (o se indica con `--puerto`).
  2. En cada paso de simulación lee el serial, actualiza el destino, avanza la referencia y calcula las órdenes de los motores con `DSLPIDControl`.
  3. Cuando el dron queda a menos de 10 cm del destino responde `LLEGO <punto>`.

## 7. Evidencias

🎥 **Video de funcionamiento:** *(pega aquí el enlace del video)*

### Puesta en marcha

1. **Entorno** (Python ≥ 3.12; se recomienda Miniconda):
   ```bash
   conda create -n drones python=3.12 -y
   conda activate drones
   git clone https://github.com/utiasDSL/gym-pybullet-drones.git
   cd gym-pybullet-drones && pip install -e . && pip install pyserial && cd ..
   ```
2. **Validar el simulador:** `python simulacion_sin_esp32.py`
3. **Cargar el firmware** `dron_waypoints.ino` en la ESP32 (placa *ESP32 Dev Module*).
4. **Cerrar el Monitor Serie** y ejecutar `python control_dron_esp32.py`.
5. Pulsar BOOT para avanzar de punto; mantenerlo 1 s para volver a A.

## Referencias
- [gym-pybullet-drones](https://github.com/utiasDSL/gym-pybullet-drones)
