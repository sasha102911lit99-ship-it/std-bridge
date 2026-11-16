# ANDROID SCREEN CAPTURE BRIDGE

Компонент захвата экрана Android → кадры в браузерный ИИ по защищённому WebSocket.
Управление игрой НЕ реализовано сознательно (только передача изображения).

## Архитектура
```
Roblox (экран телефона)
  → MediaProjection → VirtualDisplay 720p → ImageReader
  → JPEG q55, ~12 fps → WsServer (wss://:8443, ?token=) ──Wi-Fi/USB──▶
  → вкладка «📡 Мост» генератора → canvas → (опционально) TEST VISION
```
TLS обязателен: страница генератора работает по https, браузер блокирует
незашифрованный ws:// как mixed content. Сертификат самоподписанный,
генерируется в приложении при первом запуске (SAN: localhost, 127.0.0.1, IP телефона).

## Сборка APK
Вариант A — Android Studio: открыть папку `src/android`, дождаться синхронизации
Gradle, Run ▶ на подключённом телефоне (minSdk 26).
Вариант B — командная строка (нужны JDK 17 + Android SDK):
```
cd src/android
chmod +x gradlew   # на Linux/macOS, один раз
./gradlew assembleDebug
# APK: app/build/outputs/apk/debug/app-debug.apk
adb install -r app/build/outputs/apk/debug/app-debug.apk
```
Вариант C — облако: создай пустой GitHub-репозиторий, залей в его корень
СОДЕРЖИМОЕ папки `src/android` (gradlew, settings.gradle, папка app/…)
плюс файл `.github/workflows/android-build.yml` из генератора (готов, лежит рядом).
APK соберётся в Actions автоматически при push: вкладка Actions → прогон →
Artifacts → `std-bridge-apk`. Лог компиляции — там же, в шаге Build debug APK.

## Сенсорное управление — TEST TOUCH (новые файлы)
- `TouchService.kt` — AccessibilityService, жесты через GestureDescription
  (tap/hold/drag, длительность 1–2000мс). Без включённой службы в настройках —
  никаких действий.
- `TouchController` (там же) — защиты: взвод armed (по умолчанию НЕТ, снимается
  кнопкой STOP/разрывом/выходом), минимум 400мс между жестами, максимум 40/мин,
  очередь 1 (иначе `busy`), таймаут жеста 3с, координаты только 0..1.
- `res/xml/touch_service_config.xml` + `res/values/strings.xml` + службы/активити
  в `AndroidManifest.xml` (BIND_ACCESSIBILITY_SERVICE).
- `TestTouchActivity.kt` — экран TEST TOUCH: пурпурная кнопка #D62598 со счётчиком
  (её ИИ находит в кадре) + большая красная STOP.
- `MainActivity` — чекбокс взвода (одно касание), кнопки настроек спец.
  возможностей и экрана TEST TOUCH, статус сервиса жестов.
- `WsServer` — хук `onText`; `ScreenService.handleCommand` — протокол:
  `arm` / `touch` / `info` (screenW/H, orientation) / `stopAll`, ответы `result`
  (`ok`, `ms`, `error`: disarmed/busy/rate/no_service/bad_coords/…).
- Включение: Настройки → Спец. возможности → STD Bridge → вкл; затем взвод
  в приложении или из браузера (каждое действие требует взвода заново).

## Подключение
1. Телефон и ПК в одной Wi-Fi-сети (или USB + `adb forward tcp:8443 tcp:8443`,
   тогда адрес — `127.0.0.1`).
2. Открой приложение, выдай разрешение на захват экрана, нажми «Старт».
3. Открой в браузере `https://<IP-телефона>:8443/` → прими предупреждение
   сертификата (сервер отдаёт 404 — это нормально, важен сам факт доверия).
4. Во вкладке «📡 Мост» введи IP, порт 8443, токен с экрана телефона → «Подключить».
5. Включи «Передавать кадры в TEST VISION» — детекторы заработают на живом кадре.

## Протокол
- `$ wss://host:8443/?token=XXXX` — binary: JPEG-кадр; текст: JSON.
- Сервер → клиент: `{"type":"hello","w":1280,"h":720,"fps":12}` при подключении.
- Клиент → сервер: `{"type":"ping"}` → ответ `{"type":"pong"}`.
- Неверный токен → закрытие 4401. Потеря соединения — клиент переподключается
  с backoff 1→15с; остановка — кнопка «Стоп» в приложении и «Отключить» в браузере.

## Файлы
- `settings.gradle`, `build.gradle`, `app/build.gradle`, `gradle.properties` — сборка (AGP 8.x, Java-WebSocket 1.5.6, BouncyCastle для сертификата).
- `gradlew`, `gradlew.bat`, `gradle/wrapper/` — wrapper Gradle 8.9 (jar проверен, настоящий).
- `app/src/main/AndroidManifest.xml` — разрешения и сервис.
- `java/com/stdbridge/MainActivity.kt` — запрос MediaProjection, IP/токен/качество, старт/стоп.
- `java/com/stdbridge/ScreenService.kt` — foreground-сервис: VirtualDisplay → JPEG → broadcast.
- `java/com/stdbridge/WsServer.kt` — wss-сервер, токен-авторизация, heartbeat.
- `java/com/stdbridge/CertUtil.kt` — генерация/хранение самоподписанного сертификата.
- `res/layout/activity_main.xml` — интерфейс приложения.
