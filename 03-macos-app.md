# 03 — Софт для MacBook (под-план)

Возврат к [мастер-плану](00-MASTER-architecture.md). Цель: приложение в строке меню на **KMP + Compose
Multiplatform**, которое вычисляет статус (ручной + авто) и пишет его на устройство по
[BLE-контракту](00-MASTER-architecture.md#3-ble-gatt-контракт-единый-источник-истины).
Должно работать на **Intel и Apple Silicon** (Intel-first).

---

## 1. Технологический стек и почему так
| Слой           | Выбор                                                   | Кросс-арх (Intel+M)?                                |
| -------------- | ------------------------------------------------------- | --------------------------------------------------- |
| Ядро/логика    | Kotlin (`commonMain`)                                   | да (JVM)                                            |
| UI / трей      | Compose Multiplatform **Desktop (JVM)**                 | да (stable на macOS JVM)                            |
| BLE central    | **Kable**, JVM-таргет (btleplug)                        | да — btleplug даёт нативные либы под x86_64 и arm64 |
| mic/cam детект | **Swift universal binary** (subprocess, JSON по stdout) | да — собираем `lipo` x86_64+arm64                   |
| Focus/DND      | **Shortcuts** автоматизация → URL-схема                 | да                                                  |

> Почему НЕ Kotlin/Native `macosArm64` мост: JetBrains убрали Apple x64-таргеты (янв. 2026),
> значит он не соберётся на Intel. Для open source нужна обе архитектуры — поэтому «грязные»
> macOS-API изолируем в **Swift universal**-хелпере, а ядро/BLE остаются на JVM.

---

## 2. Структура Gradle-проекта (KMP)
```
mac-app/
  settings.gradle.kts
  core/                      # KMP-модуль, пока только jvm-таргет (готов к расширению)
    src/commonMain/kotlin/
      domain/Status.kt           # enum + семантика
      domain/StatusStateMachine.kt  # приоритеты (тестируемо!)
      domain/Settings.kt
      ble/BleProtocol.kt         # кодек байтов <-> Status/Text (источник истины контракта)
    src/commonTest/kotlin/       # юнит-тесты машины и кодека
  app/                       # jvmMain: Compose Desktop приложение
    src/jvmMain/kotlin/
      Main.kt                    # tray, lifecycle
      ui/TrayMenu.kt
      ui/SettingsWindow.kt
      ble/KableCentral.kt        # scan/connect/write/subscribe
      detect/MicCamDetector.kt   # запуск helper, парс JSON
      detect/FocusBridge.kt      # URL-схема / file-watch + установка Shortcuts
      AppController.kt           # связывает источники -> state machine -> BLE
  helper/                    # Swift package -> universal binary "busylight-sensors"
    Sources/...
    build-helper.sh            # swiftc + lipo -> universal
```

---

## 3. `core` (commonMain) — чистая логика

### `Status.kt`
```kotlin
enum class Status(val wire: Byte) {
    IDLE(0), BUSY(1), ON_CALL(2), CUSTOM(3);
    companion object { fun fromWire(b: Byte) = entries.first { it.wire == b } }
}
```

### `BleProtocol.kt` — единственное место кодирования контракта
```kotlin
object BleProtocol {
    const val SERVICE = "6e401b00-b5a3-f393-e0a9-e50e24dcca9e"
    const val CHR_STATUS = "6e401b01-b5a3-f393-e0a9-e50e24dcca9e"
    const val CHR_TEXT = "6e401b02-b5a3-f393-e0a9-e50e24dcca9e"
    fun encodeStatus(s: Status) = byteArrayOf(s.wire)
    fun encodeText(t: String) = t.encodeToByteArray().copyOf(minOf(32, t.length))
}
```

### `StatusStateMachine.kt` — приоритеты (см. [мастер](00-MASTER-architecture.md#4-машина-состояний-и-приоритеты-статуса))
```kotlin
data class Inputs(
    val localOverride: Status?,   // с устройства (Notify), null если нет
    val manual: Status?,          // ручной выбор; null = режим AUTO
    val micOrCamActive: Boolean,  // от Swift-хелпера
    val focusOn: Boolean,         // от Focus-моста
)
class StatusStateMachine {
    fun resolve(i: Inputs): Status = when {
        i.localOverride != null -> i.localOverride
        i.manual != null -> i.manual
        i.micOrCamActive -> Status.ON_CALL
        i.focusOn -> Status.BUSY
        else -> Status.IDLE
    }
}
```
Логика **тестируется юнит-тестами на JVM** без железа — это главное преимущество выноса логики на Mac.

---

## 4. `app` — BLE central (Kable)
- Скан по `BleProtocol.SERVICE`, выбор `BusyLight`, `connect`.
- `peripheral.write(statusCharacteristic, BleProtocol.encodeStatus(s))` при каждой смене статуса.
- Подписка (`observe`) на `Status` (локальный override устройства) и `Battery Level`.
- Реконнект-loop при потере соединения (Kable `state` Flow).

Скелет:
```kotlin
class KableCentral(private val scope: CoroutineScope) {
    private var peripheral: Peripheral? = null
    suspend fun connect() {
        val adv = Scanner { filters { match { services = listOf(uuidFrom(BleProtocol.SERVICE)) } } }
            .advertisements.first()
        peripheral = scope.peripheral(adv).also { it.connect() }
    }
    suspend fun push(status: Status) {
        peripheral?.write(statusChar, BleProtocol.encodeStatus(status), WriteType.WithResponse)
    }
}
```
> Kable JVM-таргет — экспериментальный (btleplug+UniFFI). Закладываем абстракцию `BleTransport`
> (интерфейс), чтобы при проблемах подменить реализацию на Swift-хелпер с CoreBluetooth без правок ядра.

---

## 5. `app` — детект микрофона/камеры (Swift universal helper)

### Почему хелпер, а не JVM
CoreAudio/CoreMediaIO недоступны из JVM. Пишем крошечный Swift-бинарь, который опрашивает состояние
устройств и печатает строки JSON в stdout; JVM-приложение запускает его как **subprocess** и читает поток.

### Что использует хелпер (без запроса разрешений на запись!)
- Микрофон занят: свойство **`kAudioDevicePropertyDeviceIsRunningSomewhere`** у default input device (CoreAudio).
  Это **состояние устройства**, не захват звука → **не требует** разрешения на микрофон.
- Камера занята: **`kCMIODevicePropertyDeviceIsRunningSomewhere`** (CoreMediaIO).

### Протокол хелпера (stdout, по строке на событие)
```json
{"mic": true, "cam": false}
```

### Сборка universal binary (`helper/build-helper.sh`)
```bash
swiftc -O -target x86_64-apple-macos11 Sources/*.swift -o build/helper-x64
swiftc -O -target arm64-apple-macos11  Sources/*.swift -o build/helper-arm64
lipo -create build/helper-x64 build/helper-arm64 -output build/busylight-sensors
# проверка: lipo -info build/busylight-sensors  -> x86_64 arm64
```

### Сторона JVM (`MicCamDetector.kt`)
```kotlin
class MicCamDetector(private val onChange: (Boolean) -> Unit) {
    fun start() {
        val proc = ProcessBuilder(helperPath).redirectErrorStream(true).start()
        proc.inputStream.bufferedReader().lineSequence().forEach { line ->
            val s = Json.decodeFromString<SensorState>(line)
            onChange(s.mic || s.cam)
        }
    }
}
```

---

## 6. `app` — Focus / DND (Shortcuts)

Публичного API статуса Focus нет. Самый надёжный путь — автоматизация **Shortcuts**:

1. Приложение регистрирует URL-схему `busylight://` (в `Info.plist`, `CFBundleURLTypes`).
2. Пользователь (через инструкцию/первый запуск) создаёт 2 автоматизации в Shortcuts:
   - «Когда Focus *Работа* включается» → действие *Open URL* `busylight://focus/on`.
   - «Когда Focus *Работа* выключается» → `busylight://focus/off`.
3. Приложение ловит открытие URL (`NSAppleEventManager` / обработчик схемы) и обновляет `focusOn`.
4. **Fallback** (если не хотят Shortcuts): хелпер/приложение следит за файлом-флагом, который пишет shell-скрипт.

`FocusBridge` нормализует оба источника в `Boolean focusOn` → в state machine.

---

## 7. `app` — UI (Compose Desktop tray)
- Иконка в строке меню меняется по статусу: серый (IDLE) / «занят» (BUSY) / «телефон» (ON_CALL).
- Пункты меню: `Авто` (default), `Busy`, `On call`, `Custom…`, `Idle`, разделитель, индикатор соединения + `% батареи`, `Настройки…`, `Выход`.
- Ручной выбор — опционально «с таймером» (Busy 25 мин → авто-возврат в `Авто`).
- Окно настроек: выбор устройства, яркость (запись через будущую характеристику или текст), текст для `CUSTOM`, ссылка-инструкция по Shortcuts.

`AppController` подписывается на все источники (`KableCentral` override+battery, `MicCamDetector`, `FocusBridge`, ручной выбор), на каждое изменение собирает `Inputs`, гоняет через `StatusStateMachine`, и при изменении результата вызывает `KableCentral.push()`.

---

## 8. Разрешения и Info.plist
- **Bluetooth:** `NSBluetoothAlwaysUsageDescription`.
- **Микрофон:** строго говоря, `DeviceIsRunningSomewhere` НЕ требует разрешения; добавлять `NSMicrophoneUsageDescription` только если позже понадобится capture. v1 — без него.
- **URL-схема:** `CFBundleURLTypes` = `busylight`.
- Хелпер `busylight-sensors` кладётся в `Contents/Resources` бандла; путь к нему вычисляется в рантайме.

---

## 9. Упаковка и распространение (open source)
- Сборка через **Compose Gradle plugin**: `packageDmg` / `packagePkg` (jpackage). Бандлит JRE.
- Включить universal-хелпер: `appResourcesRootDir` → положить `busylight-sensors`.
- Подпись: для open source — **ad-hoc подпись** + инструкция пользователю:
  `xattr -dr com.apple.quarantine /Applications/BusyDisplay.app` (обход Gatekeeper без платного аккаунта).
  Опц.: Developer ID + нотаризация, если будет аккаунт.
- **Автозапуск:** LaunchAgent (`~/Library/LaunchAgents/com.busydisplay.agent.plist`) или ServiceManagement `SMAppService` (через хелпер).

---

## 10. Тестирование
- **Юнит:** `StatusStateMachine` (все ветки приоритетов), `BleProtocol` (кодек) — в `commonTest`, без железа.
- **Хелпер:** запуск вручную, проверить JSON при включении/выключении мика на звонке Zoom/FaceTime.
- **Интеграция:** связка Mac ↔ устройство; матрица меняется по ручному выбору и по реальному звонку.
- **Кросс-арх:** собрать и запустить и на Intel, и на Apple Silicon; `lipo -info` хелпера показывает обе арки.

## 11. Приёмочные критерии (macOS)
- [ ] Иконка в строке меню; ручное переключение пишет статус на устройство.
- [ ] Старт звонка (занят микрофон) → устройство показывает `ON CALL` без ручных действий.
- [ ] Включение Focus *Работа* → `BUSY` (через Shortcuts).
- [ ] Локальный override с устройства синхронизирует UI (Notify).
- [ ] `%` батареи отображается в меню.
- [ ] Сборка `.dmg` ставится и работает на Intel и на Apple Silicon.

## 12. Будущее
- Несколько устройств (протокол позволяет).
- Подмена Kable на CoreBluetooth-хелпер, если JVM-таргет окажется нестабилен.
- Глобальный хоткей для быстрого Busy.
