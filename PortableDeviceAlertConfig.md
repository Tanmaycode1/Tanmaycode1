# BYOD Device Alert & Policy Configuration

**Purpose**: Configure notifications for portable device connections to BYOD workstations

## 1. Overview & Requirements

### 1.1 Core Requirements
- Alert on portable device connections
- Document configurations and policies
- Enforce across all in-scope workstations
- Provide 10% sample evidence
- Document policy if technical implementation fails

### 1.2 Alert Types
- USB storage devices
- Mobile devices
- External drives
- Removable media

## 2. Technical Implementation

### 2.1 Device Detection (udev)
```bash
# /etc/udev/rules.d/99-byod-alerts.rules
# Monitor all portable device connections
ACTION=="add", SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", \
RUN+="/usr/local/bin/byod-alert.sh 'USB: $attr{idVendor}:$attr{idProduct}'"

# Monitor removable storage
ACTION=="add", SUBSYSTEM=="block", ENV{ID_BUS}=="usb", \
RUN+="/usr/local/bin/byod-alert.sh 'Storage: $env{ID_VENDOR_ID}'"

# Monitor mobile devices
ACTION=="add", SUBSYSTEM=="usb", ENV{ID_MEDIA_PLAYER}=="1", \
RUN+="/usr/local/bin/byod-alert.sh 'Mobile: $attr{manufacturer}'"

# Monitor all removals
ACTION=="remove", SUBSYSTEM=="usb", \
RUN+="/usr/local/bin/byod-alert.sh 'Device Removed: $attr{idVendor}'"
```

### 2.2 Alert Script
```bash
# /usr/local/bin/byod-alert.sh
#!/bin/bash

# Configuration
HOSTNAME=$(hostname)
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
LOG_FILE="/var/log/byod/alerts.log"
ALERT_RECIPIENTS=(
    "security-team@company.com"
    "it-alerts@company.com"
    "compliance@company.com"
)

# Ensure log directory exists
mkdir -p /var/log/byod

# Log the event
log_event() {
    echo "$TIMESTAMP - $HOSTNAME - $1" >> $LOG_FILE
    logger -p auth.warning -t BYOD_MONITOR "$1"
}

# Send email alert
send_alert() {
    for recipient in "${ALERT_RECIPIENTS[@]}"; do
        echo "BYOD Alert: Device Connection Detected
Time: $TIMESTAMP
Host: $HOSTNAME
Event: $1" | mail -s "BYOD Alert - $HOSTNAME" "$recipient"
    done
}

# Main execution
log_event "$1"
send_alert "$1"
```

### 2.3 Audit Collection
```bash
# /usr/local/bin/byod-audit.sh
#!/bin/bash

# Configuration
BYOD_HOSTS_FILE="/etc/byod/hosts.txt"
SAMPLE_PERCENTAGE=10
OUTPUT_DIR="/var/log/byod/audit"
DATE=$(date +%Y%m%d)

# Create sample set
total_hosts=$(wc -l < "$BYOD_HOSTS_FILE")
sample_size=$(( ($total_hosts * $SAMPLE_PERCENTAGE + 99) / 100 ))
sampled_hosts=$(shuf -n "$sample_size" "$BYOD_HOSTS_FILE")

# Collect evidence
for host in $sampled_hosts; do
    mkdir -p "$OUTPUT_DIR/$DATE/$host"
    # Collect logs and configs
    scp "$host:/var/log/byod/alerts.log" "$OUTPUT_DIR/$DATE/$host/"
    scp "$host:/etc/udev/rules.d/99-byod-alerts.rules" "$OUTPUT_DIR/$DATE/$host/"
done
```

## 3. Policy Documentation

### 3.1 Technical Limitations Policy
```markdown
# BYOD Device Connection Policy
Version: 1.0
Date: October 25, 2024

## Purpose
Define requirements for portable device connections to BYOD workstations when 
technical controls are not feasible.

## Technical Limitations Documentation
1. Infrastructure Constraints:
   - [Document specific limitations]
   - [List technical barriers]
   - [Detail system constraints]

## User Requirements
1. No unauthorized portable devices may be connected
2. All connections must be pre-approved by IT
3. Violations must be reported within 24 hours
4. Annual policy review and acceptance required

## User Acceptance
I acknowledge and accept the above policy requirements:

Name: _____________________
Employee ID: _______________
Date: _____________________
Signature: _________________
```

### 3.2 Enforcement Documentation
```markdown
# BYOD Control Enforcement
Version: 1.0

## Technical Controls
1. Real-time device monitoring via udev
2. Immediate alert generation
3. Multi-channel notifications
4. Centralized logging

## Administrative Controls
1. User acceptance documentation
2. Regular compliance audits
3. Violation reporting procedures
4. Annual policy renewals

## Evidence Collection
1. System logs retention
2. Alert notifications archive
3. User acceptance records
4. Monthly 10% sampling audits
```

## 4. Deployment & Verification

### 4.1 Installation Script
```bash
#!/bin/bash
# deploy-byod-monitoring.sh

# Create directories
mkdir -p /var/log/byod
mkdir -p /etc/byod

# Install components
cp 99-byod-alerts.rules /etc/udev/rules.d/
cp byod-alert.sh /usr/local/bin/
cp byod-audit.sh /usr/local/bin/

# Set permissions
chmod +x /usr/local/bin/byod-alert.sh
chmod +x /usr/local/bin/byod-audit.sh
chmod 755 /var/log/byod

# Setup monthly audits
echo "0 1 1 * * root /usr/local/bin/byod-audit.sh" > /etc/cron.d/byod-audit

# Restart services
systemctl restart udev
```

### 4.2 Verification Steps
```bash
# Check configuration
ls -l /etc/udev/rules.d/99-byod-alerts.rules
ls -l /usr/local/bin/byod-alert.sh

# Test alert system
/usr/local/bin/byod-alert.sh "TEST_ALERT"

# Verify logging
tail /var/log/byod/alerts.log

# Check audit system
/usr/local/bin/byod-audit.sh --test
```

## 5. Implementation Notes

### 5.1 System Requirements
- Linux OS with udev support
- Mail system configured
- Syslog service running
- SSH access for audit collection

### 5.2 Alert Flow
1. Device Connection → udev Detection
2. Alert Script Triggered
3. Local Logging
4. Email Notifications
5. Syslog Recording

### 5.3 Audit Process
1. Random 10% Host Selection
2. Log Collection
3. Configuration Verification
4. Evidence Archive
5. Report Generation

### 5.4 Policy Compliance
When technical implementation isn't possible:
1. Document specific limitations
2. Obtain user acceptance
3. Maintain policy records
4. Regular compliance checks

## 6. Quick Reference

### 6.1 File Locations
```plaintext
/etc/udev/rules.d/99-byod-alerts.rules  # Device rules
/usr/local/bin/byod-alert.sh            # Alert script
/usr/local/bin/byod-audit.sh            # Audit script
/var/log/byod/alerts.log                # Alert log
/etc/byod/policy.md                     # Policy doc
```

### 6.2 Commands
```bash
# Deploy system
./deploy-byod-monitoring.sh

# Test alert
/usr/local/bin/byod-alert.sh "TEST"

# Run audit
/usr/local/bin/byod-audit.sh

# Check logs
tail -f /var/log/byod/alerts.log
```

This implementation provides:
- Real-time device monitoring
- Immediate alerts
- Policy documentation
- Evidence collection
- Compliance tracking
- Complete audit trail
