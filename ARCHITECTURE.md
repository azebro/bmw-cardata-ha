# BMW CarData for Home Assistant — Architecture & Documentation

## Table of Contents

1. [Overview](#1-overview)
2. [System Architecture](#2-system-architecture)
3. [Data Acquisition from BMW](#3-data-acquisition-from-bmw)
4. [Authentication Flow](#4-authentication-flow)
5. [MQTT Streaming — Message Subscriptions](#5-mqtt-streaming--message-subscriptions)
6. [Telematics API Polling](#6-telematics-api-polling)
7. [Data Flow: BMW → Home Assistant](#7-data-flow-bmw--home-assistant)
8. [Entity Platforms & Sensor Types](#8-entity-platforms--sensor-types)
9. [BMW Car Data Descriptors Reference](#9-bmw-car-data-descriptors-reference)
10. [Key Classes & Responsibilities](#10-key-classes--responsibilities)
11. [Configuration Reference](#11-configuration-reference)
12. [Error Handling & Resilience](#12-error-handling--resilience)
13. [Advanced Features](#13-advanced-features)
14. [File-by-File Module Reference](#14-file-by-file-module-reference)
15. [Dependencies](#15-dependencies)

---

## 1. Overview

**BMW CarData for Home Assistant** is a custom integration (~18,000 lines of Python across 47 modules) that connects to BMW's CarData platform to stream real-time vehicle telemetry into native Home Assistant entities.

### Key Capabilities

| Capability | Description |
|---|---|
| **Real-time streaming** | Push-based MQTT telemetry from BMW vehicles |
| **Supplemental polling** | Periodic REST API calls (~24/day) within BMW's 50-call quota |
| **Dynamic entity creation** | Automatically creates sensors for every descriptor that emits data |
| **Multi-vehicle support** | Handles multiple VINs per account with automatic discovery |
| **OAuth 2.0 authentication** | Device Code flow with automatic token refresh |
| **SOC prediction** | Estimates charge completion time during charging sessions |
| **Magic SOC** | Consumption-based remaining range estimation while driving |
| **Custom MQTT broker** | Optional external broker support for advanced deployments |
| **Charging history** | Historical charging session data |
| **Tyre diagnosis** | Tyre health and pressure monitoring |

### Integration Metadata

| Field | Value |
|---|---|
| Domain | `cardata` |
| Version | `5.0.2` |
| IoT Class | `cloud_push` |
| Min HA Version | `2024.6.0` |
| External Dependency | `paho-mqtt >= 1.6.1` |

---

## 2. System Architecture

### High-Level Component Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        HOME ASSISTANT CORE                              │
│  ConfigEntry · Dispatcher · EntityRegistry · StorageAPI · DeviceRegistry│
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                    ┌────────────▼───────────────┐
                    │   ENTRY LIFECYCLE LAYER    │
                    │   lifecycle.py             │
                    │   (setup / unload / reload)│
                    └────────────┬───────────────┘
                                 │
        ┌────────────────────────┼────────────────────────────┐
        │                        │                            │
        ▼                        ▼                            ▼
┌───────────────┐  ┌──────────────────────┐  ┌────────────────────────┐
│  BOOTSTRAP    │  │  TOKEN MANAGEMENT    │  │  TELEMATICS POLLING    │
│  bootstrap.py │  │  auth.py             │  │  telematics.py         │
│               │  │  device_flow.py      │  │                        │
│  · VIN disc.  │  │                      │  │  · Budget-aware polls  │
│  · Metadata   │  │  · OAuth 2.0 DCF     │  │  · Trip-end triggers   │
│  · Container  │  │  · Token refresh     │  │  · Daily diagnostics   │
│  · Images     │  │  · Reauth recovery   │  │  · Charging history    │
└───────┬───────┘  └──────────┬───────────┘  └────────────┬───────────┘
        │                     │                            │
        └─────────┬───────────┘                            │
                  │                                        │
                  ▼                                        │
┌─────────────────────────────────────────────┐            │
│        RUNTIME DATA (per config entry)      │            │
│        runtime.py · CardataRuntimeData      │            │
│                                             │            │
│  · StreamManager (MQTT)                     │            │
│  · Coordinator (state cache)                │            │
│  · ContainerManager (telemetry subs)        │            │
│  · RateLimitTracker                         │            │
│  · UnauthorizedLoopProtection               │            │
│  · ContainerRateLimiter                     │            │
│  · PendingManager                           │            │
│  · Trip-end polling state                   │            │
└───────────┬─────────────────────────────────┘            │
            │                                              │
    ┌───────┴────────┐                                     │
    │                │                                     │
    ▼                ▼                                     │
┌────────────┐  ┌────────────────────┐                     │
│   MQTT     │  │   REST API         │◄────────────────────┘
│  STREAMING │  │   http_retry.py    │
│            │  │                    │
│ stream.py  │  │  · Exponential     │
│ (paho-mqtt)│  │    backoff         │
│            │  │  · Rate-limit      │
│ · BMW or   │  │    awareness       │
│   Custom   │  │  · Retry logic     │
│   broker   │  │  · Header sanitize │
└─────┬──────┘  └────────┬───────────┘
      │                  │
      └────────┬─────────┘
               │
               ▼
┌──────────────────────────────────────────────────────┐
│           STATE COORDINATION & PROCESSING            │
│           coordinator.py · CardataCoordinator        │
│                                                      │
│  · Parse JSON payloads                               │
│  · Extract descriptor-value pairs recursively        │
│  · Update DescriptorState objects                    │
│  · Debounce updates (5-second coalesce window)       │
│  · Dispatch signals to entities                      │
│  · Computed values:                                  │
│    ├─ SOC prediction (soc_prediction.py)             │
│    ├─ Magic SOC (magic_soc.py)                       │
│    ├─ Motion detection (motion_detection.py)         │
│    └─ SOC learning (soc_learning.py)                 │
└──────────────────┬───────────────────────────────────┘
                   │
                   │  HA Dispatcher Signals
                   │  'cardata_update_{vin}'
                   ▼
┌──────────────────────────────────────────────────────┐
│               ENTITY PLATFORMS (6 types)             │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │ sensor.py          — All numeric/string values │  │
│  │ binary_sensor.py   — Boolean states            │  │
│  │ device_tracker.py  — GPS location tracking     │  │
│  │ image.py           — Vehicle photos            │  │
│  │ button.py          — Learning reset actions    │  │
│  │ number.py          — User-configurable values  │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  All inherit from CardataEntity (entity.py)          │
└──────────────────────────────────────────────────────┘

CROSS-CUTTING CONCERNS:
  stream_circuit_breaker.py  — Connection failure protection
  ratelimit.py               — API quota tracking
  pending_manager.py         — Debouncing & update batching
  utils.py                   — VIN validation, redaction, async helpers
  migrations.py              — Entity ID migration for upgrades
  frontend_cards.py          — Lovelace websocket API & card setup
  services.py                — Developer service handlers
  options_flow.py            — Settings UI
```

---

## 3. Data Acquisition from BMW

The integration acquires data through two independent channels: **MQTT streaming** (primary, push-based) and **REST API polling** (secondary, pull-based). Both require OAuth 2.0 tokens.

### 3.1 BMW APIs and Endpoints

| Service | Method | Endpoint | Purpose | Quota Impact |
|---|---|---|---|---|
| **CarData Streaming** | MQTT | `customer.streaming-cardata.bmwgroup.com:9000` | Real-time vehicle telemetry push | Unlimited |
| **Telematics API** | `GET` | `/api/v1/customers/vehicles/{vin}/tele` | Current vehicle state snapshot | 50 calls/24h total |
| **Basic Data API** | `GET` | `/api/v1/customers/vehicles/{vin}/basicData` | Vehicle metadata (model, series) | Included in quota |
| **Vehicle Mappings** | `GET` | `/api/v1/customers/vehicles/mappings` | List user's registered vehicles | Included in quota |
| **Charging History** | `GET` | `/api/v1/customers/vehicles/{vin}/chargingHistory` | Historical charging sessions | ~1 call/24h |
| **Tyre Diagnosis** | `GET` | `/api/v1/customers/vehicles/{vin}/tyreDiagnosis` | Tyre health and pressure data | ~1 call/24h |

### 3.2 Key URL Constants

```
DEVICE_CODE_URL    = https://customer.bmwgroup.com/gcdm/oauth/device/code
TOKEN_URL          = https://customer.bmwgroup.com/gcdm/oauth/token
API_BASE_URL       = https://api-cardata.bmwgroup.com
API_VERSION        = v1
DEFAULT_STREAM_HOST = customer.streaming-cardata.bmwgroup.com
DEFAULT_STREAM_PORT = 9000
```

### 3.3 OAuth Scopes

```
DEFAULT_SCOPE = "authenticate_user openid cardata:api:read cardata:streaming:read"
```

| Scope | Purpose |
|---|---|
| `authenticate_user` | User authentication |
| `openid` | OpenID Connect identity |
| `cardata:api:read` | REST API access (telematics, basic data, etc.) |
| `cardata:streaming:read` | MQTT stream access |

---

## 4. Authentication Flow

The integration uses **OAuth 2.0 Device Authorization Grant** (Device Code Flow) with **PKCE** (Proof Key for Code Exchange, S256 method).

### 4.1 Initial Authentication

```
                    User                  Home Assistant              BMW OAuth Server
                     │                         │                           │
                     │  1. Enter Client ID      │                           │
                     │ ─────────────────────►   │                           │
                     │                         │  2. POST /device/code      │
                     │                         │  (client_id, PKCE challenge│
                     │                         │   scope, response_type)    │
                     │                         │ ─────────────────────────► │
                     │                         │                           │
                     │                         │  3. device_code + user_code│
                     │                         │ ◄───────────────────────── │
                     │  4. Show verification_url│                           │
                     │     + user_code          │                           │
                     │ ◄─────────────────────   │                           │
                     │                         │                           │
                     │  5. Visit URL, enter     │                           │
                     │     code, approve        │                           │
                     │ ───────────────────────────────────────────────────► │
                     │                         │                           │
                     │                         │  6. Poll POST /token       │
                     │                         │  (device_code, verifier)   │
                     │                         │ ─────────────────────────► │
                     │                         │                           │
                     │                         │  7. access_token,          │
                     │                         │     refresh_token,         │
                     │                         │     id_token               │
                     │                         │ ◄───────────────────────── │
                     │                         │                           │
                     │  8. Integration ready    │                           │
                     │ ◄─────────────────────   │                           │
```

### 4.2 PKCE Details

1. **Code Verifier**: An 86-character random string from `[a-zA-Z0-9-._~]`
2. **Code Challenge**: Base64url-encoded SHA-256 hash of the verifier (S256 method)
3. **Client ID**: Must be a valid UUID format (8-4-4-4-12 hexadecimal), created in the BMW portal

### 4.3 Token Storage

Tokens are stored in the HA config entry's `data` dictionary:

| Key | Description |
|---|---|
| `access_token` | Bearer token for API requests |
| `refresh_token` | Long-lived token for obtaining new access tokens |
| `id_token` | Identity token used as MQTT password |
| `expires_in` | Token lifetime in seconds |
| `received_at` | Unix timestamp when tokens were received |
| `gcid` | Global Client ID (extracted from token, used as MQTT username) |

### 4.4 Automatic Token Refresh

- **Refresh interval**: Every 45 minutes (`DEFAULT_REFRESH_INTERVAL`)
- **Expiry buffer**: Refresh 5 minutes before expiry (`TOKEN_EXPIRY_BUFFER_SECONDS = 300`)
- **Concurrency protection**: `token_refresh_lock` prevents simultaneous refresh attempts
- **Process**: `auth.py` → `async_token_refresh_loop()` runs continuously, calling `refresh_tokens()` from `device_flow.py`
- **On refresh**: Updates entry data, MQTT stream credentials, and validates container

### 4.5 Reauthorization

Reauth is triggered when:
- MQTT disconnection with return code 4 or 5 (authentication failure)
- Token refresh returns HTTP 401/403
- Manual reauth from options flow

The reauth process:
1. Detects unauthorized state
2. Creates persistent notification with new verification URL
3. Starts new device code flow (SOURCE_REAUTH)
4. User re-approves on BMW portal
5. New tokens stored, MQTT reconnects

---

## 5. MQTT Streaming — Message Subscriptions

### 5.1 Connection Configuration

| Parameter | BMW Broker | Custom Broker |
|---|---|---|
| **Host** | `customer.streaming-cardata.bmwgroup.com` | User-configurable |
| **Port** | `9000` | User-configurable (default: `1883`) |
| **TLS** | Required (system CA) | Configurable (`off` / `tls` / `tls_insecure`) |
| **Username** | GCID (Global Client ID) | Optional |
| **Password** | `id_token` (JWT) | Optional |
| **Client ID** | `cardata-ha-{entry_id}` | `cardata-ha-{entry_id}` |
| **Keep-alive** | 30 seconds (configurable) | 30 seconds (configurable) |
| **Clean session** | `True` | `True` |
| **QoS** | 1 (at least once) | 1 (at least once) |

### 5.2 Topic Subscription Patterns

The integration subscribes to a **single wildcard topic** upon successful MQTT connection. The exact pattern depends on the broker type:

#### BMW Official Broker

```
Topic: {GCID}/+
```

Where `GCID` is the Global Client ID extracted from the OAuth token. The `+` is a single-level MQTT wildcard that matches any sub-topic under the GCID namespace.

**Example**: If your GCID is `abc123def456`, the subscription is:
```
abc123def456/+
```

This receives all messages published by BMW for your account, including telemetry from all registered vehicles.

#### Custom MQTT Broker

```
Topic: {prefix}+
```

Where `prefix` defaults to `bmw/` (configurable via `OPTION_CUSTOM_MQTT_TOPIC_PREFIX`).

**Example** (default prefix):
```
bmw/+
```

### 5.3 Subscription Flow in Code

The subscription happens in the `on_connect` callback inside `stream.py`:

1. `CardataStreamManager._connect_mqtt()` creates a Paho MQTT client
2. The **topic** is computed before connection and passed via `userdata`
3. On successful connection (`rc == 0`), the `_handle_connect` callback fires
4. Inside `_handle_connect`, `client.subscribe(topic)` is called with QoS 1
5. `_handle_subscribe` logs the subscription confirmation

```
_connect_mqtt()
  │
  ├── Compute topic:
  │   ├── Custom broker: topic = f"{prefix}+"     (e.g., "bmw/+")
  │   └── BMW broker:    topic = f"{self._gcid}/+" (e.g., "abc123/+")
  │
  ├── Set callbacks: on_connect, on_message, on_disconnect, on_subscribe
  │
  ├── Configure TLS (if BMW broker or custom with TLS enabled)
  │
  ├── Set credentials (GCID/id_token or custom username/password)
  │
  ├── client.connect(host, port, keepalive)
  │
  └── client.loop_start()  ──► Paho network thread
                                    │
                                    ▼
                            _handle_connect(rc=0)
                                    │
                                    ├── client.subscribe(topic, qos=1)
                                    │
                                    └── Signal connection_event
```

### 5.4 Message Format

BMW publishes JSON payloads on the subscribed topic. Each message contains nested vehicle telemetry data:

```json
{
  "vehicle": {
    "drivetrain": {
      "batteryManagement": {
        "header": 72
      },
      "electricEngine": {
        "charging": {
          "status": "CHARGINGACTIVE",
          "level": 73,
          "acVoltage": 230,
          "acAmpere": 16,
          "phaseNumber": 1
        }
      }
    },
    "cabin": {
      "infotainment": {
        "navigation": {
          "currentLocation": {
            "latitude": "48.123456",
            "longitude": "11.654321",
            "heading": "180",
            "altitude": "530"
          }
        }
      }
    }
  },
  "timestamp": "2025-03-16T10:30:00.000Z"
}
```

Individual telemetry values also come in the flat descriptor format:

```json
{
  "name": "vehicle.drivetrain.batteryManagement.header",
  "timestamp": "2025-03-16T10:30:00.000Z",
  "unit": "%",
  "value": "72"
}
```

### 5.5 Message Processing Pipeline

```
MQTT Message Received
  │
  ▼
_handle_message(client, userdata, msg)
  │
  ├── Decode payload (UTF-8 JSON)
  │
  ├── Route to CardataCoordinator.process_message()
  │
  ▼
CardataCoordinator.process_message()
  │
  ├── Parse JSON → extract descriptor-value pairs recursively
  │   (nested objects flattened to dot-notation descriptors)
  │
  ├── For each descriptor:
  │   ├── Create/update DescriptorState(value, unit, timestamp, last_seen)
  │   ├── Classify as sensor or binary_sensor
  │   └── Queue in UpdateBatcher
  │
  ├── Run debouncer (5-second coalesce window)
  │   └── Prevents rapid-fire entity updates from burst messages
  │
  ├── Calculate computed values:
  │   ├── SOC Predictor → update predicted_soc if charging
  │   ├── Magic SOC → update consumption prediction if driving
  │   ├── Motion Detector → update isMoving from GPS/mileage
  │   └── SOC Learning → accumulate charging efficiency data
  │
  └── Dispatch updates:
      └── async_dispatcher_send(hass, 'cardata_update_{vin}', vin, descriptor)
          │
          ▼
      Entity._handle_update(vin, descriptor) → update HA state
```

### 5.6 Connection Lifecycle

```
                    ┌─────────┐
                    │  INIT   │
                    └────┬────┘
                         │ async_start()
                         ▼
                    ┌──────────┐
              ┌────►│CONNECTING│
              │     └────┬─────┘
              │          │ on_connect(rc=0)
              │          ▼
              │     ┌──────────┐
              │     │CONNECTED │◄──────────────────┐
              │     └────┬─────┘                   │
              │          │ on_disconnect            │
              │          ▼                          │
              │     ┌──────────────┐               │
              │     │DISCONNECTED  │───────────────┘
              │     └────┬─────────┘  Retry after backoff
              │          │
              │          │ Auth failure (rc=4/5)
              │          ▼
              │     ┌──────────┐
              │     │  FAILED  │
              │     └────┬─────┘
              │          │ After reauth + token refresh
              └──────────┘
```

### 5.7 Global Connection Lock

A **global asyncio lock** serializes MQTT connections across all config entries. This prevents multiple entries from simultaneously connecting and overwhelming BMW's MQTT broker with concurrent handshakes.

---

## 6. Telematics API Polling

### 6.1 Polling Budget

| Parameter | Value |
|---|---|
| **Target daily polls** | 24 (`TARGET_DAILY_POLLS`) |
| **BMW daily quota** | 50 API calls/24h |
| **Remaining budget** | Reserved for bootstrap, trip-end events, manual calls |
| **Dynamic interval** | ~30–60 minutes depending on features enabled |

When optional daily features are enabled (charging history, tyre diagnosis), the polling budget is automatically reduced to keep total calls within quota.

### 6.2 Polling Triggers

| Trigger | Description |
|---|---|
| **Periodic timer** | Calculated interval based on daily budget |
| **Trip-end event** | Vehicle motion stops → immediate poll for final state |
| **Bootstrap** | Initial data seeding on startup |
| **Token refresh** | Opportunistic fetch after token renewal |
| **Manual service call** | `cardata.fetch_telematic_data` from Developer Tools |

### 6.3 Smart Polling Behavior

- **Skip fresh VINs**: If a VIN received MQTT data within the last 5 minutes, the poll is skipped
- **Auth failure tracking**: After 3 consecutive auth failures, polling pauses until reauth completes
- **Rate limit pre-flight**: Checks `RateLimitTracker` before making any API call

### 6.4 Daily Fetch Tasks (Midnight)

| Endpoint | Condition |
|---|---|
| Charging History | If `OPTION_ENABLE_CHARGING_HISTORY` is enabled |
| Tyre Diagnosis | If `OPTION_ENABLE_TYRE_DIAGNOSIS` is enabled |

---

## 7. Data Flow: BMW → Home Assistant

### Complete End-to-End Path

```
┌──────────────────────────────────────────────────────────────────────────┐
│ 1. AUTHENTICATION SETUP                                                  │
│    User enters Client ID → Device Code Flow → Tokens stored in HA       │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼─────────────────────────────────────────┐
│ 2. BOOTSTRAP SEQUENCE (async_run_bootstrap)                              │
│    ├── Token refresh + validation                                        │
│    ├── Fetch vehicle mappings → VIN discovery                            │
│    ├── VIN deduplication across config entries (global lock)             │
│    ├── Fetch basic data → vehicle metadata (model, series, brand)        │
│    ├── Create/validate HV battery telemetry container                    │
│    ├── Fetch vehicle images                                              │
│    ├── Seed telematic data (initial state population)                    │
│    └── Signal bootstrap complete → MQTT stream can start                 │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │
         ┌───────────────────────┼────────────────────────────┐
         │                       │                            │
┌────────▼───────────┐  ┌───────▼──────────────┐  ┌──────────▼──────────┐
│ 3. MQTT STREAMING  │  │ 4. API POLLING       │  │ 5. DAILY FETCHES    │
│ (PRIMARY)          │  │ (SECONDARY)          │  │                     │
│                    │  │                      │  │ Charging history    │
│ BMW car publishes  │  │ ~24 polls/day        │  │ Tyre diagnosis      │
│ JSON → {GCID}/+   │  │ GET /vehicles/{vin}  │  │ Vehicle images      │
│ Real-time push     │  │ /tele                │  │ (once per 24h)      │
└────────┬───────────┘  └──────────┬───────────┘  └──────────┬──────────┘
         │                         │                          │
         └───────────┬─────────────┘                          │
                     │                                        │
         ┌───────────▼────────────────────────────────────────┘
         │
┌────────▼─────────────────────────────────────────────────────────────────┐
│ 6. COORDINATOR PROCESSING (CardataCoordinator)                           │
│    ├── Parse JSON payload                                                │
│    ├── Extract descriptor-value pairs recursively                        │
│    ├── Update DescriptorState objects (value, unit, timestamp)           │
│    ├── Run debouncer (5-second coalesce window)                          │
│    ├── Calculate computed values:                                        │
│    │   ├── Predicted SOC (during charging)                               │
│    │   ├── Magic SOC (during driving)                                    │
│    │   ├── Motion detection (GPS-based)                                  │
│    │   └── SOC learning (charging efficiency)                            │
│    └── Dispatch via HA signal: 'cardata_update_{vin}'                    │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼─────────────────────────────────────────┐
│ 7. ENTITY UPDATE                                                         │
│    Each entity subscribes to dispatcher signal for its VIN               │
│    ├── sensor.py           → update numeric/string state                 │
│    ├── binary_sensor.py    → update boolean state                        │
│    ├── device_tracker.py   → update GPS coordinates                      │
│    ├── image.py            → load vehicle photo bytes                    │
│    ├── button.py           → enable/disable reset buttons                │
│    └── number.py           → reflect capacity overrides                  │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼─────────────────────────────────────────┐
│ 8. HOME ASSISTANT UI                                                     │
│    Dashboard cards · History database · Automations · Lovelace           │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Entity Platforms & Sensor Types

### 8.1 Platform Summary

| Platform | File | Entity Count | Description |
|---|---|---|---|
| **Sensor** | `sensor.py` | 60+ per vehicle | All numeric/string telemetry values |
| **Binary Sensor** | `binary_sensor.py` | 5–10 per vehicle | Boolean states (doors, windows, motion) |
| **Device Tracker** | `device_tracker.py` | 1 per vehicle | GPS location with heading/altitude |
| **Image** | `image.py` | 1 per vehicle | Vehicle exterior photo |
| **Button** | `button.py` | 2–3 per EV | Learning reset actions |
| **Number** | `number.py` | 1 per EV | Manual battery capacity override |

### 8.2 Sensors (`sensor.py`)

The main sensor platform creates entities for every non-boolean descriptor received from BMW. Sensors inherit from `CardataEntity`, `RestoreEntity`, and `SensorEntity`.

**Key behaviors:**
- Smart update filtering: only updates HA state when value/unit actually changes or sensor was in `unknown` state
- Automatic state class detection: `TOTAL_INCREASING` for mileage, `MEASUREMENT` for power/temperature/pressure
- Hidden by default: raw GPS latitude/longitude sensors (device tracker is preferred)
- State restoration: last known value restored on HA restart

**Diagnostic Sensors (created per integration entry):**

| Sensor | Description |
|---|---|
| `connection_status` | Current MQTT connection state |
| `last_message` | Timestamp of last MQTT message received |
| `last_telematic_api` | Timestamp of last successful API poll |

**Per-Vehicle Diagnostic Sensors:**

| Sensor | Description |
|---|---|
| `CardataVehicleMetadataSensor` | Vehicle model, brand, series, construction date |
| `CardataEfficiencyLearningSensor` | AC/DC charging efficiency learning data |
| `CardataChargingHistorySensor` | Historical charging session data |
| `CardataTyreDiagnosisSensor` | Tyre pressure and health diagnostics |

**Computed Sensors:**

| Sensor | Descriptor | Description |
|---|---|---|
| Predicted SOC | `vehicle.predicted_soc` | Estimated charge % and completion time during charging |
| Magic SOC | `vehicle.magic_soc` | Consumption-based remaining range while driving |

### 8.3 Binary Sensors (`binary_sensor.py`)

Binary sensors represent boolean on/off states with device class mapping:

| Descriptor Pattern | Device Class | Icon |
|---|---|---|
| `vehicle.cabin.door.row1.driver.isOpen` | `DOOR` | `mdi:car-door` |
| `vehicle.cabin.door.row1.passenger.isOpen` | `DOOR` | `mdi:car-door` |
| `vehicle.cabin.door.row2.driver.isOpen` | `DOOR` | `mdi:car-door` |
| `vehicle.cabin.door.row2.passenger.isOpen` | `DOOR` | `mdi:car-door` |
| `vehicle.body.trunk.isOpen` | `DOOR` | `mdi:circle` / `mdi:circle-outline` |
| `vehicle.body.hood.isOpen` | `DOOR` | `mdi:circle` / `mdi:circle-outline` |
| `vehicle.body.trunk.door.isOpen` | `DOOR` | `mdi:circle` / `mdi:circle-outline` |
| `vehicle.body.trunk.left.door.isOpen` | `DOOR` | `mdi:circle` / `mdi:circle-outline` |
| `vehicle.body.trunk.right.door.isOpen` | `DOOR` | `mdi:circle` / `mdi:circle-outline` |
| `vehicle.isMoving` | `MOVING` | `mdi:car-arrow-right` / `mdi:car-brake-parking` |

**Special `vehicle.isMoving` handling:**
- Never restored on HA restart (motion detector loses GPS history)
- Always starts as `False` (safe default)
- Signals motion detector entity creation

### 8.4 Device Tracker (`device_tracker.py`)

GPS-based vehicle location tracking using four coordinate descriptors:

| Descriptor | Purpose |
|---|---|
| `vehicle.cabin.infotainment.navigation.currentLocation.latitude` | Latitude |
| `vehicle.cabin.infotainment.navigation.currentLocation.longitude` | Longitude |
| `vehicle.cabin.infotainment.navigation.currentLocation.heading` | Compass heading |
| `vehicle.cabin.infotainment.navigation.currentLocation.altitude` | Altitude |

**Coordinate Pairing Logic:**
- BMW sends latitude and longitude as separate messages
- Primary pairing: BMW payload timestamps within 5 seconds
- Fallback: Message arrival times within 30 seconds
- Staleness check: Discard coordinates older than 10 minutes
- Movement threshold: Only update on movement > 3 meters

### 8.5 Image (`image.py`)

Vehicle exterior photo fetched from BMW metadata API:
- Auto-fetched on first load if missing
- Cached to disk as PNG
- Refreshed daily
- Returns `None` if file is 0 bytes (marker for no image available)

### 8.6 Button (`button.py`)

Learning reset buttons for EV/PHEV vehicles (only created for vehicles with `DESC_SOC_HEADER`):

| Button | Action |
|---|---|
| Reset AC Learning | Clears AC charging efficiency learned data |
| Reset DC Learning | Clears DC charging efficiency learned data |
| Reset Consumption Learning | Clears Magic SOC consumption data |

### 8.7 Number (`number.py`)

| Entity | Range | Unit | Default |
|---|---|---|---|
| Manual Battery Capacity | 0–150 kWh (step 0.1) | kWh | Disabled by default |

Setting to 0 clears the override and enables auto-detection.

### 8.8 Entity Naming Convention

- Each VIN creates a **device** in HA
- **Unique ID**: `{VIN}_{descriptor}` (e.g., `WBAPH123456789_vehicle.drivetrain.batteryManagement.header`)
- **Name**: Vehicle name prefix + human-readable descriptor name
- Descriptor titles are mapped through `DESCRIPTOR_TITLES` dictionary, with fallback to tokenized descriptor path
- Vehicle name prefix is deduplicated to avoid repetition

---

## 9. BMW Car Data Descriptors Reference

### 9.1 Battery & Charging Descriptors

| Descriptor | Constant | Description |
|---|---|---|
| `vehicle.drivetrain.batteryManagement.header` | `DESC_SOC_HEADER` | State of Charge (%) |
| `vehicle.drivetrain.batteryManagement.maxEnergy` | `DESC_MAX_ENERGY` | Maximum battery energy |
| `vehicle.drivetrain.batteryManagement.batterySizeMax` | `DESC_BATTERY_SIZE_MAX` | Maximum battery size |
| `vehicle.drivetrain.electricEngine.charging.acVoltage` | `DESC_CHARGING_AC_VOLTAGE` | AC charging voltage |
| `vehicle.drivetrain.electricEngine.charging.acAmpere` | `DESC_CHARGING_AC_AMPERE` | AC charging current |
| `vehicle.drivetrain.electricEngine.charging.phaseNumber` | `DESC_CHARGING_PHASES` | Number of AC phases |
| `vehicle.drivetrain.electricEngine.charging.status` | `DESC_CHARGING_STATUS` | Charging status string |
| `vehicle.drivetrain.electricEngine.charging.level` | `DESC_CHARGING_LEVEL` | Charging level (%) |
| `vehicle.powertrain.electric.battery.charging.power` | `DESC_CHARGING_POWER` | Battery charging power |
| `vehicle.drivetrain.electricEngine.charging.timeToFullyCharged` | — | Time to full charge |
| `vehicle.drivetrain.electricEngine.charging.timeRemaining` | — | Charging time remaining |
| `vehicle.drivetrain.electricEngine.charging.method` | — | Charging method (AC/DC) |
| `vehicle.drivetrain.electricEngine.charging.hvStatus` | — | HV charging status |
| `vehicle.drivetrain.electricEngine.charging.lastChargingReason` | — | Last charging trigger reason |
| `vehicle.drivetrain.electricEngine.charging.lastChargingResult` | — | Last charging result |
| `vehicle.drivetrain.electricEngine.charging.reasonChargingEnd` | — | Why charging ended |
| `vehicle.powertrain.electric.battery.stateOfCharge.target` | — | Target SOC (%) |
| `vehicle.powertrain.electric.battery.stateOfHealth.displayed` | — | Battery health (SOH %) |

### 9.2 HV Battery Container Descriptors

These are the descriptors subscribed to in the **HV Battery telematics container** (created during bootstrap):

```python
HV_BATTERY_DESCRIPTORS = [
    "vehicle.drivetrain.batteryManagement.header",
    "vehicle.drivetrain.electricEngine.charging.acAmpere",
    "vehicle.drivetrain.electricEngine.charging.acVoltage",
    "vehicle.powertrain.electric.battery.preconditioning.automaticMode.statusFeedback",
    "vehicle.vehicle.avgAuxPower",
    "vehicle.powertrain.tractionBattery.charging.port.anyPosition.flap.isOpen",
    "vehicle.powertrain.tractionBattery.charging.port.anyPosition.isPlugged",
    "vehicle.drivetrain.electricEngine.charging.timeToFullyCharged",
    "vehicle.powertrain.electric.battery.charging.acLimit.selected",
    "vehicle.drivetrain.electricEngine.charging.method",
    "vehicle.body.chargingPort.plugEventId",
    "vehicle.drivetrain.electricEngine.charging.phaseNumber",
    "vehicle.trip.segment.end.drivetrain.batteryManagement.hvSoc",
    "vehicle.trip.segment.accumulated.drivetrain.electricEngine.recuperationTotal",
    "vehicle.drivetrain.electricEngine.remainingElectricRange",
    "vehicle.drivetrain.electricEngine.charging.timeRemaining",
    "vehicle.drivetrain.electricEngine.charging.hvStatus",
    "vehicle.drivetrain.electricEngine.charging.lastChargingReason",
    "vehicle.drivetrain.electricEngine.charging.lastChargingResult",
    "vehicle.powertrain.electric.battery.preconditioning.manualMode.statusFeedback",
    "vehicle.drivetrain.electricEngine.charging.reasonChargingEnd",
    "vehicle.powertrain.electric.battery.stateOfCharge.target",
    "vehicle.body.chargingPort.lockedStatus",
    "vehicle.drivetrain.electricEngine.charging.level",
    "vehicle.powertrain.electric.battery.stateOfHealth.displayed",
    "vehicle.vehicleIdentification.basicVehicleData",
    "vehicle.drivetrain.batteryManagement.batterySizeMax",
    "vehicle.drivetrain.batteryManagement.maxEnergy",
    "vehicle.powertrain.electric.battery.charging.power",
    "vehicle.drivetrain.electricEngine.charging.status",
]
```

### 9.3 Location Descriptors

| Descriptor | Constant | Description |
|---|---|---|
| `vehicle.cabin.infotainment.navigation.currentLocation.latitude` | `LOCATION_LATITUDE_DESCRIPTOR` | GPS latitude |
| `vehicle.cabin.infotainment.navigation.currentLocation.longitude` | `LOCATION_LONGITUDE_DESCRIPTOR` | GPS longitude |
| `vehicle.cabin.infotainment.navigation.currentLocation.heading` | `LOCATION_HEADING_DESCRIPTOR` | Compass heading |
| `vehicle.cabin.infotainment.navigation.currentLocation.altitude` | `LOCATION_ALTITUDE_DESCRIPTOR` | Altitude |

### 9.4 Door & Window Descriptors

| Descriptor | Type |
|---|---|
| `vehicle.cabin.door.row1.driver.isOpen` | Binary (door) |
| `vehicle.cabin.door.row1.passenger.isOpen` | Binary (door) |
| `vehicle.cabin.door.row2.driver.isOpen` | Binary (door) |
| `vehicle.cabin.door.row2.passenger.isOpen` | Binary (door) |
| `vehicle.body.trunk.isOpen` | Binary (trunk) |
| `vehicle.body.hood.isOpen` | Binary (hood) |
| `vehicle.body.trunk.door.isOpen` | Binary (trunk door) |
| `vehicle.body.trunk.left.door.isOpen` | Binary (trunk left) |
| `vehicle.body.trunk.lower.door.isOpen` | Binary (trunk lower) |
| `vehicle.body.trunk.right.door.isOpen` | Binary (trunk right) |
| `vehicle.body.trunk.upper.door.isOpen` | Binary (trunk upper) |
| `vehicle.cabin.window.row1.driver.status` | Sensor (window) |
| `vehicle.cabin.window.row1.passenger.status` | Sensor (window) |
| `vehicle.cabin.window.row2.driver.status` | Sensor (window) |
| `vehicle.cabin.window.row2.passenger.status` | Sensor (window) |
| `vehicle.body.trunk.window.isOpen` | Sensor (trunk window) |

### 9.5 Fuel & Range Descriptors

| Descriptor | Constant | Description |
|---|---|---|
| `vehicle.drivetrain.fuelSystem.remainingFuel` | `DESC_REMAINING_FUEL` | Remaining fuel (liters) |
| `vehicle.drivetrain.fuelSystem.level` | `DESC_FUEL_LEVEL` | Fuel level (%) |
| `vehicle.drivetrain.electricEngine.remainingElectricRange` | — | Electric range (km) |

### 9.6 Vehicle Status Descriptors

| Descriptor | Constant | Description |
|---|---|---|
| `vehicle.vehicle.travelledDistance` | `DESC_TRAVELLED_DISTANCE` | Odometer / trip meter |
| `vehicle.vehicle.avgAuxPower` | — | Average auxiliary power |
| `vehicle.isMoving` | — | Vehicle motion state |
| `vehicle.cabin.door.status` | — | Door lock state |

### 9.7 Computed Descriptors (Integration-Generated)

| Descriptor | Constant | Source |
|---|---|---|
| `vehicle.predicted_soc` | `PREDICTED_SOC_DESCRIPTOR` | SOC predictor (charging) |
| `vehicle.magic_soc` | `MAGIC_SOC_DESCRIPTOR` | Magic SOC predictor (driving) |
| `vehicle.manual_battery_capacity` | `MANUAL_CAPACITY_DESCRIPTOR` | User input (number entity) |

---

## 10. Key Classes & Responsibilities

| Class | File | Responsibility |
|---|---|---|
| `CardataStreamManager` | `stream.py` | MQTT connection lifecycle, reconnection with exponential backoff, circuit breaker integration |
| `CardataCoordinator` | `coordinator.py` | Central state store per entry; processes MQTT & API messages; manages descriptors; dispatches updates to entities |
| `CardataRuntimeData` | `runtime.py` | Per-entry runtime container holding stream manager, coordinator, rate limiters, session, locks, and task handles |
| `CardataEntity` | `entity.py` | Base class for all entities; handles naming, device info, VIN resolution |
| `CardataSensor` | `sensor.py` | Generic sensor entity for numeric/string descriptors |
| `CardataBinarySensor` | `binary_sensor.py` | Boolean state entity with device class mapping |
| `CardataDeviceTracker` | `device_tracker.py` | GPS location entity with coordinate pairing and movement threshold |
| `CardataImage` | `image.py` | Vehicle photo entity |
| `ResetLearningButton` | `button.py` | Parametrized reset button for efficiency learning |
| `ManualBatteryCapacityNumber` | `number.py` | User-configurable battery capacity override |
| `CardataContainerManager` | `container.py` | HV battery container CRUD; descriptor signature validation |
| `SOCPredictor` | `soc_prediction.py` | Charging curve interpolation for time-to-full; session tracking |
| `MagicSOCPredictor` | `magic_soc.py` | Consumption learning for driving range prediction |
| `MotionDetector` | `motion_detection.py` | GPS-based motion detection with parking zone, door lock, and mileage fallbacks |
| `CircuitBreaker` | `stream_circuit_breaker.py` | Failure tracking; connection lockout after threshold |
| `RateLimitTracker` | `ratelimit.py` | HTTP 429 handling with exponential backoff |
| `UnauthorizedLoopProtection` | `ratelimit.py` | Prevents 401/403 retry storms |
| `ContainerRateLimiter` | `ratelimit.py` | Limits container creation operations (3/hour, 10/day) |
| `UpdateBatcher` | `pending_manager.py` | Descriptor state batching with memory protection |
| `PendingManager` | `pending_manager.py` | Generic deduplication lock for concurrent operations |
| `DescriptorState` | `descriptor_state.py` | Per-descriptor state dataclass (value, unit, timestamp, last_seen) |
| `HttpResponse` | `http_retry.py` | HTTP response wrapper with status helpers |

---

## 11. Configuration Reference

### 11.1 Config Entry Data (Stored in HA)

| Key | Type | Description |
|---|---|---|
| `client_id` | `str` | BMW OAuth Client ID (UUID format) |
| `access_token` | `str` | Bearer token for API requests |
| `refresh_token` | `str` | Long-lived refresh token |
| `id_token` | `str` | JWT identity token (MQTT password) |
| `expires_in` | `int` | Token lifetime in seconds |
| `received_at` | `float` | Unix timestamp of token receipt |
| `gcid` | `str` | Global Client ID (MQTT username) |
| `allowed_vins` | `list[str]` | VINs claimed by this config entry |
| `bootstrap_complete` | `bool` | Whether bootstrap finished |
| `hv_container_id` | `str` | HV battery container ID |
| `hv_descriptor_signature` | `str` | Hash of container descriptors |
| `vehicle_metadata` | `dict` | Cached vehicle metadata per VIN |
| `last_telematic_poll` | `float` | Timestamp of last API poll |

### 11.2 Config Entry Options (User-Configurable)

| Option | Key | Type | Default | Description |
|---|---|---|---|---|
| MQTT Keep-alive | `mqtt_keepalive` | `int` | `30` | MQTT keep-alive interval (seconds) |
| Debug Logging | `debug_log` | `bool` | `False` | Enable verbose MQTT/auth logs |
| Custom MQTT Enabled | `custom_mqtt_enabled` | `bool` | `False` | Use external MQTT broker |
| Custom MQTT Host | `custom_mqtt_host` | `str` | — | Custom broker hostname |
| Custom MQTT Port | `custom_mqtt_port` | `int` | `1883` | Custom broker port |
| Custom MQTT Username | `custom_mqtt_username` | `str` | — | Custom broker username |
| Custom MQTT Password | `custom_mqtt_password` | `str` | — | Custom broker password |
| Custom MQTT TLS | `custom_mqtt_tls` | `str` | `off` | TLS mode: `off`, `tls`, `tls_insecure` |
| Custom MQTT Topic | `custom_mqtt_topic_prefix` | `str` | `bmw/` | Topic prefix for custom broker |
| Diagnostic Interval | `diagnostic_log_interval` | `int` | `30` | Log interval for stream diagnostics (seconds) |
| Magic SOC | `enable_magic_soc` | `bool` | `False` | Enable consumption-based range prediction |
| Charging History | `enable_charging_history` | `bool` | `False` | Fetch daily charging history |
| Tyre Diagnosis | `enable_tyre_diagnosis` | `bool` | `False` | Fetch daily tyre diagnosis |
| Trip-End Polling | `enable_trip_end_polling` | `bool` | `True` | Poll API when trip ends |
| Trip Poll Cooldown | `trip_poll_cooldown_minutes` | `int` | `10` | Minimum wait between trip-end polls |

### 11.3 Integration Constants

| Constant | Value | Description |
|---|---|---|
| `DOMAIN` | `"cardata"` | Integration domain |
| `DEFAULT_REFRESH_INTERVAL` | `2700` (45 min) | Token refresh interval |
| `MQTT_KEEPALIVE` | `30` | Default MQTT keep-alive (seconds) |
| `TARGET_DAILY_POLLS` | `24` | Target API polls per day |
| `HTTP_TIMEOUT` | `30` | HTTP request timeout (seconds) |
| `LOCK_ACQUIRE_TIMEOUT` | `60` | Async lock timeout (seconds) |
| `MIN_TELEMETRY_DESCRIPTORS` | `5` | Min descriptors to classify as real vehicle |
| `DAILY_FETCH_INTERVAL` | `86400` | Daily fetch interval (seconds) |
| `DEBOUNCE_SECONDS` | `5.0` | Message coalesce window |
| `DEFAULT_TRIP_POLL_COOLDOWN_MINUTES` | `10` | Trip-end poll cooldown |

### 11.4 SOC Learning Constants

| Constant | Value | Description |
|---|---|---|
| `DEFAULT_DC_EFFICIENCY` | `0.93` | Default DC charging efficiency |
| `LEARNING_RATE` | `0.2` | EMA learning rate (20% new, 80% old) |
| `MIN_LEARNING_SOC_GAIN` | `5.0` | Minimum SOC gain to learn (%) |
| `MIN_VALID_EFFICIENCY` | `0.40` | Lower efficiency bound |
| `MAX_VALID_EFFICIENCY` | `0.98` | Upper efficiency bound |
| `TARGET_SOC_TOLERANCE` | `2.0` | SOC target match tolerance (%) |
| `DC_SESSION_FINALIZE_MINUTES` | `5.0` | DC session finalize grace period |
| `AC_SESSION_FINALIZE_MINUTES` | `15.0` | AC session finalize grace period |
| `MAX_ENERGY_GAP_SECONDS` | `600` | Max gap before skipping energy integration |

### 11.5 Driving Consumption Constants

| Constant | Value | Description |
|---|---|---|
| `DEFAULT_CONSUMPTION_KWH_PER_KM` | `0.21` | BMW BEV fleet average |
| `MIN_VALID_CONSUMPTION` | `0.10` | Lower consumption bound |
| `MAX_VALID_CONSUMPTION` | `0.40` | Upper consumption bound |
| `MIN_LEARNING_TRIP_DISTANCE_KM` | `5.0` | Minimum trip distance to learn |
| `MIN_LEARNING_SOC_DROP` | `2.0` | Minimum SOC drop to learn (%) |
| `DRIVING_SOC_CONTINUITY_SECONDS` | `300` | isMoving flap tolerance (5 min) |
| `DRIVING_SESSION_MAX_AGE_SECONDS` | `14400` | Session expiry (4 hours) |
| `GPS_MAX_STEP_DISTANCE_M` | `2000` | Max single GPS step (reject jumps) |
| `REFERENCE_LEARNING_TRIP_KM` | `30.0` | Reference distance for learning weight |

### 11.6 Model-Based Defaults

#### Consumption (kWh/km, Real-World Averages)

| Model | Consumption |
|---|---|
| iX1 xDrive30 | 0.18 |
| iX1 | 0.17 |
| iX2 xDrive30 | 0.18 |
| iX2 | 0.17 |
| iX3 | 0.20 |
| iX M60 | 0.24 |
| iX xDrive60 | 0.21 |
| iX xDrive50 | 0.22 |
| iX xDrive40 | 0.22 |
| iX | 0.22 |
| i4 M50 | 0.21 |
| i4 eDrive40 | 0.18 |
| i4 eDrive35 | 0.17 |
| i4 | 0.18 |
| i5 M60 | 0.20 |
| i5 eDrive40 | 0.18 |
| i5 xDrive40 | 0.18 |
| i5 | 0.18 |
| i7 M70 | 0.23 |
| i7 xDrive60 | 0.21 |
| i7 eDrive50 | 0.20 |
| i7 | 0.21 |

#### Battery Capacity (Usable kWh)

| Model | Capacity |
|---|---|
| iX1 xDrive30 | 64.7 |
| iX1 | 64.7 |
| iX2 xDrive30 | 64.7 |
| iX2 | 64.7 |
| iX3 | 74.0 |
| iX M60 | 105.2 |
| iX xDrive60 | 105.2 |
| iX xDrive50 | 105.2 |
| iX xDrive40 | 71.0 |
| iX | 76.6 |
| i4 M50 | 80.7 |
| i4 eDrive40 | 80.7 |
| i4 eDrive35 | 59.4 |
| i4 | 80.7 |
| i5 M60 | 81.2 |
| i5 eDrive40 | 81.2 |
| i5 xDrive40 | 81.2 |
| i5 | 81.2 |
| i7 M70 | 101.7 |
| i7 xDrive60 | 101.7 |
| i7 eDrive50 | 101.7 |
| i7 | 101.7 |

---

## 12. Error Handling & Resilience

### 12.1 MQTT Circuit Breaker (`stream_circuit_breaker.py`)

Prevents reconnection storms after repeated MQTT failures:

| Parameter | Value |
|---|---|
| Failure window | 10 failures in 60 seconds opens the circuit |
| Open duration | 300 seconds (5 minutes) |
| Extended backoff threshold | 10 consecutive failures |
| Extended backoff duration | 600 seconds (10 minutes) |
| Max backoff | 300 seconds (5 minutes) for standard retries |
| State persistence | Converts monotonic ↔ wall clock time across restarts |

**State Transitions:**
```
CLOSED ──(failures exceed threshold)──► OPEN ──(timeout expires)──► HALF-OPEN
  ▲                                                                    │
  │                                                                    │
  └──────────────────(success)─────────────────────────────────────────┘
```

### 12.2 HTTP Rate Limiting (`ratelimit.py`)

**`RateLimitTracker`** — Handles HTTP 429 (Too Many Requests):
- Exponential backoff: 1h → 2h → 4h → 8h → 24h max
- Respects `Retry-After` header (clamped 60s–24h)
- Resets after 10 consecutive successful requests

**`UnauthorizedLoopProtection`** — Prevents 401/403 retry storms:
- Blocks after 3 consecutive unauthorized attempts
- Cooldown period: 1 hour

**`ContainerRateLimiter`** — Container operation limits:
- Maximum 3 operations per hour
- Maximum 10 operations per day

### 12.3 HTTP Retry Logic (`http_retry.py`)

**Retry policy for API calls:**

| Condition | Action |
|---|---|
| Timeout/ClientError | Retry with exponential backoff + jitter |
| 5xx server errors | Retry (408, 500, 502, 503, 504) |
| 429 Rate limited | No retry; record in RateLimitTracker |
| 401/403 Unauthorized | No retry; trigger reauth flow |
| 400, 404, 405, 410, 422 | No retry (permanent failure) |

**Backoff strategy:** Equal jitter — half guaranteed delay + random up to half
**Max retries:** Configurable (0–10), default varies by caller

### 12.4 MQTT Reconnection Strategy

```
Attempt 1: Wait 5 seconds
Attempt 2: Wait 10 seconds
Attempt 3: Wait 20 seconds
...
Attempt N: Wait min(5 × 2^N, 300) seconds  (max 5 minutes)

After 10 failures: Switch to extended backoff (10 minutes)

Auth failure (RC 4/5): Stop retry, trigger reauth flow
Custom broker auth failure: 6 retries then permanent stop
```

### 12.5 Session Health Monitoring (`runtime.py`)

- Tracks consecutive aiohttp session failures
- After 5 failures: automatically recreates the HTTP session
- Validates connector health before API calls

### 12.6 Global Connection Lock

A global asyncio lock (`_GLOBAL_MQTT_CONNECT_LOCK`) serializes all MQTT connection attempts across multiple config entries, preventing simultaneous connections from overwhelming BMW's server.

### 12.7 Token Refresh Protection

- `token_refresh_lock` prevents concurrent refresh attempts
- Double-check pattern: verify expiry after acquiring lock
- Minimum 30-second gap between refresh attempts
- `ERR_TOKEN_REFRESH_IN_PROGRESS` sentinel for concurrent detection

---

## 13. Advanced Features

### 13.1 SOC Prediction (During Charging)

**File:** `soc_prediction.py`

Predicts charge completion time and current SOC% during active charging sessions.

**How it works:**
1. Detects charging start from `DESC_CHARGING_STATUS` transitioning to an active state
2. Anchors session with current SOC, battery capacity, charging method (AC/DC)
3. Integrates power readings over time to estimate energy delivered
4. Applies learned or default efficiency to predict actual SOC gain
5. Extrapolates to estimate time to reach target SOC

**Charging active states:**
```
CHARGINGACTIVE, CHARGING_ACTIVE, CHARGING, CHARGING_IN_PROGRESS
```

**Default efficiencies:**
- AC: 90%
- DC: 93%

**Learning:** Efficiency is learned per charging condition (AC phases/voltage/current bracket; DC voltage bracket) using Exponential Moving Average (LEARNING_RATE = 0.2).

### 13.2 Magic SOC (During Driving)

**File:** `magic_soc.py`

Provides consumption-based remaining range estimation while driving.

**How it works:**
1. Tracks driving sessions anchored at trip start (SOC + mileage/GPS distance)
2. Monitors SOC drop and distance traveled
3. Calculates real-time consumption (kWh/km)
4. Applies model-specific defaults initially, learns over trips
5. Predicts remaining range from current SOC and consumption rate

**Learning:** Consumption is learned per vehicle using weighted EMA. Short trips contribute less (weighted by `REFERENCE_LEARNING_TRIP_KM = 30km`).

### 13.3 Motion Detection

**File:** `motion_detection.py`

GPS-based vehicle motion detection with multiple fallback data sources.

**Detection priority:**
1. **Charging gate**: If vehicle is charging → always NOT MOVING
2. **GPS (primary)**: Movement within 2-minute active window
3. **Door lock state**: If unlocked after GPS goes stale (30-second grace)
4. **Mileage (fallback)**: Odometer increase after GPS unavailable for 5+ minutes

**Parking zone logic:**
- Park radius: 35 meters (GPS jitter tolerance)
- Escape radius: 70 meters (2× park radius, movement confirmation)
- Rolling window: 10 readings, requiring 90-second minimum time span

### 13.4 Container Management

**File:** `container.py`

Manages BMW CarData "containers" — server-side configurations that define which telemetry descriptors are included in API responses.

**HV Battery Container:**
- Name: `"BMW CarData HV Battery"`
- Contains 30 descriptors (see Section 9.2)
- Created during bootstrap, validated on token refresh
- Descriptor signature hash ensures container stays in sync
- `CONTAINER_REUSE_EXISTING = True` prevents accumulation

### 13.5 Custom MQTT Broker

For users who want to bridge BMW data through their own MQTT infrastructure:

- Enable via options flow (`OPTION_CUSTOM_MQTT_ENABLED`)
- Configure host, port, credentials, TLS mode
- Topic prefix: configurable (default `bmw/`)
- Supports: `off`, `tls`, `tls_insecure` TLS modes

---

## 14. File-by-File Module Reference

### Core Integration

| File | Lines | Purpose |
|---|---|---|
| `__init__.py` | — | Package init, forwards setup to lifecycle |
| `lifecycle.py` | ~706 | Entry setup/unload orchestration; platform registration; task management |
| `runtime.py` | ~237 | Per-entry runtime state container |
| `coordinator.py` | ~800+ | Central state management; message processing; signal dispatch |
| `stream.py` | ~700+ | MQTT client lifecycle; Paho callbacks; connection state machine |
| `const.py` | ~273 | All constants, descriptors, model defaults, option keys |

### Authentication & API

| File | Lines | Purpose |
|---|---|---|
| `auth.py` | ~681 | Token refresh loop; unauthorized handling; container sync |
| `device_flow.py` | ~281 | OAuth 2.0 Device Code flow implementation |
| `config_flow.py` | ~340 | HA config flow steps (user → authorize → tokens) |
| `options_flow.py` | ~540 | Settings UI; service triggers from options menu |
| `http_retry.py` | ~355 | Resilient HTTP with exponential backoff and rate awareness |

### Data Acquisition

| File | Lines | Purpose |
|---|---|---|
| `bootstrap.py` | ~580 | VIN discovery; metadata fetch; container creation; initial seeding |
| `telematics.py` | ~775 | Periodic polling loop; trip-end triggers; daily fetches |
| `container.py` | ~300+ | HV battery container CRUD; descriptor signature validation |
| `metadata.py` | ~700 | Device metadata building; BEV detection; state restoration |

### Entity Platforms

| File | Lines | Purpose |
|---|---|---|
| `entity.py` | ~241 | Base entity class; naming; device info |
| `sensor.py` | ~558 | Generic telemetry sensors |
| `binary_sensor.py` | ~276 | Boolean state entities |
| `device_tracker.py` | ~515 | GPS location with coordinate pairing |
| `image.py` | ~235 | Vehicle photo entity |
| `button.py` | ~228 | Learning reset buttons |
| `number.py` | ~211 | Battery capacity override |
| `sensor_diagnostics.py` | ~540 | Connection, metadata, efficiency, charging, tyre sensors |

### Computed Features

| File | Lines | Purpose |
|---|---|---|
| `soc_prediction.py` | ~817 | Charging time prediction; session tracking |
| `soc_learning.py` | ~466 | Charging efficiency learning; session finalization |
| `magic_soc.py` | ~704 | Driving consumption prediction |
| `motion_detection.py` | ~637 | GPS-based motion detection |

### Infrastructure

| File | Lines | Purpose |
|---|---|---|
| `stream_circuit_breaker.py` | ~176 | MQTT connection failure protection |
| `ratelimit.py` | ~348 | API quota tracking; 429/401 protection |
| `pending_manager.py` | ~300 | Update debouncing; deduplication locks |
| `descriptor_state.py` | ~42 | Per-descriptor state dataclass |
| `utils.py` | ~264 | VIN validation; redaction; async helpers |
| `migrations.py` | ~212 | Entity ID prefix migration |
| `device_info.py` | ~220 | Device metadata and BEV detection helpers |

### UI & Services

| File | Lines | Purpose |
|---|---|---|
| `frontend_cards.py` | ~312 | Lovelace card registration; websocket API |
| `services.py` | ~700 | Developer service handlers |
| `services.yaml` | — | Service definitions for HA Developer Tools |

---

## 15. Dependencies

### External Dependencies

| Package | Version | Purpose |
|---|---|---|
| `paho-mqtt` | `>= 1.6.1` | MQTT client library for BMW streaming |

### Home Assistant Dependencies

| Component | Purpose |
|---|---|
| `homeassistant.components.sensor` | Sensor entity platform |
| `homeassistant.components.binary_sensor` | Binary sensor entity platform |
| `homeassistant.components.device_tracker` | Device tracker entity platform |
| `homeassistant.components.image` | Image entity platform |
| `homeassistant.components.button` | Button entity platform |
| `homeassistant.components.number` | Number entity platform |
| `homeassistant.helpers.dispatcher` | Signal system for entity updates |
| `homeassistant.helpers.device_registry` | Device management |
| `homeassistant.helpers.entity_registry` | Entity management |
| `homeassistant.helpers.restore_state` | State persistence across restarts |
| `homeassistant.helpers.storage` | Persistent data storage |
| `homeassistant.helpers.event` | Async task scheduling |
| `aiohttp` | HTTP client (built into HA) |
| `voluptuous` | Config validation (built into HA) |

### After-Dependencies

```json
"after_dependencies": ["http", "lovelace"]
```

The integration loads after `http` (for static file serving) and `lovelace` (for frontend card registration).

### Python Standard Library

`asyncio`, `json`, `logging`, `time`, `threading`, `hashlib`, `base64`, `dataclasses`, `enum`, `collections.abc`, `re`, `pathlib`, `uuid`, `math`

### Development Dependencies

| Tool | Purpose |
|---|---|
| `pytest` | Test framework |
| `pytest-homeassistant-custom-component` | HA test fixtures |
| `ruff` | Python linter (config in `ruff.toml`) |

---

*This document was generated from a comprehensive review of the BMW CarData Home Assistant integration source code (version 5.0.2, 47 Python modules, ~18,000 lines of code).*
