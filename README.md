[Русский](README.ru.md)

# TeleWatch Bridge ⌚📡

[![Platform](https://img.shields.io/badge/platform-Android%2012%2B%20(API%2031%2B)-green.svg)](https://android.com)
[![Kotlin](https://img.shields.io/badge/Kotlin-2.0.21-blue.svg)](https://kotlinlang.org)
[![Compose](https://img.shields.io/badge/Jetpack%20Compose-BOM%202024.12.01-4285F4.svg)](https://developer.android.com/jetpack/compose)
[![TDLib](https://img.shields.io/badge/TDLib-JNI%20v1.8-0088cc.svg)](https://core.telegram.org/tdlib)
[![Ktor](https://img.shields.io/badge/Ktor%20Server-2.3.13%20(CIO)-orange.svg)](https://ktor.io)
[![License](https://img.shields.io/badge/license-MIT-lightgrey.svg)](LICENSE)

**TeleWatch Bridge** is a lightweight, background Android application that bridges Telegram data to proprietary smartwatches, fitness trackers, and embedded IoT devices over a local Wi-Fi network.

The app runs an embedded, low-overhead HTTP REST server powered by **Ktor (CIO engine)** and integrates directly with Telegram via **official TDLib (Telegram Database Library via JNI)**. A proprietary smartwatch can fetch chat lists, message history, contacts, user profiles, compressed images, and voice notes simply by making local HTTP `GET` requests to the phone's IP address.

---

## 🌟 Key Features

* **Embedded Ktor HTTP Server:** Lightweight, non-blocking asynchronous server running strictly on `0.0.0.0:[PORT]` (default `8080`).
* **Direct TDLib Integration:** Native C++ JNI core (`libtdjni.so`) ensures fast, low-latency, and battery-efficient communication with Telegram servers.
* **Rock-Solid Background Persistence:**
  * Android Foreground Service with persistent status notification (showing local IP and port).
  * `PARTIAL_WAKE_LOCK` prevents CPU sleep when the display is off.
  * `WifiManager.WIFI_MODE_FULL_HIGH_PERF` keeps the Wi-Fi chip awake with zero packet drop.
  * `START_STICKY` ensures immediate resurrection if killed under extreme memory pressure.
* **Secure Pairing & Authentication:**
  * Cryptographically secure 6-digit PIN with a 300-second TTL generated via `SecureRandom`.
  * Unique Bearer token issued upon pairing, stored securely in `EncryptedSharedPreferences` (AES-256 GCM).
  * All API endpoints (except pairing) enforce token validation via Ktor pipeline interception.
* **On-the-Fly Media Downscaling:**
  * **Avatars:** Center-cropped and downscaled to 200×200 px PNG.
  * **Images:** Scaled preserving aspect ratio to $\le 466\text{ px}$ (matching smartwatch display dimensions), compressed as JPEG (q=85).
  * **Audio / Voice Notes:** Streamed as binary audio chunks with non-blocking **HTTP 202 Accepted + `Retry-After: 2`** polling semantics while TDLib downloads the file.
* **Firmware Update Check:** Relays the latest release version from a configured GitHub repository.
* **Modern Material 3 UI:** Built with Jetpack Compose, featuring live server toggling, IP/port display, pairing countdown, and full multi-turn Telegram authentication (Phone $\to$ Code $\to$ 2FA Password).

---

## 📐 Architecture & Data Flow

```
┌─────────────────────────┐                     ┌───────────────────────────────┐
│       Telegram          │                     │      Proprietary Watch /      │
│     Cloud Servers       │                     │         IoT Device            │
└────────────┬────────────┘                     └───────────────▲───────────────┘
             │ MTProto                                          │ Local HTTP GET
             ▼                                                  │ (e.g. 192.168.1.50:8080)
┌───────────────────────────────────────────────────────────────┴───────────────┐
│                        Android Phone (TeleWatch Bridge)                       │
│                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                    Foreground Service (WatchBridgeService)              │  │
│  │     • PowerManager.PARTIAL_WAKE_LOCK  • WifiManager.FULL_HIGH_PERF      │  │
│  │     • Persistent System Notification                                    │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                       │                                       │
│             ┌─────────────────────────┴─────────────────────────┐             │
│             ▼                                                   ▼             │
│  ┌─────────────────────────┐                         ┌─────────────────────┐  │
│  │   TDLib Client (JNI)    │                         │  Ktor Server (CIO)  │  │
│  │   • libtdjni.so         │                         │  • 0.0.0.0:8080     │  │
│  │   • TdlibRepository     │ ──── Domain Models ───► │  • TokenValidator   │  │
│  │   • SQLite / TL-Schema  │                         │  • ImageProcessor   │  │
│  └─────────────────────────┘                         └─────────────────────┘  │
│                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                       Jetpack Compose UI (MainActivity)                 │  │
│  │     • Telegram Auth Wizard (Phone / OTP / 2FA)                          │  │
│  │     • Server Controller (Start / Stop / Port configuration)             │  │
│  │     • Watch Pairing Screen (6-digit PIN with TTL countdown)             │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔐 Pairing & Security Flow

1. User opens the **Pair Watch** tab in TeleWatch Bridge on the phone.
2. A random 6-digit PIN is generated (e.g. `482910`) with a 5-minute countdown.
3. The smartwatch sends a GET request to the pairing endpoint:
   ```http
   GET http://<PHONE_IP>:8080/api/v1/auth/pair?code=482910
   ```
4. If valid, the phone generates a UUID token, stores it securely, and returns:
   ```json
   {
     "status": "ok",
     "token": "d8f3c7e0-4a2b-11ee-be56-0242ac120002"
   }
   ```
5. The smartwatch stores this token in its persistent flash memory and includes it in all subsequent requests:
   `?token=d8f3c7e0-4a2b-11ee-be56-0242ac120002`.

---

## 📡 API Reference

All protected endpoints require the `token` query parameter. If the token is missing or invalid, the server responds with `401 Unauthorized`.

### 1. Authentication & Pairing

#### `GET /api/v1/auth/pair?code={PIN}`
Exchanges a one-time 6-digit pairing code for an API bearer token. *(No token required)*

* **Response (200 OK):**
  ```json
  {
    "status": "ok",
    "token": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d"
  }
  ```
* **Response (401 Unauthorized):**
  ```json
  {
    "error": "Invalid or expired PIN. Please regenerate on the app."
  }
  ```

---

### 2. Chats & Messages

#### `GET /api/v1/chats?limit={N}&token={T}`
Returns the top `N` chats sorted by Telegram order (pinned first, then recent).

* **Parameters:** `limit` (optional, default: 20, max: 100), `token` (required).
* **Response (200 OK):**
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
Returns recent messages from a specific chat.

* **Parameters:** `chat_id` (required), `limit` (optional, default: 20, max: 100), `token` (required).
* **Response (200 OK):**
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

### 3. Contacts & User Profiles

#### `GET /api/v1/contacts?limit={N}&token={T}`
Returns Telegram contacts list.

* **Parameters:** `limit` (optional, default: 50, max: 200), `token` (required).
* **Response (200 OK):**
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
Returns user profile details. Omit `user_id` or pass `self` for the currently logged-in account.

* **Response (200 OK):**
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

### 4. Media Streaming & Downscaling

All media routes implement **smart asynchronous downloading**. If TDLib has not finished downloading the file from Telegram servers, the server returns **`HTTP 202 Accepted`** with a `Retry-After: 2` header:

```http
HTTP/1.1 202 Accepted
Retry-After: 2
Content-Type: text/plain

Downloading... retry in 2 seconds
```

Once downloaded, the file is processed and streamed directly as binary bytes:

#### `GET /api/v1/media/avatar?user_id={ID}&token={T}`
* **Returns:** Binary `image/png` cropped and downscaled to $200 \times 200\text{ px}$.

#### `GET /api/v1/media/image?file_id={ID}&token={T}`
* **Returns:** Binary `image/jpeg` downscaled so $\max(\text{width}, \text{height}) \le 466\text{ px}$ (quality 85%).

#### `GET /api/v1/media/audio?file_id={ID}&token={T}`
* **Returns:** Binary audio stream (`audio/ogg`, `audio/mpeg`, etc.) for voice notes.

---

### 5. System & Firmware Updates

#### `GET /api/v1/system/version?token={T}`
Queries the configured GitHub Releases endpoint and returns the latest firmware version tag.

* **Response (200 OK):**
  ```json
  {
    "latest_version": "1.0.4"
  }
  ```

---

## 🛠️ Build & Setup Instructions

### Prerequisites
* **JDK:** OpenJDK 17 or higher
* **Android SDK:** Platform `android-36`, Build-Tools `35.0.0` or `36.0.0`
* **Telegram API Credentials:** Obtain `api_id` and `api_hash` from [my.telegram.org](https://my.telegram.org).

### 1. Clone the Repository
```bash
git clone https://github.com/danmilk227/watchgram.git
cd watchgram/TeleWatchBridge
```

### 2. Configure Credentials
Create or edit `local.properties` in the project root:
```properties
sdk.dir=C:/Users/<username>/AppData/Local/Android/Sdk

# Telegram API Credentials
TELEGRAM_API_ID=YOUR_API_ID
TELEGRAM_API_HASH=YOUR_API_HASH

# Firmware update URL (GitHub Releases API)
WATCH_VERSION_URL=https://api.github.com/repos/danmilk227/watchgram/releases/latest
```

### 3. Build APK
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

The compiled APK will be generated at:
```
app/build/outputs/apk/debug/app-debug.apk
```

### 4. Install via ADB
```bash
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

---

## 📱 Getting Started (First Run)

1. Launch **TeleWatch Bridge** on your Android device.
2. Grant Notification permission when prompted (required for Foreground Service).
3. **Sign In:**
   * Enter your Telegram phone number (with country code, e.g. `+1 555 0100`).
   * Enter the verification code sent to your Telegram app.
   * Enter your 2FA Cloud Password (if enabled on your account).
4. **Start Server:** Tap **"Start Server"** on the home dashboard. The persistent notification will display the listening IP and port (e.g. `192.168.1.45:8080`).
5. **Pair Your Watch:**
   * Navigate to the **"Pair Watch"** tab to view your 6-digit PIN.
   * Send the pairing request from your watch or browser:
     ```bash
     curl "http://192.168.1.45:8080/api/v1/auth/pair?code=123456"
     ```
   * Save the returned `token` on your watch for subsequent API calls.

---

## ⌚ Smartwatch Client Implementation Example (C / Arduino / ESP32)

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
        // Parse JSON payload using ArduinoJson or your preferred parser
        Serial.println(payload);
    } else {
        Serial.printf("HTTP Error: %d\n", httpResponseCode);
    }
    http.end();
}
```

---

## 📂 Project Structure

```
TeleWatchBridge/
├── gradle/
│   └── libs.versions.toml             # Centralized version catalog
├── app/
│   ├── build.gradle.kts               # Dependencies, NDK ABI filters, BuildConfig
│   ├── proguard-rules.pro             # ProGuard / R8 keep rules
│   └── src/main/
│       ├── AndroidManifest.xml        # Foreground service, permissions, theme
│       ├── jniLibs/                   # Bundled native TDLib binaries
│       │   ├── arm64-v8a/             # libtdjni.so, libcryptox.so, libsslx.so
│       │   ├── armeabi-v7a/           # 32-bit ARM support
│       │   └── x86_64/                # Emulator support
│       ├── java/
│       │   ├── org/drinkless/tdlib/   # Official TDLib Java bindings (TdApi, Client)
│       │   └── com/telewatch/bridge/
│       │       ├── TeleWatchApp.kt    # Hilt application, native library loader
│       │       ├── MainActivity.kt    # Navigation host & runtime permissions
│       │       ├── auth/              # PairingManager (PIN/TTL) & TokenValidator
│       │       ├── di/                # Hilt dependency injection module
│       │       ├── media/             # Image downscaling & cropping algorithms
│       │       ├── server/            # Ktor CIO server & route handlers
│       │       │   └── routes/        # Auth, Chats, Contacts, Media, System, User
│       │       ├── service/           # WatchBridgeService (WAKELOCK + WIFI_LOCK)
│       │       ├── tdlib/             # TdlibClient coroutine bridge & TdlibRepository
│       │       └── ui/                # Compose screens: Home, Login, Pair, Theme
│       └── res/                       # Vectors, colors, strings, XML resources
├── build.gradle.kts                   # Root build script
├── settings.gradle.kts                # Project structure & repositories
└── README.md                          # Project documentation
```

---

## 🛡️ License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
TDLib is licensed under the [Boost Software License 1.0](https://www.boost.org/LICENSE_1_0.txt).
