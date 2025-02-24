# Automated Disk Cleanup System

This system automatically cleans up files in a specified directory when the root user has been inactive for a defined period.

## Components
- Cleanup script
- Systemd service
- Systemd timer

## 1. Cleanup Script Setup

Create the script file `/root/disk_cleanup.sh`:

```bash
#!/bin/bash

TEST_DIR="/test"
LOG_FILE="/var/log/disk_cleanup.log"
IDLE_TIME=180  # 3 minutes (180 seconds)

# Get the last login time of root user
LAST_LOGIN=$(last -F root | head -1 | awk '{print $5, $6, $7, $8, $9}')
LAST_LOGIN_EPOCH=$(date -d "$LAST_LOGIN" +%s 2>/dev/null)

# If we can't get last login time, exit
if [[ -z "$LAST_LOGIN_EPOCH" ]]; then
    echo "[$(date)] Unable to get root login time. Skipping cleanup." | tee -a $LOG_FILE
    exit 1
fi

CURRENT_TIME=$(date +%s)
INACTIVE_DURATION=$((CURRENT_TIME - LAST_LOGIN_EPOCH))

if [[ $INACTIVE_DURATION -ge $IDLE_TIME ]]; then
    echo "[$(date)] Root user inactive for $((INACTIVE_DURATION / 60)) minutes. Deleting files..." | tee -a $LOG_FILE
    rm -rf "$TEST_DIR"/*
else
    echo "[$(date)] Root user accessed recently. Skipping cleanup." | tee -a $LOG_FILE
fi
```

Make the script executable:
```bash
chmod +x /root/disk_cleanup.sh
```

## 2. Service Configuration

Create the systemd service file `/etc/systemd/system/disk_cleanup.service`:

```ini
[Unit]
Description=Disk Cleanup Service
After=network.target

[Service]
Type=oneshot
ExecStart=/bin/bash /root/disk_cleanup.sh
```

## 3. Timer Configuration

Create the systemd timer file `/etc/systemd/system/disk_cleanup.timer`:

```ini
[Unit]
Description=Run Disk Cleanup Every 3 Minutes

[Timer]
OnBootSec=1min
OnUnitActiveSec=3min
Unit=disk_cleanup.service
Persistent=true

[Install]
WantedBy=timers.target
```

## 4. Activation and Testing

### Enable and Start the Timer
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now disk_cleanup.timer
```

### Verify Timer Status
```bash
systemctl list-timers --all | grep disk_cleanup
```

## Testing the Setup

1. Create test files in the `/test` directory
2. Log out from root:
```bash
exit
```
3. Wait for 3 minutes
4. Log back in and check if files in `/test` are deleted

## Monitoring

Check the log file for cleanup activities:
```bash
tail -f /var/log/disk_cleanup.log
```

## Important Notes

- The script monitors root user inactivity
- Cleanup occurs after 3 minutes of root user inactivity
- All files in `/test` directory will be deleted
- Activities are logged to `/var/log/disk_cleanup.log`

## System Requirements

- Linux system with systemd
- Root access
- `last` command available
- Write access to `/var/log`

## Security Considerations

- Script should be owned by root and not writable by others
- Test directory should be properly configured to avoid accidental deletion
- Log file should be protected from unauthorized access
