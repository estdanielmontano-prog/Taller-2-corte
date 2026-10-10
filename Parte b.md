# Parte b – Consola de mandos ESP32 para Baxter: movimiento y agarre de un objeto

Consola física de **3 pulsadores** construida con una **ESP32** que mueve en tiempo real el brazo del robot **Baxter** simulado en PyBullet (cinemática inversa) y controla su pinza para **coger y mover un objeto**.

> **b.** Desarrollar una consola de mandos con la ESP32 para un movimiento fluido del robot Baxter que permita una movilidad real de brazos y posicionamiento, además de que el robot pueda coger y mover un objeto.

Esta parte añade, sobre el movimiento fluido, el **agarre de un objeto** con la pinza. La consola de movimiento sin objeto está en [`../parte_c_baxter_movilidad/`](../parte_c_baxter_movilidad).

---

## 1. Objetivo

- Mover el efector final de Baxter en X, Y y Z con movimiento continuo y suave.
- Abrir y cerrar la pinza de forma proporcional desde la misma consola.
- Coger un cubo y llevarlo a una zona de destino marcada en la escena.

## 2. Descripción del sistema

| Botón | Pin ESP32 | Función |
|---|---|---|
| MODO | GPIO32 | Pulsación corta: cambia el modo **X → Y → Z → PINZA → X…** |
| MENOS (−) | GPIO33 | Mantenido: disminuye el valor del modo activo (en PINZA: **cierra**) |
| MÁS (+) | GPIO25 | Mantenido: lo aumenta (en PINZA: **abre**) |
| MENOS + MÁS | — | Mantenidos 1 s: vuelve a la posición inicial y abre la pinza |

- **Aceleración progresiva:** en los ejes X, Y, Z la velocidad sube de 0.06 a 0.30 m/s mientras se mantiene el botón, así se puede ir rápido y luego ajustar con precisión.
- **Pinza proporcional:** la apertura va de 0 % (cerrada) a 100 % (abierta).
- **LED integrado:** encendido mientras hay movimiento.

## 3. Hardware

- 1 × ESP32 DevKit y cable USB
- 3 × pulsadores y cables Dupont

Cada pulsador va entre su pin y **GND**; se usa `INPUT_PULLUP`, por lo que no se necesitan resistencias externas.


## 4. Arquitectura

```
ESP32                              PC (Python)                         PyBullet
┌─────────────────┐ POSE;x;y;z;g ┌───────────────────────┐   IK    ┌────────────────┐
│ 3 pulsadores    │ ───────────► │ consola_baxter.py     │ ──────► │ Brazo Baxter   │
│ rampa de        │  UART 115200 │ - interpreta mensaje  │         │ + pinza        │
│ velocidad       │  20 Hz       │ - suaviza el objetivo │         │ + cubo         │
└─────────────────┘              │ - IK + motores        │         │ + zona destino │
                                 │ - acople del agarre   │         └────────────────┘
                                 └───────────────────────┘
```

**Protocolo:** `POSE;<x>;<y>;<z>;<pinza>` — posición en metros y apertura 0–100. Ejemplo: `POSE;0.200;0.050;-0.120;100`.

## 5. Estructura del proyecto

```
parte_b_baxter_agarre/
├── protocolo_baxter.py          # Interpretación de mensajes, suavizado y pinza
├── consola_baxter.py            # Puente ESP32 → Baxter en PyBullet
├── firmware/
│   └── consola_baxter/consola_baxter.ino
├── evidencias/
└── README.md
```

## 6. Cómo funciona el código

### 6.1 Firmware (`consola_baxter.ino`)
1. El botón MODO usa antirrebote y avanza cíclicamente entre cuatro modos.
2. Cada 50 ms (20 Hz) revisa MENOS y MÁS: en los ejes suma o resta `v · dt`, con `v` creciendo linealmente durante 1.5 s de pulsación continua; en el modo PINZA cambia la apertura a 60 %/s.
3. Recorta cada valor a los límites de trabajo (X, Y: ±0.5 m; Z: −0.3 a 0.3 m).
4. Si MENOS y MÁS se mantienen 1 s, restaura la pose inicial `(0.20, 0.00, −0.10)` y abre la pinza.
5. Envía `POSE;x;y;z;pinza` en cada ciclo.

### 6.2 Python
- **`protocolo_baxter.py`**
  - `interpretar_pose()` valida la línea y recorta la posición (±0.6 m en X, Y; −0.5 a 0.5 m en Z) y la apertura (0–100).
  - `suavizar()` aplica un filtro exponencial al objetivo para evitar saltos en el brazo.
  - `apertura_a_posicion()` convierte el porcentaje en posición de los dedos.
- **`consola_baxter.py`**
  1. Carga el URDF de Baxter (`toms_baxter.urdf`), una mesa gris con el cubo y una **zona verde de destino**.
  2. En cada ciclo: lee el serial, suaviza el objetivo, resuelve la **cinemática inversa** con `calculateInverseKinematics` (iterativa, sin tocar los dedos), mueve los motores y fija la apertura de la pinza.
  3. **Agarre:** si la apertura baja de 25 % con el efector a menos de 6 cm del cubo, crea un acople rígido (`createConstraint`) que conserva la pose relativa; se libera al superar 50 % de apertura. Se usa el acople porque el brazo se mueve por IK y la fricción de los dedos no basta para sostener el cubo.
  4. Avisa por consola cuando el cubo es sujetado, soltado y cuando queda depositado en la zona destino.

## 7. Evidencias

🎥 **Video de funcionamiento:** *(https://drive.google.com/drive/u/0/folders/1ABYW4ooAyx6Ivvj62gKILjlh1gHZZKSv)*

### Puesta en marcha

1. **Entorno:**
   ```bash
   conda create -n drones python=3.12 -y
   conda activate drones
   pip install pybullet pyserial numpy
   git clone https://github.com/erwincoumans/pybullet_robots.git
   ```
2. Copiar `consola_baxter.py` y `protocolo_baxter.py` **dentro** de la carpeta `pybullet_robots` (el URDF se busca con la ruta relativa `baxter_common/baxter_description/urdf/toms_baxter.urdf`).
3. Cargar `consola_baxter.ino` en la ESP32 y cablear los tres pulsadores.
4. Con el Monitor Serie **cerrado**:
   ```bash
   cd pybullet_robots
   python consola_baxter.py
   ```
5. **Secuencia sugerida:** modo X/Y hasta quedar sobre el cubo → modo Z para bajar unos 8 cm → modo PINZA y MENOS para cerrar → subir en Z → desplazar hacia la zona verde → modo PINZA y MÁS para soltar.

## Referencias
- [pybullet_robots](https://github.com/erwincoumans/pybullet_robots)
- [baxter_ik_demo.py](https://github.com/erwincoumans/pybullet_robots/blob/master/baxter_ik_demo.py)
