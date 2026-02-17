# Kismet + Security Onion 2.4 Integration

> Integrate Kismet wireless monitoring with Security Onion 2.4 to make WiFi device data searchable in Kibana via Elasticsearch.

---

## What This Does

- Polls Kismet every **60 seconds** for WiFi device data
- Transforms Kismet JSON into **ECS (Elastic Common Schema)** format
- Stores wireless network metadata in **Elasticsearch**
- Makes WiFi monitoring data searchable and visual in **Kibana**

**Time Required:** 30–45 minutes  
**Skill Level:** Intermediate

---

## Prerequisites

| Requirement | Notes |
|---|---|
| Security Onion 2.4 | Installed and running |
| Kismet | Installed with a wireless adapter in monitor mode |
| Network connectivity | Between Security Onion and Kismet system |
| SSH access | To both Security Onion and Kismet system |
| Kibana credentials | Web interface access |
| Kismet API key | Readonly permission |

---

## Architecture

```
┌─────────────┐
│   Kismet    │  Captures WiFi packets
└──────┬──────┘
       │ REST API (every 60s)
┌──────▼──────────┐
│ Elastic Agent   │  HTTPJson input polls Kismet
└──────┬──────────┘
       │ Sends JSON via Logstash protocol
┌──────▼──────────┐
│    Logstash     │  Receives data, routes to Elasticsearch
└──────┬──────────┘
┌──────▼───────────┐
│ kismet.common    │  Parses JSON → removes seenby field
│ Ingest Pipeline  │  → extracts device type → routes to sub-pipeline
└──────┬───────────┘
       ├──► kismet.ap       (access points)
       ├──► kismet.client   (client devices)
       ├──► kismet.ad_hoc   (peer-to-peer)
       └──► kismet.wds / kismet.wds_ap / etc.
┌──────▼──────────┐
│ Elasticsearch   │  Stores in logs-kismet-* indices
└──────┬──────────┘
┌──────▼──────────┐
│     Kibana      │  Search, visualize, alert
└─────────────────┘
```

---

## Setup Guide

### Step 1 — Install Elastic Agent on the Kismet Server

1. Log into Security Onion and navigate to **Downloads**
2. Download the appropriate **Elastic Agent** installer (likely Linux)
3. Allow the agent through the firewall: **Administration → Configure → Allow Elastic Agent endpoints to send logs**
4. Enter the IP or IP range of the agent and **Synchronize Grid**
5. SCP the installer to the Kismet server, then install:

```bash
chmod +x so-elastic-agent<tab>
sudo ./so-elastic-agent<tab> install
```

6. Ensure the Kismet server can resolve Security Onion via DNS (e.g., `SecurityOnion.yourdomain.local`)
7. Verify the agent checked in: **Security Onion → Elastic Fleet**

---

### Step 2 — Create Elasticsearch Ingest Pipelines

SSH into Security Onion and copy the pipeline files from default to local:

```bash
# Ingest pipelines (9 files)
sudo cp /opt/so/saltstack/default/salt/elasticsearch/files/ingest/kismet* \
    /opt/so/saltstack/local/salt/elasticsearch/files/ingest/

# Template
sudo cp /opt/so/saltstack/default/salt/elasticsearch/templates/component/ecs/kismet* \
    /opt/so/saltstack/local/salt/elasticsearch/templates/component/ecs/

# Integration
sudo cp /opt/so/saltstack/default/salt/elasticfleet/files/integrations-optional/kismet* \
    /opt/so/saltstack/local/salt/elasticfleet/files/integrations-optional/
```

#### Fix the `kismet.common` Pipeline

The default `kismet.common` has a schema mismatch where `kismet_common_rrd_day_vec` fluctuates between `long` and `float` types, violating Elasticsearch's strict mapping rules. Replace it with the corrected version:

```bash
sudo truncate -s 0 /opt/so/saltstack/local/salt/elasticsearch/files/ingest/kismet.common
sudo vi /opt/so/saltstack/local/salt/elasticsearch/files/ingest/kismet.common
```

<details>
<summary><strong>Click to expand — corrected kismet.common contents</strong></summary>

```json
{
  "processors": [
    { "json": { "field": "message", "target_field": "message2" } },
    { "remove": { "field": "message2.kismet_device_base_seenby", "ignore_missing": true } },
    { "date": { "field": "message2.kismet_device_base_mod_time", "formats": ["epoch_second"], "target_field": "@timestamp" } },
    { "set": { "field": "event.category", "value": "network" } },
    { "dissect": { "field": "message2.kismet_device_base_type", "pattern": "%{wifi} %{device_type}" } },
    { "lowercase": { "field": "device_type" } },
    { "set": { "field": "event.dataset", "value": "kismet.{{device_type}}" } },
    { "set": { "field": "event.dataset", "value": "kismet.wds_ap", "if": "ctx?.device_type == 'wds ap'" } },
    { "set": { "field": "event.dataset", "value": "kismet.ad_hoc", "if": "ctx?.device_type == 'ad-hoc'" } },
    { "set": { "field": "event.module", "value": "kismet" } },
    { "rename": { "field": "message2.kismet_device_base_packets_tx_total", "target_field": "source.packets" } },
    { "rename": { "field": "message2.kismet_device_base_num_alerts", "target_field": "kismet.alerts.count" } },
    { "rename": { "field": "message2.kismet_device_base_channel", "target_field": "network.wireless.channel", "if": "ctx?.message2?.kismet_device_base_channel != ''" } },
    { "rename": { "field": "message2.kismet_device_base_frequency", "target_field": "network.wireless.frequency", "if": "ctx?.message2?.kismet_device_base_frequency != 0" } },
    { "rename": { "field": "message2.kismet_device_base_last_time", "target_field": "kismet.last_seen" } },
    { "date": { "field": "kismet.last_seen", "formats": ["epoch_second"], "target_field": "kismet.last_seen" } },
    { "rename": { "field": "message2.kismet_device_base_first_time", "target_field": "kismet.first_seen" } },
    { "date": { "field": "kismet.first_seen", "formats": ["epoch_second"], "target_field": "kismet.first_seen" } },
    { "rename": { "field": "message2.kismet_device_base_manuf", "target_field": "device.manufacturer" } },
    { "pipeline": { "name": "{{event.dataset}}" } },
    { "remove": { "field": ["message2", "message", "device_type", "wifi", "agent", "host", "event.created"], "ignore_failure": true } }
  ]
}
```

</details>

Then apply the update live in **Kibana → Dev Tools** using `PUT _ingest/pipeline/kismet.common` with the same JSON content above.

Verify all 9 pipelines are present:

```
GET _ingest/pipeline/kismet.*
```

Expected pipelines: `kismet.common`, `kismet.seenby`, `kismet.ap`, `kismet.client`, `kismet.ad_hoc`, `kismet.bridged`, `kismet.device`, `kismet.wds`, `kismet.wds_ap`

---

### Step 3 — Configure Fleet Integration

#### Get Your Kismet API Key

In the Kismet web UI (`http://KISMET-IP:2501`): **Settings → API Keys → Create** with **readonly** role.

#### Create a Kismet Agent Policy

1. **Fleet → Agent Policies** → find your Kismet server's current policy → **Duplicate**
2. Name it `endpoints-Kismet1` (increment for additional sensors)
3. Open the new policy → **Add Integration** → search **Custom API** → **Add Custom API**

#### Custom API Integration Settings

| Field | Value |
|---|---|
| Integration name | `Kismet Wireless Monitoring` |
| Namespace | `so` |
| Dataset name | `kismet` |
| Ingest Pipeline | `kismet.common` |
| Request URL | `http://localhost:2501/devices/last-time/-600/devices.tjson` |
| Request Interval | `60s` |
| Request HTTP Method | `GET` |

**Request Transform** (sets auth header — keep the single quotes):

```yaml
- set:
    target: header.Cookie
    value: 'KISMET=YOUR_API_KEY_HERE'
```

#### Assign the Policy to Your Kismet Server

**Fleet → Agents** → select your Kismet server → **Actions → Assign new agent policy** → select `endpoints-Kismet1`

---

## Verification

**Check agent logs on the Kismet system:**

```bash
sudo elastic-agent logs | grep -i "httpjson" | tail -5
```

**Check document count in Elasticsearch:**

```bash
sudo so-elasticsearch-query logs-kismet-*/_count
```

Expected: `{"count": <non-zero value>}`

**View data in Kibana:** Discover → index pattern `logs-kismet-*`

**Use the built-in dashboard:** Security Onion → **Dashboards → Kismet - WiFi Devices**

---

## Useful Kibana Searches

| Search | Query |
|---|---|
| All visible SSIDs | `network.wireless.ssid:*` |
| Access points only | `event.dataset:kismet.ap` |
| Client devices only | `event.dataset:kismet.client` |
| Hidden networks | `network.wireless.ssid:"Hidden"` |
| 5GHz networks | `network.wireless.frequency:>=5000000` |
| Networks with alerts | `kismet.alerts.count:>0` |

---

## Key Fields Reference

| Field | Type | Description |
|---|---|---|
| `@timestamp` | date | When device was last modified |
| `event.dataset` | keyword | Device type (`kismet.ap`, `kismet.client`, etc.) |
| `network.wireless.ssid` | keyword | Network name |
| `network.wireless.bssid` | keyword | MAC address of AP |
| `network.wireless.channel` | keyword | WiFi channel |
| `network.wireless.frequency` | long | Frequency in Hz |
| `network.wireless.ssid_cloaked` | long | `1` if hidden, `0` if visible |
| `client.mac` | keyword | Client device MAC address |
| `device.manufacturer` | keyword | OUI lookup result |
| `source.packets` | long | Packets transmitted |
| `kismet.alerts.count` | long | Number of Kismet IDS alerts |
| `kismet.first_seen` | date | First time device was seen |
| `kismet.last_seen` | date | Last time device was seen |

---

## Troubleshooting

<details>
<summary><strong>No data appearing in Elasticsearch</strong></summary>

1. Verify Kismet is running: `sudo systemctl status kismet`
2. Verify the agent is running: `sudo systemctl status elastic-agent`
3. Check agent logs: `sudo elastic-agent logs | grep -i "error\|failed" | tail -20`
4. Test the Kismet API directly:
   ```bash
   curl -H "Cookie: KISMET=YOUR_API_KEY" http://localhost:2501/devices/last-time/-600/devices.tjson | head -50
   ```
5. Check Elasticsearch logs: `sudo docker logs so-elasticsearch --tail 100 2>&1 | grep -i "kismet\|error"`

</details>

<details>
<summary><strong>Mapper conflict errors (long vs float type mismatch)</strong></summary>

This means the `seenby` field wasn't removed before data arrived. Verify the pipeline has the remove processor at position #2:

```
GET _ingest/pipeline/kismet.common
```

If missing, recreate the pipeline. Then delete the broken indices:

```bash
curl -XDELETE "localhost:9200/_data_stream/logs-kismet-so"
curl -XDELETE "localhost:9200/_data_stream/logs-kismet-default"
```

Wait for the next poll or restart Kismet to force one.

</details>

<details>
<summary><strong>Agent shows integration but httpjson isn't running</strong></summary>

```bash
sudo systemctl restart elastic-agent
sudo elastic-agent logs | grep -i "httpjson" | tail -20
```

If still not appearing, verify the agent is enrolled to the correct policy in **Fleet → Agents**.

</details>

<details>
<summary><strong>Data appears but fields are missing</strong></summary>

Check that `event.dataset` is a specific value like `kismet.ap` rather than generic `kismet`. If device type detection failed, verify all 9 sub-pipelines exist:

```
GET _ingest/pipeline/kismet.*
```

Recreate any missing pipelines, delete the indices, and wait for fresh data.

</details>

---

## Maintenance

**Weekly:** Review `kismet.alerts.count:>0` in Kibana Discover for wireless IDS alerts.

**Monthly:**
- Check index size: `sudo so-elasticsearch-query _cat/indices/logs-kismet-*?v`
- If indices exceed ~5GB, reduce poll frequency to `15m` or `30m` in the Fleet integration
- Verify integration health in Fleet → Agents

**After Security Onion updates:** Confirm the `kismet.common` pipeline still has the `seenby` remove processor at position #2.

---

## Uninstalling

1. **Fleet → Agent Policies** → find the Kismet integration → delete it
2. Optionally delete pipelines in Kibana Dev Tools: `DELETE _ingest/pipeline/kismet.*`
3. Optionally delete indices on Security Onion: `curl -XDELETE "localhost:9200/logs-kismet-*"`
4. Optionally remove the local salt file: `sudo rm /opt/so/saltstack/local/salt/elasticsearch/files/ingest/kismet.common`

---

## References

- [Security Onion Documentation](https://docs.securityonion.net)
- [Kismet Documentation](https://www.kismetwireless.net/docs/)
- [Elasticsearch Ingest Pipelines](https://www.elastic.co/guide/en/elasticsearch/reference/current/ingest.html)
- [Elastic Common Schema (ECS)](https://www.elastic.co/guide/en/ecs/current/index.html)
- [Fleet and Elastic Agent](https://www.elastic.co/guide/en/fleet/current/index.html)
