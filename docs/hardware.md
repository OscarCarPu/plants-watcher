# Hardware

MCU: XIAO ESP32-C3, MQTT over WiFi.

## Parts

| Function | Part | Notes |
|---|---|---|
| Soil moisture | Capacitive probe → ADC | Analogue out |
| Soil temp | NTC 10k at probe tip | Compensates capacitive drift; zero idle current |
| Air temp/RH | SHT40 (I2C) | ±1.8 %RH, 0.08 µA idle, low drift |
| Battery | LiPo 1200 mAh, XIAO charger | Read via ADC |
