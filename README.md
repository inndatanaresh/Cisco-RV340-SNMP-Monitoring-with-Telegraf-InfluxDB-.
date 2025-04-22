# Cisco RV340 Monitoring with Telegraf and InfluxDB v2 on Rocky Linux 9

## ✅ Overview

This guide outlines the steps to collect SNMP metrics from a Cisco RV340 router using **Telegraf** and store them in **InfluxDB v2** on **Rocky Linux 9**.

---

## ✅ Prerequisites

- Cisco RV340 with SNMP v2c enabled.
- Rocky Linux 9 machine with internet access.
- Root or sudo privileges.
- InfluxDB v2.x installed and running with Web UI available at `http://<your-server-ip>:8086`.

---

## ✅ Step 1: Enable SNMP on Cisco RV340

1. **Login to RV340 Web UI**.
2. Navigate to:  
   **System Configuration** → **SNMP Settings**.
3. Enable SNMP and configure:
   - **SNMP Enabled**: ✅
   - **SNMP Version**: v2c
   - **Get Community**: `public`
   - **Trap Receiver IP**: `<Your Telegraf VM IP>`
   - **Port**: `161`

---

## ⚙️ Step 2: Install Telegraf on Rocky Linux 9

```bash
# Install EPEL repository
sudo dnf install -y epel-release

# Install Telegraf (if available)
sudo dnf install -y telegraf
```

📝 If Telegraf is not found, add InfluxData repository:

```bash
sudo tee /etc/yum.repos.d/influxdata.repo > /dev/null <<EOF
[influxdata]
name=InfluxData Repository - RHEL
baseurl=https://repos.influxdata.com/rhel/8/\$basearch/stable
enabled=1
gpgcheck=1
gpgkey=https://repos.influxdata.com/influxdata-archive.key
EOF

# Install Telegraf
sudo dnf install -y telegraf
```

---

## ⚙️ Step 3: Configure Telegraf for SNMP Input

```bash
sudo nano /etc/telegraf/telegraf.conf
```

Add or modify the following configuration:

```toml
[[inputs.snmp]]
  agents = [ "udp://<RV340-IP>:161" ]  # Replace with your RV340 IP
  version = 2
  community = "public"

  [[inputs.snmp.field]]
    oid = "RFC1213-MIB::sysUpTime.0"
    name = "sysUptime"
    conversion = "float(2)"

  [[inputs.snmp.field]]
    oid = "RFC1213-MIB::sysName.0"
    name = "sysName"
    is_tag = true

  [[inputs.snmp.table]]
    oid = "IF-MIB::ifTable"
    name = "interface"
    inherit_tags = ["sysName"]

    [[inputs.snmp.table.field]]
      oid = "IF-MIB::ifDescr"
      name = "ifDescr"
      is_tag = true
```

📅 Tip: Uncomment or add more fields under `snmp.table.field` to include more SNMP interface metrics.

---

## ✅ Step 4: Test Telegraf SNMP Configuration

```bash
telegraf --config /etc/telegraf/telegraf.conf --test | grep snmp
```

---

## ✅ Step 5: Configure Telegraf Output to InfluxDB v2

Before configuring Telegraf output, open the InfluxDB Web UI:

1. Access the Web UI at `http://<INFLUXDB-IP>:8086`.
2. Create a new **Organization**.
3. Create a new **Bucket** named `cisco_metrics`.
4. Copy the **Token ID** after bucket creation — it will be used in the Telegraf config.

Now, update your `telegraf.conf`:

```toml
[[outputs.influxdb_v2]]
  urls = ["http://<INFLUXDB-IP>:8086"]  # Replace with your InfluxDB server IP
  token = "<your-api-token>"
  organization = "<your-org-name>"
  bucket = "cisco_metrics"
```

---

## ✅ Step 6: Enable and Start Telegraf Service

```bash
sudo systemctl enable telegraf --now
sudo systemctl status telegraf
```

---

## ✅ Verification

- Access InfluxDB Web UI at: `http://<your-influxdb-ip>:8086`
- Navigate to **Explore** tab and query the `cisco_metrics` bucket.
- Use filters like `interface`, `sysUptime`, or `sysName` to explore SNMP metrics.

```bash
sudo systemctl restart telegraf
```

---

## ✅ Grafana Integration

1. Open Grafana Web UI.
2. Go to **Connections** > **Add Data Source**.
3. Select **InfluxDB**.
4. Set the following:
   - **Query Language**: `Flux`
   - **URL**: `http://<influxdb-ip>:8086`
   - **Organization**: `<your-org-name>`
   - **Bucket**: `cisco_metrics`
   - **Token**: `<your-api-token>`
5. Click **Save & Test**.

---

## ✅ Summary

With this setup:

- Cisco RV340 SNMP metrics are collected via Telegraf.
- Metrics are stored in InfluxDB v2.
- Grafana can later be configured to visualize these metrics using the InfluxDB data source.

# Cisco RV340 SNMP Monitoring Dashboard Configuration

## Overview
This document provides the configuration for a comprehensive production-grade Grafana dashboard to monitor a Cisco RV340 router using SNMP data collected via Telegraf and stored in InfluxDB 2.x.

## Dashboard Organization
The dashboard is organized into six logical sections:
1. Router Overview - Key status metrics
2. Interface Status - Status of all router interfaces
3. Network Traffic - Overall and per-interface throughput
4. Packet Activity - Packet flow visualization
5. Error Monitoring - Error detection and tracking
6. System Resources - Monitoring of host system resources

## Panel Configurations

### 1. Router Overview Section

#### Router Uptime (Stat Panel)
```flux
from(bucket: "cisco_metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "snmp")
  |> filter(fn: (r) => r["_field"] == "sysUptime")
  |> filter(fn: (r) => r["agent_host"] == "10.10.5.254")
  |> filter(fn: (r) => r["sysName"] == "Cisco-Router-RV340")
  |> last()
  |> map(fn: (r) => ({ r with _value: r._value / 8640000.0 })) // Convert to days
```
**Panel Settings:**
- Type: Stat
- Visualization: Big Value
- Title: "Router Uptime (Days)"
- Unit: Days
- Color mode: Value
- Threshold: Green

#### System Load (Gauge Panel)
```flux
from(bucket: "cisco_metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "system")
  |> filter(fn: (r) => r["_field"] == "load1")
  |> filter(fn: (r) => r["host"] == "snmp1")
  |> last()
```
**Panel Settings:**
- Type: Gauge
- Title: "System Load (1m)"
- Thresholds: 0-1.5 green, 1.5-3 orange, 3+ red
- Max: Based on number of CPUs

#### CPU Usage (Gauge Panel)
```flux
from(bucket: "cisco_metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "cpu")
  |> filter(fn: (r) => r["_field"] == "usage_idle")
  |> filter(fn: (r) => r["host"] == "snmp1")
  |> filter(fn: (r) => r["cpu"] == "cpu-total")
  |> last()
  |> map(fn: (r) => ({ r with _value: 100.0 - r._value }))
```
**Panel Settings:**
- Type: Gauge
- Title: "CPU Usage (%)"
- Unit: Percent (0-100)
- Thresholds: 0-60 green, 60-80 yellow, 80-100 red

#### Memory Usage (Gauge Panel)
```flux
from(bucket: "cisco_metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "mem")
  |> filter(fn: (r) => r["_field"] == "used_percent")
  |> filter(fn: (r) => r["host"] == "snmp1")
  |> last()
```
**Panel Settings:**
- Type: Gauge
- Title: "Memory Usage (%)"
- Unit: Percent (0-100)
- Thresholds: 0-70 green, 70-85 yellow, 85-100 red

### 2. Interface Status Section

#### Interface Status Overview (Table Panel)
```flux
from(bucket: "cisco_metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "interface")
  |> filter(fn: (r) => r["_field"] == "ifOperStatus" or r["_field"] == "ifSpeed" or r["_field"] == "ifAdminStatus")
  |> filter(fn: (r) => r["agent_host"] == "10.10.5.254")
  |> filter(fn: (r) => r["sysName"] == "Cisco-Router-RV340")
  |> last()
  |> pivot(rowKey:["_time", "ifDescr"], columnKey: ["_field"], valueColumn: "_value")
  |> map(fn: (r) => ({ 
      r with 
      ifSpeed: r.ifSpeed / 1000000.0,
      Status: if r.ifOperStatus == 1 then "Up" else "Down",
      AdminStatus: if r.ifAdminStatus == 1 then "Enabled" else "Disabled"
    }))
```
**Panel Settings:**
- Type: Table
- Title: "Interface Status"
- Columns: 
  - ifDescr (renamed to "Interface")
  - Status (with value mapping)
  - ifSpeed (renamed to "Speed (Mbps)")
  - AdminStatus (with value mapping)
- Apply color coding for Status (Up = green, Down = red)

#### Interface Status Timeline (State Timeline Panel)
```flux
from(bucket: "cisco_metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "interface")
  |> filter(fn: (r) => r["_field"] == "ifOperStatus")
  |> filter(fn: (r) => r["agent_host"] == "10.10.5.254")
  |> map(fn: (r) => ({ r with _value: if r._value == 1 then "Up" else "Down" }))
```
**Panel Settings:**
- Type: State Timeline
- Title: "Interface Status Timeline"
- Value mapping: "Up" = 1 (green), "Down" = 0 (red)
- Group by: ifDescr

### 3. Network Traffic Section

#### Total Network Traffic (Time Series Panel)
```flux
from(bucket: "cisco_metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "interface")
  |> filter(fn: (r) => r["_field"] == "ifInOctets" or r["_field"] == "ifOutOctets")
  |> filter(fn: (r) => r["agent_host"] == "10.10.5.254")
  |> filter(fn: (r) => r["ifOperStatus"] == 1 or r["ifDescr"] =~ /WAN/ or r["ifDescr"] =~ /LAN/)
  |> derivative(unit: 1s, nonNegative: true)
  |> map(fn: (r) => ({ 
      r with 
      _value: r._value * 8.0 / 1000000.0,
      direction: if r._field == "ifInOctets" then "In" else "Out"
    }))
  |> drop(columns: ["_field"])
  |> aggregateWindow(every: v.windowPeriod, fn: mean)
```
**Panel Settings:**
- Type: Time Series
- Title: "Network Traffic (Mbps)"
- Unit: Mbps
- Stack series: On
- Legend: Auto
- Display Direction + Interface Name

#### Interface-Specific Traffic (Time Series Panel with Variable)
```flux
from(bucket: "cisco_metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "interface")
  |> filter(fn: (r) => r["_field"] == "ifInOctets" or r["_field"] == "ifOutOctets")
  |> filter(fn: (r) => r["ifDescr"] == v.interface)  // Use dashboard variable
  |> filter(fn: (r) => r["agent_host"] == "10.10.5.254")
  |> derivative(unit: 1s, nonNegative: true)
  |> map(fn: (r) => ({ 
      r with 
      _value: r._value * 8.0 / 1000000.0,
      metric: if r._field == "ifInOctets" then "Incoming" else "Outgoing"
    }))
  |> drop(columns: ["_field"])
  |> aggregateWindow(every: v.windowPeriod, fn: mean)
```
**Panel Settings:**
- Type: Time Series
- Title: "Traffic for ${interface} (Mbps)"
- Unit: Mbps
- Color by: metric (Incoming = blue, Outgoing = green)

### 4. Packet Activity Section

#### Packet Activity Heatmap (Heatmap Panel)
```flux
from(bucket: "cisco_metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "interface")
  |> filter(fn: (r) => r["_field"] == "ifInUcastPkts" or r["_field"] == "ifOutUcastPkts")
  |> filter(fn: (r) => r["agent_host"] == "10.10.5.254")
  |> derivative(unit: 1m, nonNegative: true)
  |> aggregateWindow(every: v.windowPeriod, fn: sum)
  |> pivot(rowKey:["_time", "ifDescr"], columnKey: ["_field"], valueColumn: "_value")
  |> map(fn: (r) => ({ 
      r with 
      total: (r.ifInUcastPkts + r.ifOutUcastPkts)
    }))
  |> keep(columns: ["_time", "ifDescr", "total"])
```
**Panel Settings:**
- Type: Heatmap
- Title: "Interface Packet Activity"
- Y Field: ifDescr
- X Field: _time
- Color value: total
- Color scheme: From Green (low) to Red (high)

#### Packet Counter (Bar Gauge Panel)
```flux
from(bucket: "cisco_metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "interface")
  |> filter(fn: (r) => r["_field"] == "ifInUcastPkts" or r["_field"] == "ifOutUcastPkts")
  |> filter(fn: (r) => r["agent_host"] == "10.10.5.254")
  |> filter(fn: (r) => r["ifOperStatus"] == 1 or r["ifDescr"] =~ /WAN/ or r["ifDescr"] =~ /LAN/)
  |> derivative(unit: 5m, nonNegative: true)
  |> aggregateWindow(every: v.windowPeriod, fn: last)
```
**Panel Settings:**
- Type: Bar Gauge
- Title: "Packet Rate (Last 5 min)"
- Display: Horizontal
- Calculate: Current value
- Color mode: Value

### 5. Error Monitoring Section

#### Interface Errors (Time Series Panel)
```flux
from(bucket: "cisco_metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "interface")
  |> filter(fn: (r) => r["_field"] == "ifInErrors" or r["_field"] == "ifOutErrors")
  |> filter(fn: (r) => r["agent_host"] == "10.10.5.254")
  |> derivative(unit: 1m, nonNegative: true)
  |> aggregateWindow(every: v.windowPeriod, fn: sum)
```
**Panel Settings:**
- Type: Time Series
- Title: "Interface Errors (per minute)"
- Alert threshold: Any value > 0
- Override color for errors to red
- Use Line Width 2 for better visibility

#### Error Table (Table Panel)
```flux
from(bucket: "cisco_metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "interface")
  |> filter(fn: (r) => r["_field"] == "ifInErrors" or r["_field"] == "ifOutErrors")
  |> filter(fn: (r) => r["agent_host"] == "10.10.5.254")
  |> last()
  |> pivot(rowKey:["ifDescr"], columnKey: ["_field"], valueColumn: "_value")
```
**Panel Settings:**
- Type: Table
- Title: "Interface Error Counters"
- Color cells with non-zero values red

### 6. System Resource Utilization

#### CPU Time Distribution (Pie Chart Panel)
```flux
from(bucket: "cisco_metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "cpu")
  |> filter(fn: (r) => r["_field"] == "usage_system" or r["_field"] == "usage_user" or r["_field"] == "usage_iowait" or r["_field"] == "usage_irq" or r["_field"] == "usage_idle")
  |> filter(fn: (r) => r["cpu"] == "cpu-total")
  |> filter(fn: (r) => r["host"] == "snmp1")
  |> aggregateWindow(every: v.windowPeriod, fn: mean)
  |> last()
```
**Panel Settings:**
- Type: Pie Chart
- Title: "CPU Utilization Distribution"
- Label: pretty field names (System, User, IO Wait, IRQ, Idle)

#### Memory & Swap Trend (Time Series Panel)
```flux
from(bucket: "cisco_metrics")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => (r["_measurement"] == "mem" or r["_measurement"] == "swap") and r["_field"] == "used_percent")
  |> filter(fn: (r) => r["host"] == "snmp1")
  |> aggregateWindow(every: v.windowPeriod, fn: mean)
  |> keep(columns: ["_time", "_value", "_measurement"])
  |> rename(columns: {_measurement: "metric"})
```
**Panel Settings:**
- Type: Time Series
- Title: "Memory & Swap Utilization"
- Unit: Percent (0-100)
- Show thresholds lines at 80% and 90%

## Dashboard Variables

### Interface Variable
```flux
import "influxdata/influxdb/schema"

schema.tagValues(
  bucket: "cisco_metrics",
  tag: "ifDescr",
  predicate: (r) => r._measurement == "interface" and r.agent_host == "10.10.5.254",
  start: -30d
)
```
- Name: interface
- Include All option: yes

### Time Period Variable
- Name: time_period
- Type: Custom
- Values: Last 30m, Last 1h, Last 6h, Last 12h, Last 24h, Last 7d

## Dashboard Layout
The dashboard should be arranged in a logical flow:
1. Row 1: Router status overview panels
2. Row 2: Interface status section panels
3. Row 3: Network traffic monitoring panels
4. Row 4: Packet activity panels
5. Row 5: Error monitoring panels
6. Row 6: System resources panels

## Production Requirements

### Refresh Rate
- Auto refresh every 30 seconds for real-time monitoring

### Alert Rules
1. Interface Status Changes
   - Condition: ifOperStatus changes from 1 to 2 (Up to Down)
   - Severity: Critical
   - Message: "Interface ${ifDescr} is down"

2. Error Detection
   - Condition: ifInErrors or ifOutErrors > 0
   - Severity: Warning
   - Message: "Errors detected on interface ${ifDescr}"

3. High Resource Usage
   - CPU: Usage > 80% for 5 minutes
   - Memory: Usage > 85% for 5 minutes
   - Severity: Warning

### Annotations
Configure annotations to mark important events:
- Router reboots (detected by sysUptime resets)
- Configuration changes
- Alert notifications

### Performance Considerations
1. Optimize queries using appropriate aggregation functions
2. Use non-negative derivatives for counters
3. Set appropriate time range defaults (suggestion: 1 hour default)
4. Apply data transformations in Flux queries rather than in Grafana when possible

### User Experience
1. Apply consistent color schemes across panels:
   - Incoming traffic: Blue
   - Outgoing traffic: Green
   - Errors: Red
   - Status Up: Green
   - Status Down: Red

2. Add descriptive tooltips to panels explaining metrics
3. Include dashboard documentation with field descriptions

## Maintenance Procedures
1. Verify SNMP connectivity daily
2. Review alert history weekly
3. Update device inventory as network changes
4. Backup dashboard configuration regularly

## Troubleshooting
If metrics are missing or incorrect:
1. Verify SNMP service is running on the router
2. Check Telegraf configuration
3. Validate community strings and permissions
4. Inspect InfluxDB storage and retention policies

