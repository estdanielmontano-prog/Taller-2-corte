# Taller Segundo Corte – Práctica real-to-sim con ESP32

Práctica de Micros en la que una **ESP32** funciona como consola de mando física y gobierna, por comunicación serial, robots simulados en **PyBullet** en tiempo real: un **dron** (parte a) y el brazo **Baxter** (partes b y c).

> **Enunciado:** desarrollar la práctica *real-to-sim* donde se debe:
> - **a.** Mover los drones de un lugar A a un lugar B y a un lugar C, con el control gestionado desde la ESP32. ([gym-pybullet-drones](https://github.com/utiasDSL/gym-pybullet-drones))
> - **b.** Desarrollar una consola de mandos con la ESP32 para un movimiento fluido del robot Baxter, con movilidad real de brazos y posicionamiento, y que pueda coger y mover un objeto. ([pybullet_robots](https://github.com/erwincoumans/pybullet_robots))
> - **c.** Desarrollar una consola de mandos con la ESP32 para un movimiento fluido del robot Baxter que permita una movilidad real.

---

## 1. Objetivo

- Separar la **decisión** (ESP32) de la **ejecución física** (simulador) en un flujo real-to-sim completo.
- Controlar un dron simulado por puntos de ruta definidos desde la ESP32.
- Manejar un brazo Baxter con una consola de pulsadores, con movimiento fluido y agarre de un objeto.

## 2. Descripción del sistema

| Carpeta | Parte | Qué hace |
|---|---|---|
| [`parte_a_dron/`](./parte_a_dron) | a | El botón BOOT de la ESP32 envía los puntos A → B → C; el dron simulado vuela a cada uno con control PID y avisa al llegar. |
| [`parte_b_baxter_agarre/`](./parte_b_baxter_agarre) | b | Consola de 3 pulsadores que mueve el efector final de Baxter (X, Y, Z), controla la pinza y lleva un cubo a una zona de destino. |
| [`parte_c_baxter_movilidad/`](./parte_c_baxter_movilidad) | c | Consola de 3 pulsadores centrada en la movilidad: ejes X, Y, Z con velocidad normal o fina, estela de trayectoria y región de trabajo dibujada. Sin pinza ni objeto. |

## 3. Hardware

| Parte | Componentes |
|---|---|
| a | 1 × ESP32 (se usan el botón BOOT y el LED integrado) |
| b | 1 × ESP32, 3 × pulsadores (GPIO32, GPIO33, GPIO25 a GND) |
| c | 1 × ESP32, 3 × pulsadores (GPIO16, GPIO17, GPIO18 a GND) |

Ambas partes se conectan al PC con cable USB, que alimenta la placa y transporta el serial.

## 4. Arquitectura

Las dos partes siguen el mismo patrón: **ESP32 (entrada y decisión) → UART/USB → Python → PyBullet (ejecución).**

```
ESP32                       PC (Python)                      PyBullet
┌─────────────┐  UART USB   ┌──────────────────┐             ┌──────────────┐
│ Pulsadores  │ ──────────► │ Script de control│ ──────────► │ Dron o brazo │
│ (decide)    │ texto plano │ (interpreta y    │  PID / IK   │ Baxter       │
└─────────────┘             │  comanda)        │             └──────────────┘
```

| | Parte a (dron) | Parte b (Baxter + agarre) | Parte c (Baxter + movilidad) |
|---|---|---|---|
| Mensaje | Evento discreto `GOTO <punto> x y z` | Posición continua `POSE;x;y;z;pinza` a 20 Hz | Posición continua `POSE;x;y;z` a 20 Hz y `CLR` |
| Movimiento fluido | La referencia avanza a velocidad limitada y el PID la persigue | Filtro del objetivo + cinemática inversa en cada ciclo | Igual que b, más velocidad normal/fina |
| Elemento distintivo | Aviso `LLEGO` a la ESP32 | Pinza proporcional y agarre del cubo | Estela de colores y caja de trabajo |
| Botones | Botón BOOT | MODO (X/Y/Z/PINZA), MENOS, MÁS | MODO (X/Y/Z, larga = velocidad), MENOS, MÁS |

Ambas partes recortan los valores recibidos a límites de seguridad en el PC, para que una lectura errónea no mande al robot a una posición inválida.

## 5. Estructura del proyecto

```
Taller_Real_to_Sim/
├── parte_a_dron/
│   ├── protocolo_dron.py
│   ├── simulacion_sin_esp32.py
│   ├── control_dron_esp32.py
│   ├── firmware/dron_waypoints/dron_waypoints.ino
│   ├── evidencias/
│   └── README.md
├── parte_b_baxter_agarre/
│   ├── protocolo_baxter.py
│   ├── consola_baxter.py
│   ├── firmware/consola_baxter/consola_baxter.ino
│   ├── evidencias/
│   └── README.md
├── parte_c_baxter_movilidad/
│   ├── protocolo_baxter_c.py
│   ├── consola_libre_baxter.py
│   ├── firmware/consola_libre/consola_libre.ino
│   ├── evidencias/
│   └── README.md
└── README.md
```

Cada subcarpeta tiene su propio README con el detalle de esa parte (cuatro README en total, contando este).

## 6. Cómo funciona el código

- **Separación protocolo/simulación:** en cada parte, la lectura y validación de mensajes y el suavizado están en un módulo aparte (`protocolo_*.py`) sin dependencias del simulador, por lo que se pueden probar sin instalar PyBullet.
- **Firmware sin bloqueos:** ambos sketches usan `millis()` en lugar de `delay()` para atender botones, serial y LED al mismo tiempo.
- **Parte a:** la ESP32 elige el punto y Python lo suaviza antes de entregarlo al PID de `gym-pybullet-drones`.
- **Parte b:** la ESP32 integra el movimiento (rampa de velocidad y límites) y Python resuelve la IK y simula el agarre con un acople rígido temporal.
- **Parte c:** misma base de movimiento, sin pinza; Python dibuja la estela del efector y la región de trabajo.

### Paso a paso resumido
1. Crear el entorno: `conda create -n drones python=3.12 -y` y `conda activate drones`.
2. Parte a: instalar `gym-pybullet-drones` y `pyserial`. Partes b y c: `pip install pybullet pyserial numpy` y clonar `pybullet_robots` (copiar allí los scripts de Python).
3. Cargar en la ESP32 el sketch de la parte correspondiente.
4. Cerrar el Monitor Serie y ejecutar el script de Python de esa parte.

## 7. Evidencias

🎥 **Parte a – dron:** *(pega aquí el enlace del video)*

🎥 **Parte b – Baxter con agarre:** *(pega aquí el enlace del video)*

🎥 **Parte c – Baxter, movilidad:** *(pega aquí el enlace del video)*

## Referencias
- [gym-pybullet-drones](https://github.com/utiasDSL/gym-pybullet-drones)
- [pybullet_robots](https://github.com/erwincoumans/pybullet_robots)
- [baxter_ik_demo.py](https://github.com/erwincoumans/pybullet_robots/blob/master/baxter_ik_demo.py)
