# Parte c – Consola de mandos ESP32 para la movilidad libre del robot Baxter

Consola física de **3 pulsadores** con una **ESP32** que mueve el efector final del brazo **Baxter** simulado en PyBullet con movimiento fluido y **posicionamiento preciso**. El simulador dibuja la **estela** que deja el brazo y la **región de trabajo**, para visualizar la trayectoria.

> **c.** Desarrollar una consola de mandos con la ESP32 para un movimiento fluido del robot Baxter que permita una movilidad real.

Esta parte se centra solo en la **movilidad**: no hay objeto ni pinza. El agarre de un objeto está en [`../parte_b_baxter_agarre/`](../parte_b_baxter_agarre).

---

## 1. Objetivo

- Mover el efector final de Baxter en X, Y y Z de forma continua y suave.
- Permitir tanto desplazamientos amplios como ajustes finos de posición.
- Visualizar la trayectoria recorrida y la región de trabajo en la simulación.

## 2. Descripción del sistema

| Botón | Pin ESP32 | Función |
|---|---|---|
| MODO | GPIO16 | Pulsación corta: cambia el eje activo **X → Y → Z → X…** · Pulsación larga (≥ 0.8 s): alterna velocidad **NORMAL / FINA** |
| MENOS (−) | GPIO17 | Mantenido: disminuye la coordenada del eje activo |
| MÁS (+) | GPIO18 | Mantenido: la aumenta |
| MENOS + MÁS | — | Mantenidos 1 s: vuelve a la posición inicial y borra la estela |

- **Rampa de velocidad:** la velocidad arranca en 0.05 m/s y sube hasta 0.30 m/s (NORMAL) o 0.10 m/s (FINA) tras 1.5 s de pulsación continua.
- **LED integrado:** al cambiar de eje parpadea 1, 2 o 3 veces (X, Y, Z) y queda encendido fijo en velocidad FINA.
- **En la simulación:** una caja gris marca la región de trabajo, la estela cambia de color según la distancia recorrida y un texto muestra las coordenadas objetivo.

## 3. Hardware

- 1 × ESP32 DevKit y cable USB
- 3 × pulsadores y cables Dupont

Cada pulsador va entre su pin y **GND**; se usa `INPUT_PULLUP`, sin resistencias externas.

## 4. Arquitectura

```
ESP32                              PC (Python)                          PyBullet
┌─────────────────┐ POSE;x;y;z  ┌──────────────────────────┐   IK   ┌─────────────────┐
│ 3 pulsadores    │ ──────────► │ consola_libre_baxter.py  │ ─────► │ Brazo Baxter    │
│ rampa de        │ UART 115200 │ - interpreta mensajes    │        │ + caja de       │
│ velocidad       │ 20 Hz       │ - suaviza el objetivo    │        │   trabajo       │
│ modo fino       │ ──────────► │ - IK + motores           │        │ + estela de     │
└─────────────────┘ CLR         │ - dibuja la estela       │        │   colores       │
                                └──────────────────────────┘        └─────────────────┘
```

**Protocolo (texto plano)**

| Mensaje | Significado |
|---|---|
| `POSE;<x>;<y>;<z>` | Posición objetivo del efector final, en metros. Ej.: `POSE;0.200;0.050;-0.120` |
| `CLR` | La consola volvió a la posición inicial: el PC borra la estela |

## 5. Estructura del proyecto

```
parte_c_baxter_movilidad/
├── protocolo_baxter_c.py        # Interpretación de mensajes, suavizado y región de trabajo
├── consola_libre_baxter.py      # Puente ESP32 → Baxter en PyBullet
├── firmware/
│   └── consola_libre/consola_libre.ino
├── evidencias/
└── README.md
```

## 6. Cómo funciona el código

### 6.1 Firmware (`consola_libre.ino`)
1. El botón MODO distingue pulsación corta (cambiar eje) y larga (cambiar velocidad) con antirrebote de 40 ms.
2. Cada 50 ms (20 Hz) revisa MENOS y MÁS; mueve el eje activo `v · dt`, con `v` creciendo linealmente durante 1.5 s hasta el máximo del modo actual.
3. Recorta cada coordenada a los límites (X, Y: ±0.5 m; Z: −0.3 a 0.3 m).
4. Con MENOS y MÁS mantenidos 1 s restaura la pose inicial `(0.20, 0.00, −0.10)` y envía `CLR`.
5. El LED se maneja con `millis()`, sin bloquear la lectura de botones.

### 6.2 Python
- **`protocolo_baxter_c.py`**
  - `interpretar_linea()` distingue `POSE` y `CLR`, valida el formato y recorta la posición a la región de trabajo.
  - `suavizar()` aplica un filtro exponencial al objetivo para evitar saltos.
  - `aristas_caja()` calcula las 12 aristas de la región de trabajo para dibujarla.
- **`consola_libre_baxter.py`**
  1. Carga el URDF de Baxter y dibuja la caja guía.
  2. En cada ciclo: lee el serial, suaviza el objetivo, resuelve la **cinemática inversa** de todo el brazo con `calculateInverseKinematics` (iterativa) y mueve los motores.
  3. La clase `Estela` agrega un segmento cada 6 mm de avance del efector, con un color que cambia con la longitud recorrida; al detenerse muestra la longitud total.
  4. Actualiza en pantalla el texto con las coordenadas objetivo.


### Puesta en marcha

1. **Entorno:**
   ```bash
   conda create -n drones python=3.12 -y
   conda activate drones
   pip install pybullet pyserial numpy
   git clone https://github.com/erwincoumans/pybullet_robots.git
   ```
2. Copiar `consola_libre_baxter.py` y `protocolo_baxter_c.py` **dentro** de la carpeta `pybullet_robots`.
3. Cargar `consola_libre.ino` en la ESP32 y cablear los tres pulsadores.
4. Con el Monitor Serie **cerrado**:
   ```bash
   cd pybullet_robots
   python consola_libre_baxter.py
   ```
5. **Prueba sugerida:** recorrer cada eje de extremo a extremo; dibujar un cuadrado alternando X e Y; pasar a velocidad FINA y repetirlo; mantener MENOS + MÁS para reiniciar.

## Referencias
- [pybullet_robots](https://github.com/erwincoumans/pybullet_robots)
- [baxter_ik_demo.py](https://github.com/erwincoumans/pybullet_robots/blob/master/baxter_ik_demo.py)
