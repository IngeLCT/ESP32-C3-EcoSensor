# ESP32-C3 EcoSensor

Firmware ESP-IDF para EcoSensor basado en ESP32-C3, con Wi-Fi manager,
sensores ambientales SCD40 y SEN55, GPS GY-GPS6MV2, almacenamiento en
MicroSD por SPI y OTA local ordenada por el servidor EcoSensor.

## Placa objetivo

La placa personalizada usa un módulo `ESP32-C3-WROOM-02` y expone sus señales
en dos headers. El mapa físico confirmado incluye GPIO0, GPIO1, GPIO2, GPIO3,
GPIO4, GPIO5, GPIO6, GPIO7, GPIO8, GPIO9, GPIO10, GPIO18, GPIO19, UART0 TX/RX,
3.3 V, EN y GND. Este firmware no utiliza GPIO2, GPIO8 ni GPIO9 porque son
pines de strapping, ni GPIO18/GPIO19 para conservar USB-JTAG disponible.

## Hardware y distribución de pines

### I2C compartido: SCD40 y SEN55

| Señal | ESP32-C3 |
|---|---:|
| SDA | GPIO4 |
| SCL | GPIO5 |
| Frecuencia | 100 kHz |

Ambos sensores comparten el bus. El SCD40 usa la dirección `0x62` y el SEN55
la dirección `0x69`. SDA y SCL necesitan resistencias pull-up a 3.3 V; no se
deben duplicar demasiados pull-up si los módulos ya las incluyen.

### GPS GY-GPS6MV2 por UART1

| Señal | ESP32-C3 | GPS |
|---|---:|---|
| RX | GPIO0 | TX |
| TX | GPIO1 | RX |
| Velocidad | 9600 baud | NMEA |

La entrada RX del ESP32-C3 recibe desde TX del GPS. El firmware no usa pin
WAKEUP porque el GY-GPS6MV2 no expone el control empleado por el L76K del
proyecto WROVER.

### V474 MicroSD por SPI2

| Señal | ESP32-C3 |
|---|---:|
| SCLK/CLK | GPIO6 |
| MOSI/DI | GPIO7 |
| MISO/DO | GPIO3 |
| CS | GPIO10 |

La interfaz de software usa SDSPI sobre `SPI2_HOST`. Antes de alimentar el
módulo V474 se confirmó físicamente que su entrada está marcada `3.3V`; debe
conectarse al header de 3.3 V de la placa y compartir GND. Las señales lógicas
del ESP32-C3 también son de 3.3 V. La frecuencia SPI inicial se limita a 10 MHz
para priorizar estabilidad; podrá aumentarse después de validar el módulo, el
cableado y la tarjeta.

## Identidad inicial

- mDNS: `ecosensorc3.local`
- AP temporal: `EcoSensor-C3`
- Versión inicial: `1.0.0`

Estos valores deben ajustarse antes de desplegar varias unidades para evitar
colisiones de nombre en la red.

## OTA local

Esta versión usa tabla de particiones OTA para flash de 4 MB:

- `otadata` en `0xd000`
- `ota_0` en `0x10000`
- `ota_1` en `0x200000`

El archivo reproducible es `partitions_ota_4mb.csv` y la configuración queda en `sdkconfig.defaults`.

> Importante: los equipos que todavía tengan una imagen antigua con partición Single App Large deben migrarse una vez por USB/cable usando esta tabla OTA. Después de esa primera migración podrán actualizarse por OTA local.

## Versión de firmware

La versión se define en el `CMakeLists.txt` raíz con `set(PROJECT_VER "x.y.z")`, igual que en el proyecto GSM. El firmware la expone como `firmware_version` en:

- `GET /status`
- `GET /diagnostics`
- `GET /debug`

## Endpoints OTA del ESP32

### `POST /ota/update`

Payload esperado:

```json
{
  "device_id": "ecosensor02",
  "version": "1.0.1",
  "firmware_url": "http://192.168.1.97:8765/firmware/ecosensor02/ecosensor02_v1.0.1.bin",
  "sha256": "opcional"
}
```

El endpoint valida:

- que haya red STA con IP;
- que no haya otra OTA en curso;
- que `device_id`, `version` y `firmware_url` existan;
- que `device_id` coincida con el `mdns_hostname` del firmware;
- que la URL sea HTTP local (`http://...`).

Responde de inmediato con `state: queued` y ejecuta la descarga en una tarea independiente.

### `GET /ota/status`

Devuelve:

```json
{
  "ok": true,
  "state": "idle|queued|downloading|writing|success|error|rebooting",
  "current_version": "1.0.1",
  "target_version": "1.0.2",
  "bytes_received": 0,
  "total_bytes": -1,
  "progress_pct": null,
  "last_error": null
}
```

## Rollback

Está habilitado `CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE=y`. En el primer arranque de una imagen OTA nueva, `app_main()` marca la app como válida después de un arranque mínimo correcto. La ausencia de SD no provoca rollback: el firmware ya tolera SD no disponible.

## Decisión durante OTA

La OTA corre en una tarea separada y evita múltiples actualizaciones simultáneas. No se detienen sensores ni SD por defecto para no romper la arquitectura actual; la escritura OTA usa particiones flash separadas. Si más adelante se observa contención, se puede agregar una pausa explícita de sensores durante `downloading/writing`.

## Compilar

```bash
source /home/eduardo/tools/esp-idf/export.sh
idf.py build
```

Para la primera migración por USB con OTA:

```bash
idf.py flash
```

O manualmente con los offsets reportados por ESP-IDF:

```bash
python -m esptool --chip esp32c3 -b 460800 --before default_reset --after hard_reset write_flash \
  --flash_mode dio --flash_size 4MB --flash_freq 40m \
  0x0 build/bootloader/bootloader.bin \
  0x8000 build/partition_table/partition-table.bin \
  0xd000 build/ota_data_initial.bin \
  0x10000 build/ESP32-C3-EcoSensor.bin
```
