# Sistema de adquisición EEG de 16 canales con OpenBCI Cyton + Daisy

Sistema de adquisición y almacenamiento de señales electroencefalográficas (EEG) utilizando un **OpenBCI Cyton con expansión Daisy**, conectado a una **Raspberry Pi** mediante comunicación serial y controlado mediante **Python**.

El proyecto permite adquirir datos de **16 canales EEG**, almacenarlos en un archivo CSV y generar una representación gráfica de las señales adquiridas.

---

## 1. Descripción del proyecto

El objetivo del proyecto es desarrollar un sistema capaz de realizar la adquisición de señales EEG mediante hardware OpenBCI y procesarlas utilizando una Raspberry Pi.

El sistema realiza las siguientes tareas:

1. Inicializa la comunicación con el OpenBCI.
2. Configura el sistema de adquisición.
3. Inicia el streaming de datos.
4. Lee los paquetes enviados por el dispositivo.
5. Extrae los datos correspondientes a los 16 canales.
6. Convierte los datos de 24 bits a valores enteros con signo.
7. Organiza las muestras en una matriz de 16 canales.
8. Guarda los datos en formato CSV.
9. Convierte los valores a microvoltios utilizando el factor definido en el programa.
10. Genera una gráfica de los 16 canales.

---

## 2. Arquitectura general

```text
                 ACTIVIDAD EEG
                      │
                      ▼
                  ELECTRODOS
                      │
                      ▼
             ┌─────────────────┐
             │  OpenBCI Cyton  │
             │   Canales 1–8   │
             └────────┬────────┘
                      │
                ┌─────▼─────┐
                │   Daisy   │
                │ Canales 9–16│
                └─────┬─────┘
                      │
                      │ Datos digitales
                      ▼
             ┌─────────────────┐
             │   Raspberry Pi  │
             │     Python      │
             └────────┬────────┘
                      │
                      ▼
              Lectura serial
                      │
                      ▼
             Procesamiento de
                  paquetes
                      │
                      ▼
              16 canales EEG
                  │       │
                  │       │
                  ▼       ▼
                CSV    Gráfica
```

---

## 3. Hardware utilizado

- OpenBCI Cyton.
- OpenBCI Daisy.
- Electrodos EEG.
- Raspberry Pi.
- Cable/conexión USB para comunicación serial.
- Fuente de alimentación correspondiente al sistema.

### Configuración

La combinación **Cyton + Daisy** permite trabajar con un total de **16 canales EEG**.

Conceptualmente:

```text
Cyton
├── Canal 1
├── Canal 2
├── Canal 3
├── Canal 4
├── Canal 5
├── Canal 6
├── Canal 7
└── Canal 8

Daisy
├── Canal 9
├── Canal 10
├── Canal 11
├── Canal 12
├── Canal 13
├── Canal 14
├── Canal 15
└── Canal 16
```

---

## 4. Software

El proyecto utiliza:

- Raspberry Pi OS / Linux.
- Python 3.
- NumPy.
- PySerial.
- Pandas.
- Matplotlib.

### Instalación de dependencias

Las bibliotecas necesarias pueden instalarse mediante:

```bash
pip install numpy pyserial pandas matplotlib
```

También se puede utilizar el archivo `requirements.txt`:

```bash
pip install -r requirements.txt
```

---

## 5. Comunicación serial

El programa establece comunicación con el dispositivo mediante:

```python
ser = serial.Serial('/dev/ttyUSB0', 115200, timeout=1)
```

Los parámetros utilizados son:

| Parámetro | Valor |
|---|---|
| Puerto serial | `/dev/ttyUSB0` |
| Baudrate | `115200` |
| Timeout | `1 s` |

El puerto puede variar dependiendo de cómo Linux identifique el dispositivo conectado.

Para comprobar los dispositivos seriales disponibles se puede utilizar:

```bash
ls /dev/ttyUSB*
```

o:

```bash
ls /dev/ttyACM*
```

Si el OpenBCI aparece con un nombre diferente, es necesario modificar el puerto dentro del código.

---

## 6. Formato de los paquetes

Durante el streaming, cada paquete contiene **33 bytes**.

La estructura utilizada para los datos EEG es:

```text
┌────────┬───────────┬────────────────────┬──────────┬────────┐
│ Header │ Sample #  │ 8 canales EEG      │   AUX    │ Footer │
│ 1 byte │  1 byte   │     24 bytes       │ 6 bytes  │ 1 byte │
└────────┴───────────┴────────────────────┴──────────┴────────┘

                         Total: 33 bytes
```

Los 8 canales EEG ocupan 24 bytes porque cada canal utiliza 3 bytes:

```text
8 canales × 3 bytes = 24 bytes
```

Cada valor EEG corresponde a un número de **24 bits**.

### Posición de los canales

Dentro del paquete, los datos EEG comienzan en el índice `2` de Python.

| Canal | Índices Python |
|---|---|
| Canal 1 | 2–4 |
| Canal 2 | 5–7 |
| Canal 3 | 8–10 |
| Canal 4 | 11–13 |
| Canal 5 | 14–16 |
| Canal 6 | 17–19 |
| Canal 7 | 20–22 |
| Canal 8 | 23–25 |

Por lo tanto, el programa utiliza:

```python
idx = 2 + ch*3
```

para localizar los tres bytes correspondientes a cada canal.

---

## 7. Lectura de 16 canales

Un paquete contiene información de 8 canales.

Por esta razón, el programa lee dos paquetes:

```python
packet1 = ser.read(33)
packet2 = ser.read(33)
```

El primer paquete se utiliza para los canales 1–8:

```text
packet1 → canales 1–8
```

El segundo paquete se utiliza para los canales 9–16:

```text
packet2 → canales 9–16
```

Posteriormente ambos grupos se almacenan dentro de una misma matriz:

```text
              Muestra
                 │
       ┌─────────┴─────────┐
       ▼                   ▼
   packet1              packet2
  canales 1–8          canales 9–16
       │                   │
       └─────────┬─────────┘
                 ▼
             16 canales
```

---

## 8. Conversión de datos de 24 bits

Los datos EEG se reciben como tres bytes.

La función utilizada para convertirlos es:

```python
def read_24bit_signed(b1, b2, b3):
    value = (b1 << 16) | (b2 << 8) | b3

    if value & 0x800000:
        value -= 1 << 24

    return value
```

Primero se combinan los tres bytes:

```python
value = (b1 << 16) | (b2 << 8) | b3
```

Esto genera un valor de 24 bits.

Después se comprueba el bit de signo:

```python
if value & 0x800000:
```

Si el bit más significativo está activo, el valor se interpreta como negativo utilizando representación de complemento a dos.

El resultado final es un valor entero con signo.

---

## 9. Estructura de los datos

La matriz principal utilizada por el programa es:

```python
data_EEG = np.zeros((16, N_points))
```

Actualmente:

```python
N_points = 1000
```

Por lo tanto:

```text
16 canales × 1000 muestras
```

La estructura es:

```text
                 Muestras
             1   2   3  ... 1000
          ┌───┬───┬───┬───────┐
Canal 1   │   │   │   │       │
Canal 2   │   │   │   │       │
Canal 3   │   │   │   │       │
   ...    │   │   │   │       │
Canal 16  │   │   │   │       │
          └───┴───┴───┴───────┘
```

Cada fila representa un canal y cada columna una muestra.

---

## 10. Almacenamiento en CSV

Después de finalizar la adquisición, los datos se convierten en un `DataFrame`:

```python
df = pd.DataFrame(
    data_EEG.T,
    columns=[f"Canal_{i+1}" for i in range(16)]
)
```

Se utiliza `.T` para transponer la matriz.

La matriz interna:

```text
16 × 1000
```

se convierte en:

```text
1000 × 16
```

El archivo generado es:

```text
EEG_16_canales.csv
```

Su estructura es:

```text
Canal_1,Canal_2,Canal_3,...,Canal_16
valor,valor,valor,...,valor
valor,valor,valor,...,valor
...
```

Esto facilita posteriormente utilizar los datos con Python, MATLAB, Excel, R u otras herramientas de análisis.

---

## 11. Conversión a microvoltios

El programa incluye una conversión de los valores digitales a microvoltios mediante:

```python
scale = 4.5 / (24 * (2**23 - 1)) * 1e6
```

y:

```python
data_EEG_uV = data_EEG * scale
```

Esta conversión es la utilizada actualmente por el proyecto para representar los datos en microvoltios.

El factor de conversión debe considerarse dependiente de la configuración del hardware y de adquisición utilizada; por lo tanto, cualquier análisis cuantitativo posterior debe verificar la configuración de ganancia y referencia correspondiente.

---

## 12. Eje temporal

El programa utiliza:

```python
fs = 250
```

y genera el eje temporal mediante:

```python
t = np.arange(N_points) / fs
```

Con 1000 muestras:

```text
1000 / 250 = 4 segundos
```

por lo que la gráfica representa aproximadamente 4 segundos de datos bajo esta configuración temporal.

> **Nota:** `fs` en este programa se utiliza para construir el eje temporal. Cambiar este valor en Python no modifica por sí mismo la frecuencia física de muestreo del hardware.

---

## 13. Visualización

Para visualizar simultáneamente los 16 canales se utiliza Matplotlib.

Se aplica un desplazamiento vertical entre canales:

```python
offset = 100
```

y:

```python
plt.plot(t, data_EEG_uV[ch] + ch*offset)
```

Esto produce una gráfica similar a:

```text
Amplitud
   │
   │  Canal 16 ─────────────────────
   │
   │  Canal 15 ─────────────────────
   │
   │  Canal 14 ─────────────────────
   │
   │             ...
   │
   │  Canal 2  ─────────────────────
   │  Canal 1  ─────────────────────
   └────────────────────────────────►
                    Tiempo
```

El desplazamiento vertical es únicamente para facilitar la visualización y no modifica los datos originales almacenados.

---

# 14. Código completo

El siguiente código corresponde a la versión utilizada para realizar la adquisición de los 16 canales:

```python
import numpy as np
import serial
import time
import pandas as pd
import matplotlib.pyplot as plt


# ---------- Función para convertir 24 bits signed ----------
def read_24bit_signed(b1, b2, b3):
    value = (b1 << 16) | (b2 << 8) | b3

    if value & 0x800000:
        value -= 1 << 24

    return value


# ---------- Conexión ----------
ser = serial.Serial('/dev/ttyUSB0', 115200, timeout=1)
time.sleep(2)

print("Inicializando Cyton...")

ser.write(b'v')   # reset
time.sleep(1)

ser.write(b'd')   # canales por defecto
time.sleep(1)


# ---------- Configuración ----------
fs = 250  # Hz


# ---------- Iniciar streaming ----------
ser.reset_input_buffer()
ser.write(b'b')
time.sleep(0.1)

print("Adquiriendo datos...")


# ---------- Lectura de 16 canales ----------
N_points = 1000

data_EEG = np.zeros((16, N_points))

i = 0

while i < N_points:

    packet1 = ser.read(33)
    packet2 = ser.read(33)

    if len(packet1) == 33 and len(packet2) == 33:

        if packet1[0] == 0xA0 and packet2[0] == 0xA0:

            # Canales 1–8
            for ch in range(8):

                idx = 2 + ch * 3

                data_EEG[ch, i] = read_24bit_signed(
                    packet1[idx],
                    packet1[idx + 1],
                    packet1[idx + 2]
                )

            # Canales 9–16
            for ch in range(8):

                idx = 2 + ch * 3

                data_EEG[ch + 8, i] = read_24bit_signed(
                    packet2[idx],
                    packet2[idx + 1],
                    packet2[idx + 2]
                )

            i += 1


# ---------- Detener streaming ----------
ser.write(b's')
ser.close()

print("Datos obtenidos:", data_EEG.shape)


# ---------- Guardar CSV ----------
df = pd.DataFrame(
    data_EEG.T,
    columns=[f"Canal_{i+1}" for i in range(16)]
)

df.to_csv("EEG_16_canales.csv", index=False)

print("Archivo guardado: EEG_16_canales.csv")


# ---------- Convertir a microvoltios ----------
scale = 4.5 / (24 * (2**23 - 1)) * 1e6

data_EEG_uV = data_EEG * scale


# ---------- Graficar ----------
t = np.arange(N_points) / fs

offset = 100  # microvoltios

plt.figure(figsize=(12, 10))

for ch in range(16):

    plt.plot(
        t,
        data_EEG_uV[ch] + ch * offset
    )

plt.title("EEG 16 canales - OpenBCI Cyton + Daisy")
plt.xlabel("Tiempo (s)")
plt.yticks([])
plt.tight_layout()
plt.show()
```

---

# 15. Ejecución

Después de conectar el sistema OpenBCI a la Raspberry Pi, ejecutar:

```bash
python3 adquisicion_eeg_16ch.py
```

El programa mostrará:

```text
Inicializando Cyton...
Adquiriendo datos...
Datos obtenidos: (16, 1000)
Archivo guardado: EEG_16_canales.csv
```

Al finalizar se generará el archivo:

```text
EEG_16_canales.csv
```

y aparecerá la gráfica correspondiente a los 16 canales.

---

# 16. Estructura recomendada del repositorio

```text
OpenBCI-EEG-16-Canales/
│
├── README.md
│
├── codigo/
│   └── adquisicion_eeg_16ch.py
│
├── datos/
│   └── EEG_16_canales.csv
│
└── requirements.txt
```

### `README.md`

Este documento. Contiene la descripción, funcionamiento, arquitectura y código del proyecto.

### `codigo/adquisicion_eeg_16ch.py`

Programa encargado de realizar la adquisición y procesamiento inicial de los datos.

### `datos/EEG_16_canales.csv`

Archivo generado por el programa con las muestras de los 16 canales.

### `requirements.txt`

Lista de bibliotecas de Python necesarias para ejecutar el programa.

Un ejemplo de su contenido es:

```text
numpy
pyserial
pandas
matplotlib
```

---

# 17. Limitaciones actuales

La versión actual del sistema está enfocada principalmente en la **adquisición y almacenamiento de señales EEG**.

Actualmente no se implementan:

- Filtros digitales.
- Filtro notch.
- Eliminación automática de artefactos.
- FFT.
- PSD.
- Espectrogramas.
- Clasificación de señales.
- Detección automática de eventos.
- Interfaz gráfica.
- Visualización EEG en tiempo real.

Estas funciones pueden incorporarse posteriormente sobre los datos adquiridos.

---

# 18. Trabajo futuro

Como continuación del proyecto se pueden implementar:

1. Filtrado digital de las señales.
2. Eliminación o detección de artefactos.
3. Análisis de frecuencia mediante FFT.
4. Análisis de las bandas:
   - Delta
   - Theta
   - Alpha
   - Beta
   - Gamma
5. Cálculo de potencia espectral.
6. Visualización en tiempo real.
7. Registro de sesiones EEG.
8. Base de datos para almacenar experimentos.
9. Interfaz gráfica para controlar la adquisición.
10. Algoritmos de clasificación y reconocimiento de patrones.

---

# 19. Resumen

El proyecto implementa un sistema de adquisición EEG de **16 canales** utilizando un **OpenBCI Cyton + Daisy** y una **Raspberry Pi** como plataforma de procesamiento.

El software desarrollado en Python permite recibir los paquetes enviados por el dispositivo, extraer los valores correspondientes a cada canal, convertir los datos de 24 bits a valores numéricos, almacenarlos en una matriz de 16 canales, generar un archivo CSV y visualizar las señales adquiridas.

La arquitectura desarrollada proporciona una base para incorporar posteriormente técnicas de procesamiento digital de señales y análisis EEG más avanzadas.

---

## Tecnologías

**Hardware**

- OpenBCI Cyton
- OpenBCI Daisy
- Raspberry Pi
- Electrodos EEG

**Software**

- Python
- NumPy
- PySerial
- Pandas
- Matplotlib
- Linux / Raspberry Pi OS
