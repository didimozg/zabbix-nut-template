# Universal NUT UPS Template for Zabbix 7.x

![Zabbix](https://img.shields.io/badge/Zabbix-7.0%2B-red) ![License](https://img.shields.io/github/license/didimozg/zabbix-nut-template)

Russian documentation: [README_RU.md](./README_RU.md).

Universal template for monitoring Uninterruptible Power Supplies (UPS) via **Network UPS Tools (NUT)** in **Zabbix 7.0+**.

This template is optimized for **APC Smart-UPS** but works with most UPS devices supported by NUT. It solves the common console "garbage" output issue (`Init SSL without certificate database`), correctly handles JSON Discovery, and supports automatic parameter detection.

## 📊 Features

The template automatically discovers and collects the following metrics:

* **Battery:**
    * Charge Level (%)
    * Voltage (V)
    * Temperature (°C)
    * **Runtime** (Remaining runtime in seconds/minutes)
    * Status (Charging, Discharging, Replace Battery, etc.)
* **Input / Output:**
    * Input/Output Voltage (V)
    * Frequency (Hz)
    * Output Current (A)
* **Load:**
    * Current UPS Load (%)
* **Inventory:**
    * Model, Serial Number, Battery Manufacture Date.

## 🛠️ Requirements

1.  Zabbix Server 7.0 or higher.
2.  Zabbix Agent (installed on the machine where the UPS is connected).
3.  Configured `nut-server` and `nut-client` (verify using `upsc -L`).

## ⚙️ Installation & Configuration (Linux)

To ensure correct Discovery and clean output (removing SSL warnings), a custom wrapper script is used.

### 1. Create the Handler Script

Create the script file on the host where the UPS is connected:

```bash
sudo nano /etc/zabbix/scripts/ups_handler.sh

```

Paste the following code:

```bash
#!/bin/bash

# IP address of your NUT server (usually 127.0.0.1 or server IP)
NUT_HOST="127.0.0.1" 
# If NUT requires authentication, use format: nut-user:nut-password@127.0.0.1

# --- MODE 1: Discovery ---
if [ "$1" = "ups.discovery" ]; then
    # Gets the list of UPS, removes SSL warnings, formats as Zabbix JSON
    /usr/bin/upsc -L $NUT_HOST 2>/dev/null | sed 's/://' | awk 'BEGIN {printf "{\"data\":["} {printf "{\"{#UPSNAME}\":\""$1"\"},"} END {print "]}"}' | sed 's/,]}/]}/'
    exit 0
fi

# --- MODE 2: Metric Collection ---
if [ -z "$3" ]; then
    # Option A: 2 parameters received (UPS Name, Metric) - New Template
    UPS_NAME=$1
    METRIC=$2
    TARGET_HOST=$NUT_HOST
else
    # Option B: 3 parameters received (Name, IP, Metric) - Legacy compatibility
    UPS_NAME=$1
    TARGET_HOST=$2
    METRIC=$3
fi

# Execute query, suppressing SSL errors
/usr/bin/upsc "${UPS_NAME}@${TARGET_HOST}" "$METRIC" 2>/dev/null

```

> **Note:** If your NUT server listens on an external IP, change the `NUT_HOST` variable inside the script (e.g., to `192.0.2.10`).

Make the script executable:

```bash
sudo chmod +x /etc/zabbix/scripts/ups_handler.sh

```

### 2. Configure Zabbix Agent

Open the agent configuration file (or create a separate config in `/etc/zabbix/zabbix_agentd.d/nut.conf`):

```bash
sudo nano /etc/zabbix/zabbix_agentd.conf

```

Add the following **UserParameter** at the end of the file:

```ini
UserParameter=upsmon[*],/etc/zabbix/scripts/ups_handler.sh "$1" "$2" "$3"

```

Restart the agent:

```bash
sudo systemctl restart zabbix-agent

```

### 3. Verify Operation (Test)

You can test the script from the Zabbix server or locally using `zabbix_get`:

```bash
# Check Discovery (should return clean JSON)
zabbix_get -s 127.0.0.1 -k upsmon[ups.discovery]

# Check Metric Collection (replace 'apc' with your UPS name)
zabbix_get -s 127.0.0.1 -k upsmon[apc,battery.charge]

```

---

## 📦 Installation in Zabbix (Web Interface)

1. Download the `zabbix_nut_universal_en.xml` file from this repository.
2. Go to the Zabbix Web Interface: **Data collection** -> **Templates**.
3. Click **Import** (top right corner).
4. Select the downloaded file and click **Import**.
5. Go to the **Hosts** section and find the host connected to the UPS.
6. Link the **NUT UPS Universal EN** template to the host.
7. Wait 1-2 minutes (or click "Execute now" in Discovery rules) for metrics to appear.

## 📝 License

This project is licensed under the **GPL v3** License.
Feel free to use, modify, and distribute this template.

Author: [didimozg](https://github.com/didimozg)
