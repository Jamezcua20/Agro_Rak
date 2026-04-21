# Agro-Node RAK3172: Monitoreo de Presión Industrial y Suelo (RS485)

Nodo LoRaWAN de ultra bajo consumo diseñado para entornos agrícolas. Integra un transceptor **RAK3172** con una etapa de potencia conmutada para alimentar sensores externos de 12V (Modbus RTU) solo durante el ciclo de lectura.

## Especificaciones de Hardware

| Componente | Referencia | Función |
| :--- | :--- | :--- |
| **MCU/LoRa** | RAK3172 (U1) | STM32WLE5CC, Stack LoRaWAN v1.0.3. |
| **RS485 IC** | THVD2450DR (U5) | Transceptor robusto con protección contra fallas. |
| **Boost 12V** | MT3608 (U3) | Elevador para alimentación de sensores (V_CC +12). |
| **LDO 3.3V** | MCP1700T (U4) | Regulación de sistema con $I_q$ de 1.6µA. |
| **Cargador** | MCP73831T (U2) | Gestión de carga Li-Po/Solar. |

## Mapeo de Pines (RAK3172)

| Pin | Señal | Función |
| :--- | :--- | :--- |
| **PA15** | `RS_485_ENA` | Habilitación de etapa Boost +12V (Active HIGH). |
| **PA0** | `DE` | Driver Enable (RS485). |
| **PA1** | `RE` | Receiver Enable (RS485 - Negado). |
| **PA8** | `NIV_BAT` | Lectura ADC de voltaje de batería (Divisor resistivo). |
| **PB2/3/4** | `LEDS` | Indicadores de estado: Sistema, LoRa y RS485. |
| **PA2/3** | `UART2` | Interfaz de flasheo / Galio Flash Tool. |

## Estrategia de Eficiencia Energética

El diseño utiliza **Power Gating** para maximizar la vida útil de la batería:
1. El sistema permanece en **Deep Sleep** (MCU ~1.6µA).
2. Al despertar, se activa `RS_485_ENA` (PA15) para encender el regulador **MT3608**.
3. Se realiza el polling Modbus a los sensores de suelo/presión.
4. Se apaga la etapa de potencia y el nodo vuelve a modo sleep tras el Uplink LoRaWAN.

## Modelo de Datos (Payload Sugerido)

Para optimizar el *Airtime*, los datos se transmiten en formato binario:

| Offset | Variable | Tipo | Factor |
| :--- | :--- | :--- | :--- |
| 0-1 | Presión | uint16 | 0.01 (bar) |
| 2-3 | Humedad | uint16 | 0.1 (%) |
| 4-5 | Temp | int16 | 0.1 (°C) |
| 6-7 | V_Bat | uint16 | 0.001 (V) |

## Despliegue Rápido

1. **Flasheo:** Conectar vía UART2 (J3) o SWD (J2).
2. **Configuración:** Ajustar `AppEUI` y `AppKey` en el firmware basado en RAK Unified Interface (RUI3).
3. **Sensores:** Conectar sensores RS485 en el bloque **J5**. Asegurar que la polaridad de `P_N` y `P_P` sea correcta.

## Consumo Energético

Análisis de consumo proyectado desde $V_{BAT}$ (3.3V nominal). Los valores reflejan el comportamiento del hardware bajo gestión de energía activa (Power Gating).

| Componente | Modo Dormido | Modo Activo | Pico / Máximo |
| :--- | :--- | :--- | :--- |
| **RAK3172** (MCU + LoRa) | ~2 µA (Deep Sleep) | ~5 mA (RX/Processing) | ~87 mA (TX @ +22dBm) |
| **MT3608** (Boost 3.3V ⮕ 12V) | 0.1 µA (Shutdown) | 100-200 µA (PFM) | ~260 mA (Carga sensores) |
| **MCP1700** (LDO 3.3V) | 1.6 µA (Iq típica) | ~90 mA (Sistema) | 250 mA (Límite IC) |
| **THVD2450** (RS485) | < 1 µA (Disabled) | 1.8-2.4 mA (RX) | 3.5-5.6 mA (TX+RX) |
| **MCP73831** (Charger) | ~5 µA (No Panel) | ~1 mA + Carga | 212 mA (Carga R4=470Ω) |
| **Sensores Modbus x2** | 2-5 mA (Standby) | 25-50 mA (Medición) | ~60 mA (Arranque) |
| **LEDs (R/V/A)** | 0 mA (Apagados) | ~4 mA (c/u) | ~12 mA (Full On) |
| **ADC Batería** (R1+R2) | 16.5 µA (Constante) | 16.5 µA | 16.5 µA |

### Notas de Eficiencia
* **Corriente de Reposo Total:** Se estima un *quiescent current* de sistema de **~25-30 µA** (considerando que el Power Gating corta totalmente la alimentación de 12V a los sensores).
* **Consumo ADC:** Los 16.5 µA del divisor resistivo representan la mayor fuga constante. *Mejora sugerida para v1.1:* Implementar un MOSFET de canal P para conmutar el divisor solo durante la lectura.
* **Impacto del Boost:** La eficiencia del MT3608 es crítica; se recomienda mantener los cables de los sensores de suelo cortos para minimizar caídas de tensión y picos de corriente innecesarios.

---
**Diseñado por:** José Amezcua
**Estado:** MVP v1.0
