# Kismet Integration with Security Onion 2.4 - Standard Operating Procedure

---

## Table of Contents
[TOC]

---

## Overview

This SOP guides you through integrating Kismet wireless monitoring with Security Onion 2.4. The integration:
- Polls Kismet every 60 seconds for WiFi device data
- Transforms Kismet JSON into ECS (Elastic Common Schema) format
- Stores wireless network meta data in Elasticsearch
- Makes WiFi monitoring data searchable in Kibana
- Help searches
- Troubleshooting

**Time Required:** 30-45 minutes

**Skill Level:** Intermediate (requires familiarity with Security Onion, Kibana, and basic Linux)

---

## Prerequisites

### Required Software
- ✅ Security Onion 2.4 installed and running
- ✅ Kismet installed on a system with a wireless adapter in monitor mode
- ✅ Network connectivity between Security Onion and Kismet system

### Required Access
- ✅ SSH access to Security Onion server
- ✅ SSH/terminal access to Kismet system
- ✅ Kibana web interface credentials
- ✅ Kismet API key (readonly permission)

---

## Install Elastic Agent on Kismet Server

1. **Log** into Security Onion
2. On the left hand side **select Downloads**
3. Select the **Elastic Agent installer you need**, most likely linux.
4. Allow the Elastic Agent through the Sec Onion firewall
   1. Select **Administration** > **Configure**
   2. Select the blue hyperlink on the right hand side **Allow Elastic Agent endpoints to send logs**
   3. Enter in the **IP or IP range of the agents** > select the **green Check**
   4. Select **Options** at the top of the page select **SYNCHRONIZE GRID**
5. **SCP** the **Elastic Agent** to the kismet server
6. for linux **chmod** the elastic agent
```
chmod +x so-elastic-agent<tab out the rest>
```
7. **Install** the agent on the kismet server, **if it does not work the first time give Security Onion time to synchronize the grid.**
```
sudo ./so-elastic-agent<tab out the rest> install
```
Expect output:
```
Installation initiated, view install log for further details.


Installation completed successfully.
```
8. Make sure that **kismet server can reach your Security Onion via DNS**.
   1. **For example:** you need to take the hostname of the Security onion and the Domain of your network and make sure that your DNS can resolve that domain name/URL. The elastic agent is pre configured with you Sec Onions host and domain name as shown below.
   2. If my **Hostname** is `SecurityOnion`
   3. And my **networks domain** is `exampledomain.local`
   4. I would **add** `Securityonion.exampledomain.local` to your local DNS
      1.*You filled this information out when you made you security onion*
9. Verify the agent checked in back in **Security Onion** > **Elastic Fleet** > look for Kismet Servers Hostname

## Part 1: Create Elasticsearch Ingest Pipelines

### Method: CLI Copying from defaults to local

**Why copy from defaults to local:** Faster than doing it with the GUI, and security onion already packages the files you need in defaults. 

### Step 1.1: SSH into Sec Onion and copy over the default files

1. **SSH** into Sec Onion
2. **Copy** the 9 transform files from default to local
>**What This Does**
Creates 9 pipelines that transform Kismet's JSON data into Security Onion's format. The main pipeline routes data to device-type-specific sub-pipelines.
```
sudo cp /opt/so/saltstack/default/salt/elasticsearch/files/ingest/kismet* /opt/so/saltstack/local/salt/elasticsearch/files/ingest/
```
1. **Copy** the one template file default to local
>**What This Does**
Defines how Elasticsearch stores Kismet fields (data types for dates, numbers, nested objects).
```
sudo cp /opt/so/saltstack/default/salt/elasticsearch/templates/component/ecs/kismet* /opt/so/saltstack/local/salt/elasticsearch/templates/component/ecs/
```
1. **Copy** the one integrations-optional file default to local
```
sudo cp /opt/so/saltstack/default/salt/elasticfleet/files/integrations-optional/kismet* /opt/so/saltstack/local/salt/elasticfleet/files/integrations-optional/
```

### Replace the `kismet.common` with the correct config

**Reasoning:** The `kismet.common` file that security onion has a schema mismatch where the `kismet_common_rrd_day_vec` field fluctuated between `long` and `float` types, violating Elasticsearch's strict mapping rules.

1. Wipe everything inside `kismet.common`
```
sudo truncate -s 0 /opt/so/saltstack/local/salt/elasticsearch/files/ingest/kismet.common
```
2. Open `Kismet.common`
```
sudo vi /opt/so/saltstack/local/salt/elasticsearch/files/ingest/kismet.common
```
3. verify that `kismet.common` is empty
4. Type `i` to allow you to paste the correct configuration below `Ctrl + Shift + V`
```
{
  "processors": [
    {
      "json": {
        "field": "message",
        "target_field": "message2"
      }
    },
    {
      "remove": {
        "field": "message2.kismet_device_base_seenby",
        "ignore_missing": true
      }
    },
    {
      "date": {
        "field": "message2.kismet_device_base_mod_time",
        "formats": ["epoch_second"],
        "target_field": "@timestamp"
      }
    },
    {
      "set": {
        "field": "event.category",
        "value": "network"
      }
    },
    {
      "dissect": {
        "field": "message2.kismet_device_base_type",
        "pattern": "%{wifi} %{device_type}"
      }
    },
    {
      "lowercase": {
        "field": "device_type"
      }
    },
    {
      "set": {
        "field": "event.dataset",
        "value": "kismet.{{device_type}}"
      }
    },
    {
      "set": {
        "field": "event.dataset",
        "value": "kismet.wds_ap",
        "if": "ctx?.device_type == 'wds ap'"
      }
    },
    {
      "set": {
        "field": "event.dataset",
        "value": "kismet.ad_hoc",
        "if": "ctx?.device_type == 'ad-hoc'"
      }
    },
    {
      "set": {
        "field": "event.module",
        "value": "kismet"
      }
    },
    {
      "rename": {
        "field": "message2.kismet_device_base_packets_tx_total",
        "target_field": "source.packets"
      }
    },
    {
      "rename": {
        "field": "message2.kismet_device_base_num_alerts",
        "target_field": "kismet.alerts.count"
      }
    },
    {
      "rename": {
        "field": "message2.kismet_device_base_channel",
        "target_field": "network.wireless.channel",
        "if": "ctx?.message2?.kismet_device_base_channel != ''"
      }
    },
    {
      "rename": {
        "field": "message2.kismet_device_base_frequency",
        "target_field": "network.wireless.frequency",
        "if": "ctx?.message2?.kismet_device_base_frequency != 0"
      }
    },
    {
      "rename": {
        "field": "message2.kismet_device_base_last_time",
        "target_field": "kismet.last_seen"
      }
    },
    {
      "date": {
        "field": "kismet.last_seen",
        "formats": ["epoch_second"],
        "target_field": "kismet.last_seen"
      }
    },
    {
      "rename": {
        "field": "message2.kismet_device_base_first_time",
        "target_field": "kismet.first_seen"
      }
    },
    {
      "date": {
        "field": "kismet.first_seen",
        "formats": ["epoch_second"],
        "target_field": "kismet.first_seen"
      }
    },
    {
      "rename": {
        "field": "message2.kismet_device_base_manuf",
        "target_field": "device.manufacturer"
      }
    },
    {
      "pipeline": {
        "name": "{{event.dataset}}"
      }
    },
    {
      "remove": {
        "field": ["message2", "message", "device_type", "wifi", "agent", "host", "event.created"],
        "ignore_failure": true
      }
    }
  ]
}
```
5. Press the `esc` key and type `:wq!` to save and quite.


### Live Update kismet.common and Verify All Pipelines are Created

1. log into elastic
2. Click **☰ Menu** → **Management** → **Dev Tools**
3. Run the below configuration by hitting the little play button in the shell to update the current running config for `kismet.common`
```
PUT _ingest/pipeline/kismet.common
{
  "processors": [
    {
      "json": {
        "field": "message",
        "target_field": "message2"
      }
    },
    {
      "remove": {
        "field": "message2.kismet_device_base_seenby",
        "ignore_missing": true
      }
    },
    {
      "date": {
        "field": "message2.kismet_device_base_mod_time",
        "formats": ["epoch_second"],
        "target_field": "@timestamp"
      }
    },
    {
      "set": {
        "field": "event.category",
        "value": "network"
      }
    },
    {
      "dissect": {
        "field": "message2.kismet_device_base_type",
        "pattern": "%{wifi} %{device_type}"
      }
    },
    {
      "lowercase": {
        "field": "device_type"
      }
    },
    {
      "set": {
        "field": "event.dataset",
        "value": "kismet.{{device_type}}"
      }
    },
    {
      "set": {
        "field": "event.dataset",
        "value": "kismet.wds_ap",
        "if": "ctx?.device_type == 'wds ap'"
      }
    },
    {
      "set": {
        "field": "event.dataset",
        "value": "kismet.ad_hoc",
        "if": "ctx?.device_type == 'ad-hoc'"
      }
    },
    {
      "set": {
        "field": "event.module",
        "value": "kismet"
      }
    },
    {
      "rename": {
        "field": "message2.kismet_device_base_packets_tx_total",
        "target_field": "source.packets"
      }
    },
    {
      "rename": {
        "field": "message2.kismet_device_base_num_alerts",
        "target_field": "kismet.alerts.count"
      }
    },
    {
      "rename": {
        "field": "message2.kismet_device_base_channel",
        "target_field": "network.wireless.channel",
        "if": "ctx?.message2?.kismet_device_base_channel != ''"
      }
    },
    {
      "rename": {
        "field": "message2.kismet_device_base_frequency",
        "target_field": "network.wireless.frequency",
        "if": "ctx?.message2?.kismet_device_base_frequency != 0"
      }
    },
    {
      "rename": {
        "field": "message2.kismet_device_base_last_time",
        "target_field": "kismet.last_seen"
      }
    },
    {
      "date": {
        "field": "kismet.last_seen",
        "formats": ["epoch_second"],
        "target_field": "kismet.last_seen"
      }
    },
    {
      "rename": {
        "field": "message2.kismet_device_base_first_time",
        "target_field": "kismet.first_seen"
      }
    },
    {
      "date": {
        "field": "kismet.first_seen",
        "formats": ["epoch_second"],
        "target_field": "kismet.first_seen"
      }
    },
    {
      "rename": {
        "field": "message2.kismet_device_base_manuf",
        "target_field": "device.manufacturer"
      }
    },
    {
      "pipeline": {
        "name": "{{event.dataset}}"
      }
    },
    {
      "remove": {
        "field": ["message2", "message", "device_type", "wifi", "agent", "host", "event.created"],
        "ignore_failure": true
      }
    }
  ]
}
```
**Expected output:**
```
{
  "acknowledged": true
}
```
4. run the below command, by hitting the little play button:

```
GET _ingest/pipeline/kismet.*
```

**Expected output:** JSON response showing all 9 pipelines (kismet.common, kismet.seenby, kismet.ap, kismet.client, kismet.ad_hoc, kismet.bridged, kismet.device, kismet.wds, kismet.wds_ap).

---

## Part 2: Configure Fleet Integration

### What This Does
Configures the Elastic Agent on your Kismet system to poll the Kismet API every 60 seconds and send data through the pipelines to Elasticsearch.

### Step 2.1: Get Your Kismet API Key

On the system running Kismet:

**Option A - From Kismet Web UI:**
1. Browse to `http://KISMET-IP:2501`
2. Go to **Settings** → **API Keys**
3. Create a new API key with **readonly** role
4. Copy the key (long hexadecimal string)

### Step 2.2: Create a Kismet Agent Policy and Add the Kismet Integration

1. In Kibana, go to **Fleet** → **Agents policies**
2. Select the Agent Policy for your **Kismet servers Agent** 
   1. Likely "endpoints-initial" or similar, you can verify this on the agents page by looking for the kismet servers hostname
3. Select the 3 dots `...` for your **Agents policie**
4. Select **Duplicate**
   1. Name the New Policy `endpoints-Kismet#` *EX: endpoints-Kismet1, if you have other kismet serves the next will be 2 and so on.*
   2. Add a Description if you know where this kismet server will be placed, you can add this later
   3. Select **Duplicate Policy**
5. Click on the new policy you just made `EX: endpoints-Kismet#`
6. Click **"Add integration"**
7. Search for `Custom API` > Click **"Custom API"** > then **"Add Custom API"**

### Step 2.3: Configure the Custom API Integration

Fill in these fields:

**Integration settings:**
- **Integration name:** `Kismet Wireless Monitoring`
- **Description:** (optional) `Polls Kismet for WiFi device data`
- select **Advanced options** 
- **Namespace** `so`

**Custom API Input:**
- **Dataset name:** `kismet`
- **Ingest Pipeline:** `kismet.common`
- **Request URL:** 
  - Kismet is on the SAME system as the agent: `http://localhost:2501/devices/last-time/-600/devices.tjson`
> If Kismet was NOT, aka is on a DIFFERENT system: `http://KISMET-IP:2501/devices/last-time/-600/devices.tjson`
- **Request Interval:** `60s`
- **Request HTTP Method:** `GET`

**Request Transforms:** (Critical - sets authentication header)
- Replace `YOUR_API_KEY_HERE` with your actual Kismet API key. **Keep the single quotes!**
```yaml
- set:
    target: header.Cookie
    value: 'KISMET=YOUR_API_KEY_HERE'
```

**Leave all other fields blank/default.**

- Click **"Save and continue"** (bottom of page)
- Click **"Save and deploy changes"**

### Step 2.4 Assigning the new policy to your kismet Server

1. Select the Hamburger in the top right 
2. Select **Fleet**
3. Select **Agents**
4. Select the **hostname of your Kismet Server**
5. Select **Actions**
6. Select **Assign new agent policy** > select `endpoints-Kismet#` > **Assign Policy**

**What happens next:** The agent will receive the new configuration within 1-2 minutes and start polling Kismet every 60s.

---

## Verification

### Verify Data Collection is Working

**Step 1: Check Agent Logs (on Kismet system)**

```bash
sudo elastic-agent logs | grep -i "httpjson" | tail -5
```

**Expected output:** Should show lines like:
```
"request finished: ### events published"
"input_url":"http://127.0.0.1:2501/devices/last-time/-600/devices.tjson"
```

**Step 2: Check Data in Elasticsearch (on Security Onion)**

```bash
sudo so-elasticsearch-query logs-kismet-*/_count
```

**Expected output:** `{"count":###}` where ### is greater than 0 (number of WiFi devices/events collected). **EX:**
```
{"count":22081,"_shards":{"total":1,"successful":1,"skipped":0,"failed":0}}[root@securityonio integrations-optional]#
```

**Step 3: View Data in Kibana**

1. Open Kibana → **Discover**
2. Set index pattern to `logs-kismet-*`
3. You should see WiFi network data with fields like:
   - `network.wireless.ssid`
   - `network.wireless.bssid`
   - `network.wireless.channel`
   - `network.wireless.frequency`
   - `device.manufacturer`
   - `kismet.first_seen`
   - `kismet.last_seen`

**If you experience issues:**
- Increase poll interval to `15m` or `30m` in Fleet integration
- Reduce retention by adjusting index lifecycle policies
- Consider deploying Kismet on a separate sensor vs Security Onion itself


### Useful Kibana Searches *(More Below)*

Once in Discover with `logs-kismet-*` index:

- **All visible SSIDs:** `network.wireless.ssid:*`
- **Access points only:** `event.dataset:kismet.ap`
- **Client devices only:** `event.dataset:kismet.client`
- **Hidden networks:** `network.wireless.ssid:"Hidden"`
- **5GHz networks:** `network.wireless.frequency:>=5000000`
- **Networks with alerts:** `kismet.alerts.count:>0`

### Security Onion Kismet Dashboard 
- **Log into** Security Onion 
- On the left hand side select **Dashboards**
- From the search bar/drop down menu **scroll down to the bottom and select Kismet - WiFi Devices**

---

## Appendix A: Understanding the Data Flow

```
┌─────────────┐
│   Kismet    │  Captures WiFi packets
└──────┬──────┘
       │ REST API (every 60s)
       │
┌──────▼──────────┐
│ Elastic Agent   │  HTTPJson input polls Kismet
│ (on Kali/sensor)│  
└──────┬──────────┘
       │ Sends JSON via Logstash protocol
       │
┌──────▼──────────┐
│   Logstash      │  Receives data from agent
│ (Security Onion)│
└──────┬──────────┘
       │ Routes to Elasticsearch
       │
┌──────▼───────────┐
│ kismet.common    │  Main pipeline:
│ Ingest Pipeline  │  1. Parse JSON
│                  │  2. Remove seenby field (critical!)
│                  │  3. Extract device type
│                  │  4. Route to sub-pipeline
└──────┬───────────┘
       │
       ├─────► kismet.ap (access points)
       ├─────► kismet.client (devices)
       ├─────► kismet.adhoc (peer-to-peer)
       └─────► (etc)
       │
┌──────▼──────────┐
│ Elasticsearch   │  Stores in logs-kismet-* indices
└──────┬──────────┘
       │
┌──────▼──────────┐
│     Kibana      │  Search, visualize, alert
└─────────────────┘
```

## Appendix B: Field Reference

### Common Kismet Fields

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `@timestamp` | date | When device was last modified | `2026-02-16T22:06:29Z` |
| `event.module` | keyword | Always "kismet" | `kismet` |
| `event.dataset` | keyword | Device type | `kismet.ap`, `kismet.client` |
| `event.category` | keyword | Always "network" | `network` |
| `network.wireless.ssid` | keyword | Network name | `MyWiFi`, `Hidden` |
| `network.wireless.bssid` | keyword | MAC address of AP | `00:11:22:33:44:55` |
| `network.wireless.channel` | keyword | WiFi channel | `6`, `36`, `44` |
| `network.wireless.frequency` | long | Frequency in Hz | `2437000000`, `5220000` |
| `network.wireless.ssid_cloaked` | long | 1 if hidden, 0 if visible | `0`, `1` |
| `client.mac` | keyword | Client device MAC | `AA:BB:CC:DD:EE:FF` |
| `device.manufacturer` | keyword | OUI lookup | `Apple`, `Samsung`, `Unknown` |
| `source.packets` | long | Packets transmitted | `1351` |
| `kismet.alerts.count` | long | Number of Kismet alerts | `0`, `3` |
| `kismet.first_seen` | date | First time seen | `2026-02-16T20:01:04Z` |
| `kismet.last_seen` | date | Last time seen | `2026-02-16T22:06:29Z` |

---

## Appendix C: Useful Kibana Visualizations

### Top SSIDs (Table)

**Visualization Type:** Table  
**Data Source:** `logs-kismet-*`  
**Metric:** Count  
**Rows:** `network.wireless.ssid.keyword` (Top 10)  

**Use Case:** See which networks are most active.

---

### Access Points by Manufacturer (Pie Chart)

**Visualization Type:** Pie  
**Data Source:** `logs-kismet-*`  
**Filter:** `event.dataset:kismet.ap`  
**Metric:** Count  
**Slice by:** `device.manufacturer.keyword`  

**Use Case:** Identify what vendors' equipment is in your environment.

---

### WiFi Activity Timeline (Area Chart)

**Visualization Type:** Area  
**Data Source:** `logs-kismet-*`  
**Metric:** Count  
**Horizontal axis:** `@timestamp` (Date Histogram, 10 minute intervals)  
**Break down by:** `event.dataset.keyword`  

**Use Case:** See when WiFi activity peaks.

---

### Hidden Networks (Metric)

**Visualization Type:** Metric  
**Data Source:** `logs-kismet-*`  
**Filter:** `network.wireless.ssid_cloaked:1`  
**Metric:** Count  

**Use Case:** Count how many cloaked (hidden) networks are detected.

---

## Appendix D: Maintenance Tasks

### Weekly

- **Review Kismet Alerts:**
  - Kibana → Discover → `kismet.alerts.count:>0`
  - Investigate any alerts from Kismet's wireless IDS

### Monthly

- **Check Index Size:**
  ```bash
  sudo so-elasticsearch-query _cat/indices/logs-kismet-*?v
  ```
  - If indices are growing too large (>5GB), consider:
    - Reducing poll frequency (15m or 30m)
    - Shortening retention policy

- **Verify Integration Health:**
  - Fleet → Agents → Check agent status
  - Fleet → Integrations → Verify Kismet integration has no errors

### After Security Onion Updates

- **Verify Pipeline Persisted:**
  ```bash
  # In Kibana Dev Tools
  GET _ingest/pipeline/kismet.common
  ```
  - Confirm the `remove` processor for `seenby` is still present (position #2)
  - If missing, pipeline was overwritten - restore from Part 4, Step 4.2

---

## Appendix E: Uninstalling

### To Remove Kismet Integration

1. **Remove Fleet Integration:**
   - Kibana → Fleet → Agent policies
   - Click on policy with Kismet integration
   - Find "Kismet Wireless Monitoring"
   - Click **⋮** → **Delete integration**

2. **Delete Pipelines (optional):**
   ```bash
   # In Kibana Dev Tools
   DELETE _ingest/pipeline/kismet.*
   ```

3. **Delete Indices (optional):**
   ```bash
   # On Security Onion
   curl -XDELETE "localhost:9200/logs-kismet-*"
   ```

4. **Remove Local Salt File (optional):**
   ```bash
   # On Security Onion
   sudo rm /opt/so/saltstack/local/salt/elasticsearch/files/ingest/kismet.common
   ```

---

## Troubleshooting

### No Data Appearing in Elasticsearch

**Problem:** Count returns 0 or indices don't exist.

**Diagnosis Steps:**

1. **Check if Kismet is running:**
   ```bash
   # On Kismet system
   sudo systemctl status kismet
   ```

2. **Check if agent is running:**
   ```bash
   # On Kismet system
   sudo systemctl status elastic-agent
   ```

3. **Check agent logs for errors:**
   ```bash
   # On Kismet system
   sudo elastic-agent logs | grep -i "error\|failed" | tail -20
   ```

4. **Test Kismet API directly:**
   ```bash
   # On Kismet system
   curl -H "Cookie: KISMET=YOUR_API_KEY" http://localhost:2501/devices/last-time/-600/devices.tjson | head -50
   ```
   **Expected:** Should return JSON data with WiFi devices.

5. **Check Elasticsearch logs for errors:**
   ```bash
   # On Security Onion
   sudo docker logs so-elasticsearch --tail 100 2>&1 | grep -i "kismet\|error"
   ```

**Common Issues:**
- **Kismet not running:** Start Kismet with `sudo systemctl start kismet`
- **Wrong API key:** Regenerate in Kismet web UI and update Fleet integration
- **Agent not enrolled:** Verify in Fleet → Agents that your Kismet system's agent appears and is "Healthy"
- **Network connectivity:** Test `ping` and `telnet` from Kismet system to Security Onion

### Mapper Conflict Errors

**Problem:** Elasticsearch logs show errors like:
```
mapper [kismet.seenby...] cannot be changed from type [long] to [float]
```

**This means:** The `seenby` field wasn't properly removed before data arrived.

**Solution:**

1. **Verify the pipeline has the remove processor:**
   ```bash
   # In Kibana Dev Tools
   GET _ingest/pipeline/kismet.common
   ```
   
   Look for this processor early in the pipeline (position #2):
   ```json
   {
     "remove": {
       "field": "message2.kismet_device_base_seenby",
       "ignore_missing": true
     }
   }
   ```

2. **If the remove processor is missing, add it:**
   - Go back to Part 1, Step 1.2 and recreate the `kismet.common` pipeline
   - Make sure the `remove` processor is position #2 (right after `json`)

3. **Delete the broken indices:**
   ```bash
   # On Security Onion
   curl -XDELETE "localhost:9200/_data_stream/logs-kismet-so"
   curl -XDELETE "localhost:9200/_data_stream/logs-kismet-default"
   ```

4. **Verify deletion:**
   ```bash
   sudo so-elasticsearch-query _cat/indices?v | grep kismet
   ```
   **Expected:** No output (indices deleted).

5. **Wait for next poll (10 minutes) or force one:**
   ```bash
   # On Kismet system
   sudo systemctl restart kismet
   ```
   Wait 30 seconds.

6. **Verify fresh data:**
   ```bash
   # On Security Onion
   sudo so-elasticsearch-query logs-kismet-*/_count
   ```
   **Expected:** Non-zero count.

### Agent Shows Integration But httpjson Not Running

**Problem:** Fleet shows the integration but the agent process list doesn't include httpjson.

**Diagnosis:**

```bash
# On Kismet system
sudo systemctl status elastic-agent
```

Look in the CGroup for a line mentioning `httpjson`.

**If httpjson is missing:**

1. **Restart the agent:**
   ```bash
   sudo systemctl restart elastic-agent
   ```

2. **Wait 1 minute and check logs:**
   ```bash
   sudo elastic-agent logs | grep -i "httpjson" | tail -20
   ```

3. **If still not appearing, check Fleet policy:**
   - In Kibana → Fleet → Agent policies
   - Click on your agent's policy
   - Verify "Kismet Wireless Monitoring" integration is listed
   - If not, the agent is enrolled to the wrong policy

**To fix wrong policy enrollment:**

1. In Kibana → Fleet → Agents
2. Click on your Kismet system's agent
3. Click **"Assign to new policy"**
4. Select the policy that has the Kismet integration
5. Confirm and wait 2 minutes

### Data Flows But Fields Are Missing

**Problem:** Documents appear in Elasticsearch but lack expected fields like `network.wireless.ssid`.

**This means:** The device-specific pipelines might not be working.

**Diagnosis:**

1. **Check one document in detail:**
   ```bash
   # On Security Onion
   sudo so-elasticsearch-query 'logs-kismet-so/_search?size=1'
   ```

2. **Look at the `event.dataset` field:**
   - Should be something like `kismet.ap`, `kismet.client`, etc.
   - If it's just `kismet` (generic), the device type detection failed

3. **Verify all sub-pipelines exist:**
   ```bash
   # In Kibana Dev Tools
   GET _ingest/pipeline/kismet.*
   ```
   **Expected:** All 9 pipelines present.

4. **If pipelines are missing:**
   - Go back to Part 1 and recreate the missing pipeline(s)
   - Delete the indices (see Mapper Conflict section)
   - Wait for fresh data

---

## References

- **Security Onion Documentation:** https://docs.securityonion.net
- **Kismet Documentation:** https://www.kismetwireless.net/docs/
- **Elasticsearch Ingest Pipelines:** https://www.elastic.co/guide/en/elasticsearch/reference/current/ingest.html
- **Elastic Common Schema (ECS):** https://www.elastic.co/guide/en/ecs/current/index.html
- **Fleet and Elastic Agent:** https://www.elastic.co/guide/en/fleet/current/index.html

---
