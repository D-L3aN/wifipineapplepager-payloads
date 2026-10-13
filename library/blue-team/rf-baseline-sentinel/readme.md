# RF-Baseline Sentinel

> **A passive, longitudinal RF behavioral baseline and anomaly correlation engine for the WiFi Pineapple Pager.**

RF-Baseline Sentinel learns what "normal" looks like in your RF environment over time — then alerts you when reality deviates. It's a **defensive blue-team tool** designed to help security researchers, red teamers operating in hostile environments, and privacy-conscious operators detect surveillance, rogue infrastructure, and RF anomalies that static signature-based tools miss.

---

## Table of Contents

- [Concept](#concept)
- [Why This Exists](#why-this-exists)
- [How It Works](#how-it-works)
- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
- [Understanding the Output](#understanding-the-output)
- [Anomaly Examples](#anomaly-examples)
- [Ethical Use & Legal](#ethical-use--legal)
- [Limitations](#limitations)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)
- [Acknowledgments](#acknowledgments)

---

## Concept

Traditional RF monitoring tools work on **static signatures** — "alert me when I see a deauth flood," "alert me when I see SSID `FreeWiFi`." These work well for known threats but miss everything else.

RF-Baseline Sentinel takes a different approach: **statistical behavioral profiling**.

Over days or weeks, it silently observes your RF environment and builds a baseline model:

- How many access points typically exist at 3 PM on a Tuesday?
- What's the normal client-to-AP ratio in your office?
- Which BSSIDs are "always there" vs. transient?
- What's the typical signal strength distribution?

Once the baseline is established, it flags statistically significant deviations — not because they match a known attack, but because **they don't match reality**.

---

## Why This Exists

This payload fills a gap that no other WiFi Pineapple Pager payload addresses:

| Existing Tools | RF-Baseline Sentinel |
|---|---|
| Static signatures ("alert on any deauth") | Behavioral baselines ("alert when deauth is anomalous for *this* time and place") |
| One-shot scanning | Longitudinal learning over days/weeks |
| Manual threshold tuning | Automatic statistical thresholds (z-score based) |
| Human interprets raw data | Automated correlation and contextual alerts |
| Reactive to known attacks | Detects novel/unknown anomalies |

**Real-world scenarios where this matters:**

- **Detecting targeted surveillance** — an unexpected AP appearing near your location at the same time every day
- **Rogue AP / evil twin detection** — a new BSSID appearing that mimics your baseline AP's characteristics
- **RF interference tracking** — gradual or sudden shifts in the RF landscape
- **Travel security** — carry the Pager, build a baseline of your home environment, then get alerted when something changes
- **Red team OPSEC** — detect when *you* are being monitored during an engagement

---

## How It Works
─────────────────────────────────────────────────────────────┐
│ RF-Baseline Sentinel │
├─────────────────────────────────────────────────────────────┤
│ │
│ ┌──────────────┐ ┌──────────────┐ ┌──────────┐ │
│ │ Passive │ │ SQLite │ │ Anomaly │ │
│ │ Recon Read │─────▶│ Baseline │─────▶│ Detector │ │
│ │ (every 60s) │ │ Storage │ │ (z-score)│ │
│ └──────────────┘ └──────────────┘ └──────────┘ │
│ │ │ │ │
│ │ │ ▼ │
│ │ │ ┌──────────┐ │
│ │ │ │ Alert │ │
│ │ │ │ (LED/ │ │
│ │ │ │ Vibe/ │ │
│ │ │ │ Log) │ │
│ │ │ └──────────┘ │
│ ▼ ▼ │
│ /root/recon/recon.db /root/loot/rf-baseline/ │
│ │
└─────────────────────────────────────────────────────────────┘

### Phase 1: Learning Mode

During the initial training period (default: 100 samples, one per minute), the payload:

1. Reads the Pager's passive recon database (`/root/recon/recon.db`)
2. Records AP count, client count, and hour-of-day for each sample
3. Populates a `bssid_registry` table tracking every BSSID seen
4. **Does not alert** — it's building the model

### Phase 2: Monitoring Mode

Once the baseline is established:

1. Takes a new sample every 2 minutes
2. Computes the z-score for the current AP count vs. historical average for **that hour of day**
3. If |z-score| > `Z_THRESHOLD` (default: 3.0), triggers an alert
4. Logs all alerts to `anomalies.log` with full context

### Why Hour-of-Day Matters

RF environments are highly time-dependent. A university campus at 3 AM looks nothing like it does at 3 PM. By baselining per-hour, the payload avoids false positives from normal temporal variation.

---

## Features

- ✅ **Fully passive** — reads existing recon data, does not transmit, does not scan actively
- ✅ **Longitudinal learning** — builds a statistical model over days/weeks
- ✅ **Hour-aware baselines** — accounts for time-of-day variation
- ✅ **Z-score anomaly detection** — statistically rigorous, no manual threshold tuning required
- ✅ **Persistent storage** — SQLite database survives reboots
- ✅ **BSSID registry** — tracks first-seen and last-seen for every device
- ✅ **Multi-modal alerts** — LED, vibration, and log file
- ✅ **Zero external dependencies** — uses only tools present on stock Pager firmware
- ✅ **Recoverable** — can pause, resume, or reset the baseline at any time
- ✅ **Exportable** — baseline DB can be pulled off-device for external analysis

---

## Requirements

| Requirement | Notes |
|---|---|
| WiFi Pineapple Pager | Current firmware |
| Recon service running | Default on stock firmware; can be enabled via Pager UI |
| `sqlite3` | Present on stock firmware |
| `bc` | For floating-point math; install via `opkg install bc` if missing |
| DuckyScript + Bash | Standard Pager payload environment |

**Storage:** The baseline grows by roughly 1 KB per day under normal conditions. A 30-day baseline uses < 1 MB.

---

## Installation

### Option 1: Via Payload Library (Recommended)

1. On your Pager, navigate to **Payloads → Blue Team → RF-Baseline Sentinel**
2. Ensure recon is running: `Pager UI → Recon → Start`
3. Run the payload

### Option 2: Manual Install

```bash
# Copy payload to the library
scp library/blue-team/rf-baseline-sentinel/payload.sh root@pager:/root/payloads/library/blue-team/rf-baseline-sentinel/

# SSH into the Pager
ssh root@pager

# Make executable
chmod +x /root/payloads/library/blue-team/rf-baseline-sentinel/payload.sh

# Run
/root/payloads/library/blue-team/rf-baseline-sentinel/payload.sh
ption 3: Running on Boot
To have RF-Baseline Sentinel start automatically on boot, add to /etc/rc.local:

bash
# RF-Baseline Sentinel - start in background
nohup /root/payloads/library/blue-team/rf-baseline-sentinel/payload.sh > /dev/null 2>&1 &
**Configuration**
# Directory where baseline DB and logs are stored
BASELINE_DIR="/root/loot/rf-baseline"

# Learning mode: set to "false" after baseline is established
LEARNING_MODE="true"

# Anomaly sensitivity: how many standard deviations before alerting
# Lower = more sensitive (more false positives)
# Higher = less sensitive (may miss subtle anomalies)
Z_THRESHOLD=3.0

# Minimum samples required before baseline is considered valid
# 100 samples at 1/minute = ~1.7 hours of learning
MIN_SAMPLES=100

# Sample interval during learning mode (seconds)
LEARN_INTERVAL=60

# Sample interval during monitoring mode (seconds)
MONITOR_INTERVAL=120
