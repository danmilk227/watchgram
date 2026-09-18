[English](README.md)

# TeleWatch Bridge ⌚📡

[![Platform](https://img.shields.io/badge/platform-Android%2012%2B%20(API%2031%2B)-green.svg)](https://android.com)
[![Kotlin](https://img.shields.io/badge/Kotlin-2.0.21-blue.svg)](https://kotlinlang.org)
[![Compose](https://img.shields.io/badge/Jetpack%20Compose-BOM%202024.12.01-4285F4.svg)](https://developer.android.com/jetpack/compose)
[![TDLib](https://img.shields.io/badge/TDLib-JNI%20v1.8-0088cc.svg)](https://core.telegram.org/tdlib)
[![Ktor](https://img.shields.io/badge/Ktor%20Server-2.3.13%20(CIO)-orange.svg)](https://ktor.io)
[![License](https://img.shields.io/badge/license-MIT-lightgrey.svg)](LICENSE)

**TeleWatch Bridge** — это легковесное фоновое Android-приложение, которое служит мостом для передачи данных Telegram на проприетарные смарт-часы, фитнес-трекеры и встраиваемые IoT-устройства по локальной сети Wi-Fi.

Приложение запускает встроенный HTTP REST сервер с низкими накладными расходами на базе **Ktor (движок CIO)** и интегрируется напрямую с Telegram через **официальную библиотеку TDLib (Telegram Database Library через JNI)**. Проприетарные смарт-часы могут получать списки чатов, историю сообщений, контакты, профили пользователей, сжатые изображения и голосовые сообщения, просто делая локальные HTTP `GET` запросы на IP-адрес телефона.

---

## 🌟 Основные возможности

* **Встроенный HTTP-сервер Ktor:** Легковесный, неблокирующий асинхронный сервер, работающий строго на `0.0.0.0:[PORT]` (по умолчанию `8080`).
* **Прямая интеграция с TDLib:** Нативное ядро C++ JNI (`libtdjni.so`) обеспечивает быструю связь с серверами Telegram с низкой задержкой и низким энергопотреблением.
* **Надежная работа в фоне:**
  * Android Foreground Service (служба переднего плана) с постоянным уведомлением о статусе (показывает локальный IP и порт).
  * `PARTIAL_WAKE_LOCK` предотвращает засыпание процессора при выключенном экране.
  * `WifiManager.WIFI_MODE_FULL_HIGH_PERF` поддерживает Wi-Fi чип в активном состоянии без потери пакетов.
  * `START_STICKY` обеспечивает немедленный перезапуск, если приложение было убито из-за нехватки памяти.
* **Безопасное сопряжение и аутентификация:**
  * Криптографически надежный 6-значный PIN-код со временем жизни 300 секунд, генерируемый через `SecureRandom`.
  * Уникальный токен Bearer выдается после сопряжения и надежно хранится в `EncryptedSharedPreferences` (AES-256 GCM).
  * Все эндпоинты API (кроме сопряжения) требуют валидации токена через перехватчик Ktor (pipeline interception).
* **Сжатие медиа на лету:**
  * **Аватары:** Обрезаются по центру и уменьшаются до 200×200 px в формате PNG.
  * **Изображения:** Масштабируются с сохранением пропорций до $\le 466\text{ px}$ (соответствует размерам дисплея смарт-часов), сжимаются в JPEG (q=85).
  * **Аудио / Голосовые сообщения:** Передаются в виде потока бинарных аудио-фрагментов с неблокирующей семантикой опроса **HTTP 202 Accepted + `Retry-After: 2`**, пока TDLib скачивает файл.
* **Проверка обновления прошивки:** Передает последнюю версию релиза из настроенного репозитория GitHub.
* **Современный UI в стиле Material 3:** Построен с использованием Jetpack Compose, включает в себя переключатель сервера в реальном времени, отображение IP/порта, обратный отсчет сопряжения и полноценную многошаговую аутентификацию в Telegram (Телефон $\to$ Код $\to$ 2FA Пароль).

---

## 📐 Архитектура и поток данных

```
┌─────────────────────────┐                     ┌───────────────────────────────┐
│        Облачные         │                     │     Проприетарные часы /      │
│    Серверы Telegram     │                     │         IoT устройство        │
└────────────┬────────────┘                     └───────────────▲───────────────┘
             │ MTProto                                          │ Локальный HTTP GET
             ▼                                                  │ (напр. 192.168.1.50:8080)
┌───────────────────────────────────────────────────────────────┴───────────────┐
│                        Android Телефон (TeleWatch Bridge)                     │
│                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                    Foreground Service (WatchBridgeService)              │  │
│  │     • PowerManager.PARTIAL_WAKE_LOCK  • WifiManager.FULL_HIGH_PERF      │  │
│  │     • Постоянное системное уведомление                                  │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                       │                                       │
│             ┌─────────────────────────┴─────────────────────────┐             │
│             ▼                                                   ▼             │
│  ┌─────────────────────────┐                         ┌─────────────────────┐  │
│  │   TDLib Client (JNI)    │                         │  Ktor Server (CIO)  │  │
│  │   • libtdjni.so         │                         │  • 0.0.0.0:8080     │  │
│  │   • TdlibRepository     │ ──── Доменные модели ─► │  • TokenValidator   │  │
│  │   • SQLite / TL-Schema  │                         │  • ImageProcessor   │  │
│  └─────────────────────────┘                         └─────────────────────┘  │
│                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                       Jetpack Compose UI (MainActivity)                 │  │
│  │     • Мастер авторизации Telegram (Телефон / OTP / 2FA)                 │  │
│  │     • Контроллер сервера (Старт / Стоп / Настройка порта)               │  │
│  │     • Экран сопряжения часов (6-значный PIN-код с таймером)             │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔐 Сопряжение и безопасность

1. Пользователь открывает вкладку **Pair Watch** (Сопряжение) в приложении TeleWatch Bridge на телефоне.
2. Генерируется случайный 6-значный PIN-код (например, `482910`) с обратным отсчетом 5 минут.
3. Смарт-часы отправляют GET-запрос на эндпоинт сопряжения:
   ```http
   GET http://<PHONE_IP>:8080/api/v1/auth/pair?code=482910
   ```
4. В случае успеха телефон генерирует UUID токен, безопасно сохраняет его и возвращает:
   ```json
   {
     "status": "ok",
     "token": "d8f3c7e0-4a2b-11ee-be56-0242ac120002"
   }
   ```
5. Смарт-часы сохраняют этот токен в своей энергонезависимой флэш-памяти и включают его во все последующие запросы:
   `?token=d8f3c7e0-4a2b-11ee-be56-0242ac120002`.

---

## 📡 API Справочник

Все защищенные эндпоинты требуют наличие query-параметра `token`. Если токен отсутствует или недействителен, сервер отвечает `401 Unauthorized`.

### 1. Аутентификация и сопряжение

#### `GET /api/v1/auth/pair?code={PIN}`
Обменивает одноразовый 6-значный код сопряжения на токен API (Bearer). *(Токен не требуется)*

* **Ответ (200 OK):**
  ```json
  {
    "status": "ok",
    "token": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d"
  }
  ```
* **Ответ (401 Unauthorized):**
  ```json
  {
    "error": "Invalid or expired PIN. Please regenerate on the app."
  }
  ```

---

### 2. Чаты и сообщения

#### `GET /api/v1/chats?limit={N}&token={T}`
Возвращает топ `N` чатов, отсортированных в порядке Telegram (сначала закрепленные, затем последние).

* **Параметры:** `limit` (необязательно, по умолчанию: 20, максимум: 100), `token` (обязательно).
* **Ответ (200 OK):**
  ```json
  {
    "chats": [
      {
        "id": -1001234567890,
        "title": "Embedded Developers",
        "unreadCount": 3,
        "lastMessage": "Alex: Firmware v1.0.4 is ready for testing!",
        "timestamp": 1726665420,
        "avatarAvailable": true
      }
    ],
    "count": 1
  }
  ```

#### `GET /api/v1/messages?chat_id={ID}&limit={N}&token={T}`
Возвращает последние сообщения из определенного чата.

* **Параметры:** `chat_id` (обязательно), `limit` (необязательно, по умолчанию: 20, максимум: 100), `token` (обязательно).
* **Ответ (200 OK):**
  ```json
  {
    "messages": [
      {
        "id": 542109,
        "chatId": -1001234567890,
        "senderId": 987654321,
        "timestamp": 1726665420,
        "contentType": "text",
        "text": "Firmware v1.0.4 is ready for testing!",
        "mediaFileId": null
      },
      {
        "id": 542110,
        "chatId": -1001234567890,
        "senderId": 987654321,
        "timestamp": 1726665450,
        "contentType": "photo",
        "text": "[Photo]",
        "mediaFileId": 14205
      }
    ],
    "count": 2
  }
  ```

---

### 3. Контакты и профили пользователей

#### `GET /api/v1/contacts?limit={N}&token={T}`
Возвращает список контактов Telegram.

* **Параметры:** `limit` (необязательно, по умолчанию: 50, максимум: 200), `token` (обязательно).
* **Ответ (200 OK):**
  ```json
  {
    "contacts": [
      {
        "userId": 12345678,
        "displayName": "John Doe",
        "online": true,
        "lastSeenTs": 1726665500
      }
    ],
    "count": 1
  }
  ```

#### `GET /api/v1/user/info?user_id={ID|self}&token={T}`
Возвращает детали профиля пользователя. Опустите `user_id` или передайте `self` для текущего авторизованного аккаунта.

* **Ответ (200 OK):**
  ```json
  {
    "id": 12345678,
    "username": "johndoe",
    "firstName": "John",
    "lastName": "Doe",
    "bio": "Building wearable tech 🚀"
  }
  ```

---

### 4. Стриминг и сжатие медиа

Все маршруты для медиа реализуют **умное асинхронное скачивание**. Если TDLib еще не завершил загрузку файла с серверов Telegram, сервер возвращает **`HTTP 202 Accepted`** с заголовком `Retry-After: 2`:

```http
HTTP/1.1 202 Accepted
Retry-After: 2
Content-Type: text/plain

Downloading... retry in 2 seconds
```

После завершения загрузки файл обрабатывается и передается напрямую в виде потока байтов:

#### `GET /api/v1/media/avatar?user_id={ID}&token={T}`
* **Возвращает:** Бинарный `image/png`, обрезанный и уменьшенный до $200 \times 200\text{ px}$.

#### `GET /api/v1/media/image?file_id={ID}&token={T}`
* **Возвращает:** Бинарный `image/jpeg`, масштабированный так, чтобы $\max(\text{ширина}, \text{высота}) \le 466\text{ px}$ (качество 85%).

#### `GET /api/v1/media/audio?file_id={ID}&token={T}`
* **Возвращает:** Бинарный аудио-поток (`audio/ogg`, `audio/mpeg` и т.д.) для голосовых сообщений.

---

### 5. Система и обновления прошивки

#### `GET /api/v1/system/version?token={T}`
Опрашивает настроенный эндпоинт GitHub Releases и возвращает тег последней версии прошивки.

* **Ответ (200 OK):**
  ```json
  {
    "latest_version": "1.0.4"
  }
  ```

---

## 🛠️ Сборка и настройка

### Предварительные требования
* **JDK:** OpenJDK 17 или выше
* **Android SDK:** Platform `android-36`, Build-Tools `35.0.0` или `36.0.0`
* **Telegram API Credentials:** Получите `api_id` и `api_hash` на сайте [my.telegram.org](https://my.telegram.org).

### 1. Клонирование репозитория
```bash
git clone https://github.com/danmilk227/watchgram.git
cd watchgram/TeleWatchBridge
```

### 2. Настройка учетных данных
Создайте или отредактируйте `local.properties` в корне проекта:
```properties
sdk.dir=C:/Users/<username>/AppData/Local/Android/Sdk

# Telegram API Credentials
TELEGRAM_API_ID=YOUR_API_ID
TELEGRAM_API_HASH=YOUR_API_HASH

# URL для обновления прошивки (GitHub Releases API)
WATCH_VERSION_URL=https://api.github.com/repos/danmilk227/watchgram/releases/latest
```

### 3. Сборка APK
* **Windows (PowerShell):**
  ```powershell
  $env:JAVA_HOME = "C:\Path\To\jdk-17"
  .\gradlew.bat assembleDebug
  ```
* **Linux / macOS:**
  ```bash
  export JAVA_HOME=/path/to/jdk-17
  ./gradlew assembleDebug
  ```

Скомпилированный APK будет сгенерирован по пути:
```
app/build/outputs/apk/debug/app-debug.apk
```

### 4. Установка через ADB
```bash
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

---

## 📱 Начало работы (Первый запуск)

1. Запустите **TeleWatch Bridge** на вашем Android устройстве.
2. Предоставьте разрешение на уведомления при запросе (требуется для Foreground Service).
3. **Авторизация:**
   * Введите ваш номер телефона Telegram (с кодом страны, например `+1 555 0100`).
   * Введите код подтверждения, отправленный в ваше приложение Telegram.
   * Введите ваш облачный пароль 2FA (если включен для вашего аккаунта).
4. **Запуск сервера:** Нажмите **"Start Server"** на главной панели управления. Постоянное уведомление отобразит IP-адрес и порт для прослушивания (например, `192.168.1.45:8080`).
5. **Сопряжение часов:**
   * Перейдите на вкладку **"Pair Watch"**, чтобы увидеть ваш 6-значный PIN-код.
   * Отправьте запрос на сопряжение с ваших часов или из браузера:
     ```bash
     curl "http://192.168.1.45:8080/api/v1/auth/pair?code=123456"
     ```
   * Сохраните возвращенный `token` на ваших часах для последующих вызовов API.

---

## ⌚ Пример реализации клиента для смарт-часов (C / Arduino / ESP32)

```c
#include <WiFi.h>
#include <HTTPClient.h>

const char* serverUrl = "http://192.168.1.45:8080/api/v1/chats?limit=5&token=YOUR_TOKEN_HERE";

void fetchChats() {
    HTTPClient http;
    http.begin(serverUrl);
    int httpResponseCode = http.GET();

    if (httpResponseCode == 200) {
        String payload = http.getString();
        // Парсинг JSON-ответа с использованием ArduinoJson или вашего парсера
        Serial.println(payload);
    } else {
        Serial.printf("HTTP Error: %d\n", httpResponseCode);
    }
    http.end();
}
```

---

## 📂 Структура проекта

```
TeleWatchBridge/
├── gradle/
│   └── libs.versions.toml             # Централизованный каталог версий
├── app/
│   ├── build.gradle.kts               # Зависимости, NDK ABI фильтры, BuildConfig
│   ├── proguard-rules.pro             # Правила ProGuard / R8
│   └── src/main/
│       ├── AndroidManifest.xml        # Foreground service, разрешения, темы
│       ├── jniLibs/                   # Встроенные бинарные файлы TDLib
│       │   ├── arm64-v8a/             # libtdjni.so, libcryptox.so, libsslx.so
│       │   ├── armeabi-v7a/           # Поддержка 32-bit ARM
│       │   └── x86_64/                # Поддержка эмуляторов
│       ├── java/
│       │   ├── org/drinkless/tdlib/   # Официальные Java-бинднги TDLib (TdApi, Client)
│       │   └── com/telewatch/bridge/
│       │       ├── TeleWatchApp.kt    # Приложение Hilt, загрузчик нативных библиотек
│       │       ├── MainActivity.kt    # Хост навигации и разрешения времени выполнения
│       │       ├── auth/              # PairingManager (PIN/TTL) и TokenValidator
│       │       ├── di/                # Модуль внедрения зависимостей Hilt
│       │       ├── media/             # Алгоритмы уменьшения и обрезки изображений
│       │       ├── server/            # Ktor CIO сервер и обработчики маршрутов
│       │       │   └── routes/        # Auth, Chats, Contacts, Media, System, User
│       │       ├── service/           # WatchBridgeService (WAKELOCK + WIFI_LOCK)
│       │       ├── tdlib/             # TdlibClient coroutine bridge и TdlibRepository
│       │       └── ui/                # Экраны Compose: Home, Login, Pair, Theme
│       └── res/                       # Векторы, цвета, строки, XML-ресурсы
├── build.gradle.kts                   # Корневой скрипт сборки
├── settings.gradle.kts                # Структура проекта и репозитории
└── README.md                          # Документация проекта
```

---

## 🛡️ Лицензия

Этот проект лицензируется на условиях MIT License - подробности смотрите в файле [LICENSE](LICENSE).
TDLib лицензируется на условиях [Boost Software License 1.0](https://www.boost.org/LICENSE_1_0.txt).
