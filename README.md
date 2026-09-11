# Práctica 2: Control de Semáforo Peatonal 

## Objetivo General
Implementar un sistema de control de semáforo inteligente que regule el paso de vehículos y peatones utilizando una **máquina de estados finitos**. El sistema utiliza un botón de petición peatonal para cambiar entre estados.

---

## Componentes Utilizados

### Hardware
| Componente | Cantidad | Pines GPIO | Función |
|-----------|----------|-----------|---------|
| **Raspberry Pi Pico** | 1 | - | Microcontrolador principal |
| **LED Rojo (Autos)** | 1 | GP15 | Señal ROJO para vehículos |
| **LED Amarillo (Autos)** | 1 | GP14 | Señal AMARILLO para vehículos |
| **LED Verde (Autos)** | 1 | GP13 | Señal VERDE para vehículos |
| **LED Rojo (Peatones)** | 1 | GP12 | Señal ROJO para peatones |
| **LED Verde (Peatones)** | 1 | GP11 | Señal VERDE para peatones |
| **Botón Pulsador** | 1 | GP16 | Petición peatonal |
| **Resistencias** | 5 | - | 330Ω (limitadoras de corriente para LEDs) |

### Software
- **Lenguaje**: MicroPython
- **Plataforma de Simulación**: Wokwi
- **Biblioteca**: `machine` (control de GPIO), `time` (delays)

---

## Máquina de Estados

El sistema implementa una máquina de estados con **4 estados principales**:

```
┌─────────────────────────────────────────────┐
│  S0: REPOSO - Autos pasan, peatones esperan │
│  • Semáforo autos: 🟢 (verde)               │
│  • Semáforo peatones: 🔴 (rojo)             │
│  • Duración: Indefinida (hasta botón)       │
└────────────────┬────────────────────────────┘
                 │ [Botón presionado]
                 ▼
┌──────────────────────────────────────────────┐
│ S1: TRANSICIÓN - Autos preparan alto        │
│ • Semáforo autos: 🟡 (amarillo)             │
│ • Semáforo peatones: 🔴 (rojo)              │
│ • Duración: 1500 ms                         │
└────────────────┬─────────────────────────────┘
                 │ [Espera 1.5 segundos]
                 ▼
┌──────────────────────────────────────────────┐
│ S2: CRUCE - Peatones pueden pasar           │
│ • Semáforo autos: 🔴 (rojo)                 │
│ • Semáforo peatones: 🟢 (verde)             │
│ • Duración: 4000 ms                         │
└────────────────┬─────────────────────────────┘
                 │ [Espera 4 segundos]
                 ▼
┌──────────────────────────────────────────────┐
│ S3: FIN DE CRUCE - Peatón verde parpadea   │
│ • Semáforo autos: 🔴 (rojo)                 │
│ • Semáforo peatones: Verde parpadea 6 veces│
│ • Duración: 1800 ms (6 × 300ms)             │
└────────────────┬─────────────────────────────┘
                 │ [Parpadeo completo]
                 ▼
            Retorna a S0
```

---

## Descripción del Código

### 1. **Configuración de Pines**
```python
CAR_RED=15      # GP15: LED rojo autos
CAR_YELLOW=14   # GP14: LED amarillo autos
CAR_GREEN=13    # GP13: LED verde autos
PED_RED=12      # GP12: LED rojo peatones
PED_GREEN=11    # GP11: LED verde peatones
BUTTON=16       # GP16: Botón petición peatonal
```

### 2. **Función Principal: `set_lights()`**
```python
def set_lights(car_r, car_y, car_g, ped_r, ped_g):
    car_red.value(car_r)
    car_yellow.value(car_y)
    car_green.value(car_g)
    ped_red.value(ped_r)
    ped_green.value(ped_g)
```
**Propósito**: Establece el estado de todos los LEDs en una sola llamada.
- Parámetros: 0 (apagado) o 1 (encendido)

### 3. **Funciones de Estado**

#### **`cars_go()`** - Estado S0
```python
def cars_go():
    print("S0 REPOSO: Autos pasan, peaton espera")
    set_lights(0,0,1,1,0)  # Verde autos, Rojo peatones
```

#### **`cars_prepare_to_stop()`** - Estado S1
```python
def cars_prepare_to_stop():
    print("S1 TRANSICION: Autos preparan alto")
    set_lights(0,1,0,1,0)  # Amarillo autos, Rojo peatones
```

#### **`pedestrians_go()`** - Estado S2
```python
def pedestrians_go():
    print("S2 CRUCE: peaton puede pasar")
    set_lights(1,0,0,0,1)  # Rojo autos, Verde peatones
```

#### **`pedestrians_finish()`** - Estado S3
```python
def pedestrians_finish():
    print("S3 FIN DE CRUCE: peaton verde parpadea")
    set_lights(1,0,0,0,1)
    for _ in range(6):
        ped_green.toggle()     # Parpadea el LED verde
        sleep_ms(300)
    ped_green.value(0)
    ped_red.value(1)
```
**Nota**: Usa `.toggle()` para cambiar el estado del LED 6 veces, creando el efecto de parpadeo.

### 4. **Secuencia de Cruce: `crossing_sequence()`**
```python
def crossing_sequence():
    cars_prepare_to_stop()    # S1: 1.5 seg
    sleep_ms(1500)
    
    pedestrians_go()          # S2: 4 seg
    sleep_ms(4000)
    
    pedestrians_finish()      # S3: Parpadeo
    sleep_ms(500)
    
    cars_go()                 # Retorna a S0
```

### 5. **Bucle Principal - Detección de Botón**
```python
cars_go()  # Inicia en estado S0
last = 1

while True:
    now = button.value()
    
    if last == 1 and now == 0:  # Flanco descendente (botón presionado)
        sleep_ms(30)             # Debouncing
        
        if button.value() == 0:  # Confirma que sigue presionado
            print("Peticion peatonal: !!")
            crossing_sequence()  # Ejecuta toda la secuencia
            
            # Espera a que se suelte el botón
            while button.value() == 0:
                sleep_ms(10)
    
    last = now
    sleep_ms(10)
```

---

## Conceptos Clave

### **Máquina de Estados Finitos (FSM)**
El sistema funciona como una máquina con estados bien definidos. Cada estado tiene:
- Salidas específicas (qué LEDs están encendidos)
- Duraciones definidas
- Transiciones claras al siguiente estado

### **Debouncing**
```python
if last == 1 and now == 0:
    sleep_ms(30)
    if button.value() == 0:
```
Detecta el flanco descendente del botón y espera 30ms para evitar falsos contactos.

### **Parpadeo**
```python
for _ in range(6):
    ped_green.toggle()
    sleep_ms(300)
```
Alterna el estado del LED cada 300ms para crear un efecto de advertencia.

---

## Flujo de Ejecución

1. **Inicio**: Sistema en reposo (S0) - Autos circulan
2. **Usuario presiona botón**: Se detecta el flanco descendente
3. **Transición** (S1): Amarillo en autos (1.5 seg)
4. **Cruce** (S2): Verde en peatones (4 seg)
5. **Aviso de fin** (S3): Verde parpadea (1.8 seg)
6. **Retorno** (S0): Vuelve a reposo - Ciclo completo ≈ 7.8 segundos

---

## Conexiones en Wokwi

```
Raspberry Pi Pico
├─ GP15 → LED Rojo Autos (a través de R1 330Ω)
├─ GP14 → LED Amarillo Autos (a través de R2 330Ω)
├─ GP13 → LED Verde Autos (a través de R3 330Ω)
├─ GP12 → LED Rojo Peatones (a través de R4 330Ω)
├─ GP11 → LED Verde Peatones (a través de R5 330Ω)
├─ GP16 → Botón Pulsador
└─ GND → Común (resistencias y botón)
```

---

## Mejoras Potenciales

- ✅ Agregar sensor ultrasónico para detectar vehículos automáticamente
- ✅ Implementar diferentes tiempos según hora del día
- ✅ Agregar buzzer sonoro para alertar a peatones
- ✅ Usar timers en lugar de `sleep_ms()` para operaciones no bloqueantes
- ✅ Agregar pantalla LCD para mostrar contadores

---

## Conclusión

Esta práctica implementa un **sistema de control de tráfico real** usando conceptos fundamentales de:
- Control de GPIO en microcontroladores
- Máquinas de estados finitos
- Detección de eventos (botones)
- Temporización precisa

Es la base para proyectos más complejos como semáforos inteligentes y sistemas de control de tráfico urbano.
