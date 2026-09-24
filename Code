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
                idx = 2 + ch*3
                data_EEG[ch, i] = read_24bit_signed(packet1[idx], packet1[idx+1], packet1[idx+2])

            # Canales 9–16
            for ch in range(8):
                idx = 2 + ch*3
                data_EEG[ch+8, i] = read_24bit_signed(packet2[idx], packet2[idx+1], packet2[idx+2])

            i += 1

# ---------- Detener streaming ----------
ser.write(b's')
ser.close()

print("Datos obtenidos:", data_EEG.shape)

# ---------- Guardar CSV ----------
df = pd.DataFrame(data_EEG.T, columns=[f"Canal_{i+1}" for i in range(16)])
df.to_csv("EEG_16_canales.csv", index=False)
print("Archivo guardado: EEG_16_canales.csv")

# ---------- (Opcional) Convertir a microvoltios ----------
scale = 4.5 / (24 * (2**23 - 1)) * 1e6
data_EEG_uV = data_EEG * scale

# ---------- Graficar ----------
t = np.arange(N_points) / fs
offset = 100  # microvoltios

plt.figure(figsize=(12,10))
for ch in range(16):
    plt.plot(t, data_EEG_uV[ch] + ch*offset)

plt.title("EEG 16 canales - OpenBCI Cyton + Daisy")
plt.xlabel("Tiempo (s)")
plt.yticks([])
plt.tight_layout()
plt.show()
