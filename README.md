# Onocoy Smart Watchdog for RTKBase

An automated smart watchdog designed for Onocoy stations running via RTKBase. This script is specifically optimized for Raspberry Pi, Orange Pi, and other Linux-based systems to ensure 24/7 uptime and protect your Appreciation Streak.

## Installation

1. **Download the script:**
   ```bash
   wget [https://raw.githubusercontent.com/tomatdb52/onocoy-smart-watchdog/main/onocoy_watchdog.sh](https://raw.githubusercontent.com/tomatdb52/onocoy-smart-watchdog/main/onocoy_watchdog.sh)
   ```
   
2. **Make it executable:**

   ```bash
   chmod +x onocoy_watchdog.sh
   ```
   
3. **Configure:**
Edit the script and replace `KUMA_URL` and `SERVICE` with your specific details.

4. **Setup Cron:**
Add the script to your crontab to run every minute:

  ```bash
  crontab -e
  ```
  
Add the following line:

  ```
  * * * * * /bin/bash /home/pi/onocoy_watchdog.sh > /dev/null 2>&1
  ```

Note: Replace `/home/pi/` with your actual home directory, e.g., `/home/username/`

## Requirements
* RTKBase installed.
* `curl` installed.
* Sudo permissions for reading logs (`journalctl`) and restarting services.

## Proven Reliability
This system has been verified to detect "zombie" connection states (e.g., recv error 114) and resolve service failures automatically, restoring station services within 2 minutes of a downtime event.

----
```bash
#!/bin/bash

# --- CONFIGURATION ---
# Your unique Uptime Kuma Push URL (do not include &ping=, the script adds it automatically)
KUMA_URL="http://YOUR_KUMA_IP:3001/api/push/YOUR_KEY?status=up&msg=OK"
# The service you want to monitor (e.g., str2str_ntrip_B.service)
SERVICE="str2str_ntrip_B.service"
# --- END CONFIGURATION ---

# Start timer for latency measurement
START_TIME=$(date +%s%3N)

# 1. Check if the service is active in systemd
STATUS=$(systemctl is-active "$SERVICE")

# 2. Scan logs for common "zombie" patterns in the last 2 minutes
# This catches 'recv error (114)', timeouts, and connection loops
ERROR_COUNT=$(sudo journalctl -u "$SERVICE" --since "2 minutes ago" | grep -cE "error|connecting|timeout|114")

# Calculate elapsed time in ms for the Kuma ping graph
END_TIME=$(date +%s%3N)
ELAPSED=$((END_TIME - START_TIME))

if [ "$STATUS" = "active" ] && [ "$ERROR_COUNT" -lt 5 ]; then
    # Everything is healthy - Send heartbeat to Kuma
    curl -fsS --retry 3 "${KUMA_URL}&ping=${ELAPSED}" > /dev/null
else
    # Issue detected - Trigger self-healing by restarting the service
    sudo systemctl restart "$SERVICE"
    # Heartbeat is withheld, triggering your Uptime Kuma alert (e.g., Discord)
fi
```
