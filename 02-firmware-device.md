# 02 — Прошивка устройства (под-план)

Возврат к [мастер-плану](00-MASTER-architecture.md). Цель: прошивка для **XIAO ESP32-C6**,
которая поднимает BLE GATT-peripheral по [контракту](00-MASTER-architecture.md#3-ble-gatt-контракт-единый-источник-истины),
рендерит статус на MAX7219 и экономит батарею.

---

## 1. Технологический стек
- **Arduino-ESP32 core 3.x** (поддерживает ESP32-C6). Сборка через **PlatformIO** (воспроизводимо, удобно для open source).
- **NimBLE-Arduino** — лёгкий BLE-стек (меньше RAM/flash, чем стоковый BLEDevice), GATT-server.
- **MD_Parola + MD_MAX72XX** (`majicDesigns`) — рендер текста/скролла на MAX7219.
- **Preferences (NVS)** — сохранение последнего статуса/яркости.

`platformio.ini` (ориентир):
```ini
[env:xiao_esp32c6]
platform = espressif32
board = seeed_xiao_esp32c6
framework = arduino
monitor_speed = 115200
lib_deps =
  h2zero/NimBLE-Arduino
  majicdesigns/MD_MAX72XX
  majicdesigns/MD_Parola
```

---

## 2. Структура прошивки
```
firmware/
  platformio.ini
  src/
    main.cpp           # setup/loop, оркестрация
    BleService.h/.cpp  # GATT: Status/Text/Battery, колбэки
    Display.h/.cpp     # обёртка MD_Parola: renderStatus(), shutdown()
    Override.h/.cpp    # кнопка/геркон + debounce
    Battery.h/.cpp     # ADC -> проценты, калибровка
    Power.h/.cpp       # light sleep, conn params
    Config.h           # UUID, пины, яркость, тайминги
```

### `Config.h` — общие константы (синхронизированы с контрактом!)
```cpp
#define DEVICE_NAME        "BusyLight"
#define SVC_UUID           "6e401b00-b5a3-f393-e0a9-e50e24dcca9e"
#define CHR_STATUS_UUID    "6e401b01-b5a3-f393-e0a9-e50e24dcca9e"
#define CHR_TEXT_UUID      "6e401b02-b5a3-f393-e0a9-e50e24dcca9e"
// Battery: стандартные 0x180F / 0x2A19

#define PIN_DIN  18   // D10 / MOSI
#define PIN_CLK  19   // D8  / SCK
#define PIN_CS    2   // D2
#define PIN_BTN  21   // D3, INPUT_PULLUP
#define PIN_VBAT  0   // D0 / ADC1

#define NUM_MODULES 4         // 8x32
#define BRIGHTNESS_DEFAULT 4  // 0..15

enum Status : uint8_t { IDLE=0, BUSY=1, ON_CALL=2, CUSTOM=3 };
```

---

## 3. GATT-server (NimBLE)
- Создать сервис `SVC_UUID` + характеристики `Status` (R/W/Notify), `Text` (R/W), и стандартный Battery Service.
- На запись `Status`: распарсить 1 байт → вызвать `Display::renderStatus()` → сохранить в NVS.
- На запись `Text`: сохранить строку (≤32 байт), если текущий статус `CUSTOM` — перерисовать.
- Advertising: имя `BusyLight` + Service UUID; рестарт advertising при дисконнекте.

Скелет колбэка записи:
```cpp
class StatusCb : public NimBLECharacteristicCallbacks {
  void onWrite(NimBLECharacteristic* c) override {
    auto v = c->getValue();
    if (v.size() >= 1) display.renderStatus((Status)v[0]);
  }
};
```

---

## 4. Рендер статуса (`Display`)
| Статус | Поведение |
|---|---|
| `IDLE` | `mx.control(SHUTDOWN, ON)` — матрица гаснет (µA), главная экономия |
| `BUSY` | статично «BUSY» (центр), выбранная яркость |
| `ON_CALL` | статично «ON CALL» (или иконка телефона из user-defined char) |
| `CUSTOM` | бегущая строка из `Text` (`MD_Parola` scroll) |

- Яркость: `mx.control(INTENSITY, brightness)`.
- При выходе из `IDLE` — снять `shutdown` перед отрисовкой.
- Кириллица: MD_MAX72XX по умолчанию ASCII; для «ЗАНЯТ» подключить кастомный шрифт/иконки (или использовать латиницу «BUSY» в v1).

---

## 5. Локальный override (`Override`)
- Кнопка на `PIN_BTN` (`INPUT_PULLUP`), фронт по нажатию.
- Debounce: программный (≥30 мс) или аппаратный (0.1 µF, см. [`01`](01-hardware.md)).
- Логика: нажатие циклически переключает `IDLE ⇄ BUSY` **локально** и:
  1. немедленно рендерит,
  2. шлёт `Status` **Notify** на Mac (чтобы приложение знало об override и не перетёрло его).
- Геркон-вариант: «магнит снят» = `BUSY`, «магнит на месте» = `IDLE` (или наоборот — конфигурируемо).

---

## 6. Измерение заряда (`Battery`)
- Читать `analogRead(PIN_VBAT)`, усреднять (скользящее среднее N=16).
- Перевод: `V_bat = adc_voltage * 2` (делитель 1:1). Карта в проценты по кривой alkaline (нелинейная): например 4.7 В→100%, 4.2 В→60%, 3.8 В→20%, 3.6 В→0%.
- Калибровка: измерить мультиметром и поправить коэффициент ADC (ESP32 ADC нелинеен — желательна калибровка по `esp_adc_cal`).
- Публиковать в `Battery Level`: периодически (таймер) и при изменении на ≥5%.

---

## 7. Power management (`Power`)
Цель — минимальный ток в `IDLE` при сохранении отзывчивости.
- **Матрица:** `shutdown` в `IDLE` (см. выше) — даёт основной выигрыш.
- **BLE conn params:** запросить у central высокую *slave latency* и умеренный *connection interval*
  (например, interval ~30–50 мс, latency ~30–60), чтобы radio просыпался реже.
- **Light sleep:** между BLE-событиями — automatic light sleep (`esp_pm_config`), wake по BLE и по GPIO кнопки.
- **Опц. (v2):** при долгом `IDLE` — deep sleep + только advertising; central переподключается при смене статуса (ценой латентности).

Замерять ток мультиметром на каждом этапе (см. приёмку в [`01`](01-hardware.md)).

---

## 8. Сборка и прошивка
```bash
# из папки firmware/
pio run                  # сборка
pio run -t upload        # прошивка по USB-C
pio device monitor       # serial-лог (115200)
```
Первичная проверка GATT без Mac-приложения: приложение **nRF Connect** (телефон) →
найти `BusyLight` → подключиться → писать в `Status` байты `00/01/02/03` и наблюдать рендер.

---

## 9. Приёмочные критерии (firmware)
- [ ] Устройство видно как `BusyLight`, отдаёт сервис и 3 характеристики.
- [ ] Запись `Status` = `01/02` мгновенно меняет надпись; `00` — гасит матрицу.
- [ ] `CUSTOM` + запись `Text` — корректная бегущая строка (≤32 байт, UTF-8 latin).
- [ ] Кнопка/геркон переключает локально и шлёт Notify; Mac-приложение это видит.
- [ ] `Battery Level` отдаёт правдоподобный %, обновляется по таймеру.
- [ ] Ток в `IDLE` в пределах энергобюджета.

## 10. Будущее
- OTA-обновления (ArduinoOTA / esp_https_ota).
- Shared-secret в `Text` для защиты от чужой записи.
- Кириллический шрифт и анимации/иконки.
