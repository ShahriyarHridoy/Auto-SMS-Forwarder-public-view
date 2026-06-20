# Auto-SMS-Forwarder-public-view

## Auto SMS Forwarder (Ghost Rider)

A Flutter + Native Android application that automatically forwards incoming SMS messages to configured destinations with 24/7 background operation. Supports forwarding via SMS, HTTP API, or Google Sheets — even when the app is closed or the device has rebooted.

https://private-user-images.githubusercontent.com/35798678/610718023-2d232b76-c81b-4947-b296-b449d0992c31.mp4?jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3ODE5MTY4ODEsIm5iZiI6MTc4MTkxNjU4MSwicGF0aCI6Ii8zNTc5ODY3OC82MTA3MTgwMjMtMmQyMzJiNzYtYzgxYi00OTQ3LWIyOTYtYjQ0OWQwOTkyYzMxLm1wND9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNjA2MjAlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjYwNjIwVDAwNDk0MVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTA1YTZjZDBiZGE5OGVmY2YwYTljMTZlOTVjYTcxZjMzNjNjNDE4MWUwODljZGM2MmIxNmIxMDUyZWQ0ZmI2MDAmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JnJlc3BvbnNlLWNvbnRlbnQtdHlwZT12aWRlbyUyRm1wNCJ9.E5-X3R8BNe0qo1JyfnSdc9QGgeJccmOCMsjaRz1mZpE

#### note: in this vodeo sms forwarding shows failed because the http API is invalid right now. 
---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [How It Works](#how-it-works)
- [Architecture](#architecture)
- [Forwarding Modes](#forwarding-modes)
- [Tech Stack](#tech-stack)
- [Android Permissions](#android-permissions)
- [Project Structure](#project-structure)
- [Data Flow](#data-flow)
- [Remote Configuration](#remote-configuration)
- [API Payload Format](#api-payload-format)
- [Multi-SIM Support](#multi-sim-support)
- [Message Logging](#message-logging)
- [Background & Keep-Alive Strategy](#background--keep-alive-strategy)
- [Setup & Configuration](#setup--configuration)
- [Build Requirements](#build-requirements)

---

## Overview

Auto SMS Forwarder intercepts every incoming SMS at the native Android level and forwards it to a configured destination in real time. It is designed to run continuously in the background — surviving app swipes, low-memory conditions, and device reboots — making it suitable for unattended monitoring devices (e.g., IoT SIM cards, remote field phones, OTP relay setups).

The app uses a **native-first approach**: SMS interception is handled entirely in Kotlin without depending on the Flutter runtime, which ensures zero-lag forwarding and operation when the app UI is closed.

---

## Features

- **24/7 Background Operation** — Foreground service with wake lock and auto-restart keeps the forwarder alive.
- **SMS Interception at Highest Priority** — `BroadcastReceiver` registered with priority 1000 intercepts SMS before any other app.
- **Three Forwarding Modes** — Forward via SMS, HTTP API, or Google Sheets (Google Apps Script).
- **Remote Configuration** — API endpoints and intervals are fetched dynamically from a Google Sheets backend, allowing changes without an app update.
- **Multi-SIM Support** — Captures which SIM received the message, along with slot index, phone number, and carrier name.
- **Online Status Reporting** — Periodically POSTs a heartbeat to a configured API so you can monitor device connectivity.
- **Device Boot Survival** — `BootReceiver` restarts the forwarding service after a reboot.
- **Message Logging** — Full log of received messages with forwarding status (pending → forwarded/failed), persisted to shared storage and viewable in the app UI.
- **Duplicate Prevention** — Multi-layer deduplication at the receiver level and the log level prevents double-forwarding.
- **Battery Optimization Exemption** — Requests `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` so Android does not throttle or kill the service.
- **Setup Wizard** — Guided configuration screen with dedicated UI for each forwarding mode.

---

## How It Works

### SMS Interception

When an SMS arrives, Android broadcasts a `SMS_RECEIVED` intent to all registered receivers. `NativeSmsReceiver` is registered with the highest possible priority (`1000`), so it receives the broadcast before any other application. It immediately:

1. Extracts the sender number, message body, and subscription (SIM) details.
2. Checks a deduplication table to ensure the message has not already been processed.
3. Marks the message as processed.
4. Launches `SmsForwardingWorker` in a Kotlin coroutine to handle the actual forwarding asynchronously.

This entire path is native Kotlin — it does not require the Flutter engine to be running.

### Forwarding

`SmsForwardingWorker` reads the active forwarding mode and destination from `SharedPreferences` (populated by the Flutter UI when the user configures the app), then executes one of three forwarding strategies:

| Mode | Mechanism |
|------|-----------|
| 0 — SMS | Sends an SMS to a configured phone number using Android's `SmsManager`. |
| 1 — API | POSTs a JSON payload to a user-configured HTTP endpoint (e.g., Google Apps Script, webhook, custom backend). |
| 2 — Google Sheets | POSTs to a Google Apps Script URL that writes directly into a Google Sheet. |

After forwarding, the worker updates the message's status in `SharedPreferences` (`forwarded` or `failed`). The Flutter UI polls this shared storage every 500 ms and updates the message list in real time.

### Keep-Alive

`BackgroundService` runs as an Android **Foreground Service** with a persistent notification. It acquires a `PARTIAL_WAKE_LOCK` to prevent the CPU from sleeping. If the system kills it (low memory) or the user swipes the app, `onDestroy()` and `onTaskRemoved()` both trigger a restart. After a device reboot, `BootReceiver` starts the service again automatically.

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│                  Flutter UI (Dart)               │
│                                                  │
│  HomeScreen ─ SetupScreen ─ SettingsScreen       │
│       │              │              │             │
│  MessageLogService  SettingsService RemoteConfig  │
│  (polls every 500ms) (SharedPrefs)  Service       │
└──────────────────────┬──────────────────────────┘
                       │ MethodChannel / SharedPreferences
┌──────────────────────┴──────────────────────────┐
│               Android Native (Kotlin)            │
│                                                  │
│  NativeSmsReceiver (priority 1000)               │
│       └─► SmsForwardingWorker (coroutine)        │
│              ├─ forwardBySms()                   │
│              └─ forwardToApi()                   │
│                                                  │
│  BackgroundService (Foreground Service)          │
│       ├─ WakeLock                                │
│       ├─ Keep-alive ping (every 60s)             │
│       └─ OnlineStatusWorker (every 5 min)        │
│                                                  │
│  BootReceiver ─ ScreenLockReceiver               │
│  RemoteConfigPlugin ─ SharedPreferencesPlugin    │
└─────────────────────────────────────────────────┘
           │ Shared SharedPreferences
           │ (key: flutter.sms_message_log)
           └─► Unified message log (JSON array)
```

### Flutter ↔ Native Communication

- **MethodChannels** — Flutter invokes native plugins to update forwarding settings, start/stop the background service, and request battery optimization exemption.
- **SharedPreferences bridge** — Both Flutter and native code read/write from the same `SharedPreferences` file. Native code logs messages; Flutter reads and displays them. Native code reads user-configured settings; Flutter writes them.

---

## Forwarding Modes

### Mode 0 — SMS Forward

The app sends the received SMS content to a designated phone number using the Android `SmsManager` API. No internet connection is required.

**Use case:** Relay SMS from a field SIM to a personal number.

### Mode 1 — API Forward

The app POSTs a JSON body to a user-provided HTTP endpoint every time an SMS is received. The endpoint can be any webhook-compatible service — a Google Apps Script, a custom REST API, n8n, Make, etc.

**Use case:** Integrate SMS data into a backend system, CRM, or automation workflow.

### Mode 2 — Google Sheets

The app POSTs to a published Google Apps Script URL that appends a row to a Google Sheet. This requires a Google Apps Script deployment configured to accept POST requests.

**Use case:** Log all SMS messages to a Google Sheet for auditing or monitoring.

---

## Tech Stack

### Flutter / Dart

| Package | Purpose |
|---------|---------|
| `provider ^6.1.5` | State management across the widget tree |
| `shared_preferences ^2.5.3` | Persisting user settings and message log |
| `permission_handler ^12.0.1` | Runtime SMS, phone, and notification permissions |
| `flutter_background_service ^5.1.0` | Flutter-level background service integration |
| `flutter_secure_storage ^9.2.4` | Secure storage for sensitive credentials |
| `device_info_plus ^12.2.0` | Device metadata for diagnostics |
| `http ^1.1.0` | HTTP requests for API and remote config calls |
| `cupertino_icons ^1.0.8` | Icon assets |

### Android / Kotlin

| Component | Details |
|-----------|---------|
| Kotlin Coroutines | Async SMS processing without blocking the main thread |
| `SmsManager` | Native SMS sending for mode 0 |
| `SubscriptionManager` | Multi-SIM slot, number, and carrier info |
| `BroadcastReceiver` | SMS interception (priority 1000) |
| `Service` | Foreground service for 24/7 operation |
| `PowerManager.WakeLock` | Prevents CPU sleep during background processing |
| `SharedPreferences` | Shared storage bridge between Flutter and native |

### External Services

| Service | Role |
|---------|------|
| Google Sheets + Apps Script | Remote configuration source and optional message destination |
| Custom HTTP Endpoints | User-defined SMS forwarding and online status targets |

---

## Android Permissions

| Permission | Reason |
|-----------|--------|
| `RECEIVE_SMS` | Intercept incoming SMS messages |
| `READ_SMS` | Read SMS content |
| `SEND_SMS` | Forward SMS via SmsManager (mode 0) |
| `READ_PHONE_STATE` | Access SIM slot and subscription info |
| `FOREGROUND_SERVICE` | Run persistent foreground service |
| `FOREGROUND_SERVICE_DATA_SYNC` | Foreground service sub-type (Android 14+) |
| `WAKE_LOCK` | Prevent CPU sleep during processing |
| `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` | Exempt the app from Doze mode |
| `SCHEDULE_EXACT_ALARM` | Precise keep-alive scheduling |
| `RECEIVE_BOOT_COMPLETED` | Restart service after device reboot |
| `POST_NOTIFICATIONS` | Show foreground service notification (Android 13+) |
| `INTERNET` | API and remote config HTTP calls |
| `ACCESS_NETWORK_STATE` | Check connectivity before sending |

---

## Project Structure

```
Auto-SMS-Forwarder/
├── lib/
│   ├── main.dart                        # Entry point, permission check, routing
│   ├── models/
│   │   ├── settings_model.dart          # ForwardingMode enum, filter lists
│   │   ├── sms_message_model.dart       # Message data + MessageStatus enum
│   │   └── remote_config_model.dart     # Remote config schema
│   ├── services/
│   │   ├── settings_service.dart        # SharedPreferences CRUD for settings
│   │   ├── remote_config_service.dart   # Fetch & cache Google Sheets config
│   │   ├── message_log_service.dart     # Poll shared log, notify UI
│   │   ├── sms_service.dart             # SMS monitoring + API forwarding
│   │   └── online_status_service.dart   # Periodic heartbeat to status API
│   ├── screens/
│   │   ├── home_screen.dart             # Dashboard + message list
│   │   ├── setup_screen.dart            # First-run configuration wizard
│   │   ├── messages_screen.dart         # Full message history
│   │   ├── settings_screen.dart         # Edit forwarding settings
│   │   └── permission_aware_widget.dart # Permission prompt UI
│   ├── helpers/
│   │   └── battery_optimization_helper.dart
│   └── external/
│       ├── sms_flutter.dart             # Custom SMS plugin interface
│       ├── sms_flutter_method_channel.dart
│       └── sms_flutter_platform_interface.dart
│
└── android/app/src/main/
    ├── kotlin/com/ghostrider/app/
    │   ├── MainActivity.kt              # Flutter activity, plugin registration
    │   ├── SmsApplication.kt           # Application class, Flutter engine init
    │   ├── NativeSmsReceiver.kt        # SMS BroadcastReceiver (priority 1000)
    │   ├── SmsForwardingWorker.kt      # Core forwarding logic (coroutine)
    │   ├── BackgroundService.kt        # Foreground service, wake lock, keep-alive
    │   ├── BootReceiver.kt             # Auto-start on device boot
    │   ├── OnlineStatusWorker.kt       # Native heartbeat sender
    │   ├── RemoteConfigPlugin.kt       # MethodChannel for config sync
    │   ├── BatteryOptimizationPlugin.kt
    │   ├── SharedPreferencesReaderPlugin.kt
    │   ├── NativeServicePlugin.kt      # Service start/stop from Flutter
    │   └── ScreenLockReceiver.kt       # Keep-alive on screen lock
    └── AndroidManifest.xml
```

---

## Data Flow

### App Startup

```
main()
├── RemoteConfigService.initialize()
│   └── Fetch from Google Sheets → cache for 1 hour
├── Push remote config to native via MethodChannel (RemoteConfigPlugin)
├── Sync user forwarding settings to native SharedPreferences
└── Check permissions
    ├── Not granted → Permission screen
    ├── Not configured → SetupScreen
    └── Configured → HomeScreen
```

### SMS Received → Forwarded → UI Updated

```
Incoming SMS
└── NativeSmsReceiver.onReceive()         [Kotlin, priority 1000]
    ├── Extract sender, body, SIM info
    ├── Deduplicate (last 100 message IDs)
    └── SmsForwardingWorker (coroutine)
        ├── Check log for duplicates (10-second window)
        ├── Write message JSON to SharedPreferences (status: pending)
        ├── Forward based on mode
        │   ├── Mode 0 → SmsManager.sendTextMessage()
        │   ├── Mode 1 → HTTP POST to API URL
        │   └── Mode 2 → HTTP POST to Google Sheets URL
        └── Update message status (forwarded / failed)
                    ↓
MessageLogService (Flutter, polls every 500ms)
    ├── Reads SharedPreferences
    ├── Detects change via hashCode
    └── Notifies HomeScreen → rebuilds message list
```

### Keep-Alive Lifecycle

```
App launched → BackgroundService.onCreate()
    ├── Acquire PARTIAL_WAKE_LOCK
    ├── Show persistent notification
    ├── Heartbeat: update notification every 60s
    └── OnlineStatusWorker: POST heartbeat every 5 min

App swiped → onTaskRemoved() → restart service
Service killed → onDestroy() → restart service
Device reboot → BootReceiver → start BackgroundService
```

---

## Remote Configuration

The app fetches a configuration document from a Google Apps Script URL at startup and caches it for 1 hour. This allows you to change API endpoints, forwarding intervals, and feature flags without pushing an app update.

**Supported config keys:**

| Key | Type | Description |
|-----|------|-------------|
| `sms_forward_api_url` | String | Default SMS forwarding endpoint |
| `online_status_api_url` | String | Default online status endpoint |
| `online_status_interval_minutes` | Int | How often to send heartbeat (default: 5) |
| `enable_native_forwarding` | Bool | Toggle native forwarding |
| `app_version` | String | Latest app version |
| `force_update` | Bool | Block old versions |

The config supports three response formats: a direct JSON object, a key-value array (`[{"Key": "...", "Value": "..."}]`), or a nested `data` object. If the fetch fails, the app falls back to the last cached config, then to built-in defaults.

**URL resolution priority (for API mode):**

1. User-configured URL (entered in setup screen)
2. Remote config URL (from Google Sheets)
3. Hardcoded default

---

## API Payload Format

### SMS Forwarding (POST)

```json
{
  "sender": "+1234567890",
  "message": "Your OTP is 123456",
  "mobileNumber": "+9876543210",
  "timestamp": "2024-01-15T10:30:45.000Z"
}
```

- `sender` — The number that sent the original SMS.
- `message` — Full SMS body.
- `mobileNumber` — The SIM's own phone number (which received the SMS).
- `timestamp` — ISO 8601 UTC timestamp.

A `200–299` HTTP response code is treated as success. The request times out after 30 seconds.

### Online Status Heartbeat (POST)

```json
{
  "mobileNumber": "+9876543210",
  "mobileNumber2": "+1122334455"
}
```

- `mobileNumber` — SIM slot 0 phone number.
- `mobileNumber2` — SIM slot 1 phone number (dual-SIM devices).

---

## Multi-SIM Support

The app uses Android's `SubscriptionManager` API to extract per-SIM metadata at the time each SMS is received:

- **SIM slot index** (0 or 1)
- **Phone number** associated with the SIM
- **Carrier / operator name**
- **Display name** of the subscription

This information is included in the forwarded payload and stored in the message log, so you always know which SIM received a message on a dual-SIM device. The online status heartbeat also reports both SIM numbers simultaneously.

---

## Message Logging

All intercepted messages are stored in a shared JSON array under the `SharedPreferences` key `flutter.sms_message_log` (max 100 entries, newest first).

Each entry includes:

```json
{
  "sender": "+1234567890",
  "body": "Hello world",
  "timestamp": "2024-01-15T10:30:45.000Z",
  "status": "forwarded",
  "simSlot": 0,
  "simNumber": "+9876543210",
  "carrierName": "Operator Name",
  "messageForword": "Success",
  "failureReason": null
}
```

**Status values:** `pending` → `forwarded` | `failed` | `notForwarded`

The Flutter UI polls this storage every 500 ms, detects changes via hashCode comparison, and rebuilds the message list without requiring any explicit notification from the native side.

### Deduplication

The app prevents the same message from being forwarded twice through two independent checks:

1. **NativeSmsReceiver level** — Tracks the last 100 processed message IDs (format: `sender-body-timestampMs`). If a duplicate broadcast arrives, it is dropped immediately.
2. **SmsForwardingWorker level** — Before logging and forwarding, checks if a message with the same sender and a timestamp within a 10-second window already exists in the log.

---

## Background & Keep-Alive Strategy

Keeping a service alive on modern Android (with aggressive battery optimization) requires multiple layers:

| Layer | Mechanism |
|-------|-----------|
| Foreground Service | Persistent notification keeps Android from killing the process |
| WakeLock | `PARTIAL_WAKE_LOCK` prevents CPU from entering deep sleep |
| `START_STICKY` | Service auto-restarts if the system kills it |
| `onTaskRemoved` | Restarts service when the user swipes the app from recents |
| `onDestroy` | Restart fallback if any other shutdown path is triggered |
| `BootReceiver` | Starts service on `BOOT_COMPLETED` and `LOCKED_BOOT_COMPLETED` |
| Battery Exemption | `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` prevents Doze throttling |
| `stopWithTask="false"` | Manifest flag ensures service survives app removal from recents |
| `ScreenLockReceiver` | Triggers a keep-alive check whenever the screen is locked |

The native `BackgroundService` runs independently of the Flutter engine. Even if Flutter is not initialized, the Kotlin service continues receiving and forwarding SMS.

---

## Setup & Configuration

1. **Install the APK** on your target Android device.
2. **Grant permissions** — SMS, Phone State, Notifications, and Battery Optimization exemption when prompted.
3. **Choose a forwarding mode** in the setup wizard:
   - **SMS Forward** — Enter the destination phone number.
   - **API Forward** — Enter your HTTP endpoint and (optionally) an online status URL.
   - **Google Sheets** — Enter your published Google Apps Script URL.
4. The app starts the background service automatically. You can close the app — forwarding continues in the background.
5. Optionally configure a **Google Sheets remote config** URL in settings for dynamic endpoint management without reinstalling the app.

---

## Build Requirements

| Requirement | Version |
|------------|---------|
| Flutter SDK | `>=3.7.2` |
| Dart SDK | `>=3.7.2` |
| Android min SDK | 21 (Android 5.0) |
| Android target SDK | 34 (Android 14) |
| Gradle | 8.10.2 |
| Java | 17+ |

```bash
# Install dependencies
flutter pub get

# Build release APK
flutter build apk --release

# Install on connected device
flutter install
```

The release APK will be at `build/app/outputs/flutter-apk/app-release.apk`.

