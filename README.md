# Doorbell — Smart Presence System

A Bluetooth-native smart doorbell system built in Java. The hub lives at your front door on a Raspberry Pi, connects to a camera over Bluetooth, and streams live video and visitor events directly to a Swing dashboard on your home computer — no internet, no cloud, no subscription.

---

## How it works

When someone presses the doorbell:

- **Single press** — plays your current status message out the front speaker ("I'm busy right now — to ring the bell, hold for 3 seconds. To leave a message, press twice.")
- **Double press** — opens a 5-minute voice message recording for the visitor
- **Hold 3 seconds** — emergency override, rings the house normally regardless of status

You set your status from the dashboard on your computer. The Pi knows instantly. Everything communicates over Bluetooth — no router configuration, no port forwarding, no IP addresses.

---

## Architecture

```
┌─────────────────┐         Bluetooth          ┌──────────────────────┐
│   Raspberry Pi  │ ◄─────────────────────────► │  Swing Dashboard     │
│   (hub)         │                             │  (your computer)     │
│                 │                             │                      │
│  - Button GPIO  │         Bluetooth          │  - Status control    │
│  - AudioPlayer  │ ◄──────────────────────────  │  - Live camera feed  │
│  - BluetoothSrv │                             │  - Visitor log       │
│  - Camera mgmt  │                             │  - Message playback  │
└────────┬────────┘                             └──────────────────────┘
         │ Bluetooth
         ▼
┌─────────────────┐
│   Camera module │
│  - Live stream  │
│  - Snapshots    │
└─────────────────┘
```

**Three Bluetooth connections:**
- Pi ↔ Camera — live stream + snapshots
- Pi ↔ Dashboard — events, status updates, compressed live feed

**No internet required.** The hub is air-gapped by design. Recordings stay on device.

---

## Project structure

```
doorbell/
├── pi/                          # Runs on Raspberry Pi
│   ├── DoorbellButton.java      # GPIO listener — detects single/double/hold
│   ├── AudioPlayer.java         # Plays status messages out front speaker
│   ├── PiClient.java            # Bluetooth client — sends events to backend
│   ├── PiBoot.java              # Pi startup sequence
│   └── PiConfig.java            # Pin numbers, device addresses, constants
│
├── backend/
│   ├── data/
│   │   ├── media/               # 24hr rolling recordings, 3-day retention
│   │   ├── messages/            # Visitor voice messages (5 min max each)
│   │   ├── errors/              # System error logs
│   │   ├── system/              # Version info, update timestamps
│   │   └── user/                # settings.json — passcode, theme, schedule
│   │
│   ├── systemrun/
│   │   ├── Boot.java            # Starts server, loads config, connects devices
│   │   ├── FindUpdate.java      # Polls for updates periodically
│   │   ├── Update.java          # Downloads and applies updates
│   │   ├── ErrorLog.java        # Writes timestamped logs to data/errors/
│   │   └── MediaPurge.java      # Deletes recordings older than 3 days
│   │
│   ├── bluetooth/
│   │   ├── BluetoothServer.java # Pi-side — accepts connections from UI + camera
│   │   ├── BluetoothClient.java # UI-side — connects to Pi, receives events
│   │   ├── EventDispatcher.java # Parses messages, routes to handlers
│   │   ├── StreamReceiver.java  # Receives compressed video frames
│   │   └── MessageProtocol.java # Defines message format between all devices
│   │
│   ├── camera/
│   │   ├── CameraManager.java   # Manages camera connection and state
│   │   ├── SnapshotCapture.java # Grabs still frame on button press
│   │   ├── RecordingSession.java# Starts/stops timed 24hr recording segment
│   │   └── StreamServer.java    # Serves compressed stream to dashboard
│   │
│   ├── core/
│   │   ├── DoorbellServer.java      # Routes all incoming Pi events
│   │   ├── ButtonEventHandler.java  # Decides what happens per press type
│   │   ├── StatusManager.java       # Owns current status, reads/writes settings
│   │   ├── MessageRecorder.java     # Records visitor audio, enforces 5 min cap
│   │   ├── VisitorLog.java          # Appends visit entries with metadata
│   │   ├── Scheduler.java           # Auto DND by time-of-day
│   │   ├── TrustedVisitors.java     # Always-ring whitelist
│   │   ├── AudioMessageBuilder.java # Assembles correct audio per status
│   │   └── NotificationSender.java  # Pushes visit alerts to dashboard
│   │
│   └── model/                   # Shared data classes
│       ├── Status.java          # Enum: AVAILABLE, BUSY, DND, SLEEPING, CUSTOM
│       ├── VisitorEntry.java    # Timestamp, snapshot path, message path
│       ├── UserSettings.java    # Passcode hash, theme, schedule, trusted list
│       └── PressEvent.java      # Enum: SINGLE, DOUBLE, HOLD
│
└── UI/
    ├── boot/
    │   ├── BootScreen.java      # Splash screen while backend connects
    │   └── PasscodeScreen.java  # PIN entry on startup
    │
    ├── dashboard/
    │   ├── DashboardFrame.java      # Main JFrame — assembles all panels
    │   ├── StatusPanel.java         # Mode selector with one-tap switching
    │   ├── LiveFeedPanel.java       # Live Bluetooth camera stream
    │   ├── VisitorLogPanel.java     # Scrollable visit history
    │   ├── MessagePlaybackPanel.java# Audio player for visitor messages
    │   ├── SchedulePanel.java       # 24×7 grid for auto DND windows
    │   ├── TrustedVisitorPanel.java # Manage always-ring list
    │   └── CustomMessagePanel.java  # Record your own status messages
    │
    ├── settings/
    │   ├── SettingsPanel.java       # Tabbed settings container
    │   ├── PasscodeSettings.java    # Change PIN
    │   ├── ThemeSettings.java       # Light/dark/color theme
    │   ├── StorageSettings.java     # Disk usage + manual purge
    │   └── SystemInfo.java          # Version, update status, connection state
    │
    ├── components/
    │   ├── StatusBadge.java         # Reusable colored mode badge
    │   ├── VisitorCard.java         # Single visit row — thumbnail + time + icons
    │   ├── AudioPlayerWidget.java   # Reusable play/pause/scrub bar
    │   ├── TimeBlockGrid.java       # Clickable 24×7 scheduling grid
    │   └── ThemeManager.java        # Applies color theme globally
    │
    └── api/
        ├── BluetoothClient.java     # UI-side Bluetooth connection to Pi
        └── PollingService.java      # Heartbeat — refreshes UI state periodically
```

---

## Message protocol

All communication between devices is plain text over Bluetooth sockets, one message per line.

| Message | Direction | Meaning |
|---|---|---|
| `PRESS:SINGLE` | Pi → Backend | Single press detected |
| `PRESS:DOUBLE` | Pi → Backend | Double press detected |
| `PRESS:HOLD` | Pi → Backend | 3-second hold detected |
| `STATUS:AVAILABLE` | Backend → Pi | Status changed |
| `STATUS:BUSY` | Backend → Pi | Status changed |
| `STATUS:DND` | Backend → Pi | Status changed |
| `STATUS:SLEEPING` | Backend → Pi | Status changed |
| `STATUS:CUSTOM` | Backend → Pi | Status changed |
| `VISITOR:{timestamp}` | Backend → UI | New visitor event |
| `STREAM:{base64frame}` | Camera → Backend → UI | Compressed video frame |
| `ERROR:{message}` | Any → ErrorLog | System error |

---

## Status modes

| Mode | Behavior |
|---|---|
| **Available** | Rings normally |
| **Busy** | Plays busy message — hold to override |
| **Do Not Disturb** | Plays DND message — hold to override |
| **Sleeping** | Plays sleeping message — hold to override |
| **Custom** | Plays your recorded message — hold to override |

In every non-available mode, a 3-second hold bypasses the status and rings the house. The visitor is never silently turned away — they always know what to do.

---

## Data storage

All data lives locally on the hub. Nothing is sent to the cloud.

| Path | Contents | Retention |
|---|---|---|
| `data/media/` | 24hr rolling recordings, compressed | 3 days, auto-purged |
| `data/messages/` | Visitor voice messages, named by timestamp | Until manually deleted |
| `data/errors/` | Timestamped error logs | 30 days |
| `data/system/` | `version.json`, last update timestamp | Permanent |
| `data/user/` | `settings.json` — passcode hash, theme, schedule, trusted list | Permanent |

Recordings are stored at full quality locally. The live stream sent to the dashboard is compressed for Bluetooth bandwidth (~320×240, 15fps, MJPEG).

---

## Requirements

**Hardware:**
- Raspberry Pi 4 (or newer)
- Bluetooth-capable camera module
- Small speaker wired to Pi audio out
- Physical button wired to GPIO pin 18

**Software:**
- Java 17+
- Pi4J (GPIO library for Raspberry Pi)
- Bluecove or Bluez Java bindings (Bluetooth)
- javax.sound (built into Java — audio playback and recording)

---

## Setup

**1. Clone the repo**
```bash
git clone https://github.com/yourname/doorbell.git
cd doorbell
```

**2. Configure the Pi**
```bash
# Edit pi/PiConfig.java
GPIO_PIN = 18                    # your button pin
BACKEND_BT_ADDRESS = "XX:XX:XX:XX:XX:XX"   # your hub's Bluetooth address
```

**3. Build and deploy to Pi**
```bash
javac -cp pi4j-core.jar pi/*.java
scp *.class pi@raspberrypi.local:~/doorbell/
```

**4. Start the backend on the hub**
```bash
java -cp .:pi4j-core.jar:bluecove.jar backend.systemrun.Boot
```

**5. Launch the dashboard on your computer**
```bash
javac UI/**/*.java
java UI.boot.BootScreen
```

**6. Pair devices**

On first boot, the dashboard will scan for and pair with the Pi automatically. Approve the pairing on both devices when prompted.

---

## Design decisions

**Why Bluetooth instead of WiFi?**
It was easier for us to build and test. It also draws less power, requires less security enhancements, and is cheaper to integrate.

**Why no app?**
I'm lazy.

**Why plain text messages over Bluetooth?**
Simple to debug, simple to extend. If something breaks you can log the raw socket output and read exactly what happened. Binary protocols are faster but not meaningfully so at this scale.

**Why local storage only?**
Your doorbell footage is yours. It stays on your hardware, it gets auto-purged on your schedule, and it's never uploaded anywhere. This is a deliberate choice, not a limitation.

---

## Built by

Ryan Lee & Oliver White for the AP Computer Science A Final Project, Carlmont High School, 2026.

Ownership taken by Pigeon & Toast.

---

## License

All rights reserved.
No permission is granted to use, copy, modify, or distribute this software without explicit permission from the author.
