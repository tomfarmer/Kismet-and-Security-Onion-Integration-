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

## References

- [Security Onion Documentation](https://docs.securityonion.net)
- [Kismet Documentation](https://www.kismetwireless.net/docs/)
- [Elasticsearch Ingest Pipelines](https://www.elastic.co/guide/en/elasticsearch/reference/current/ingest.html)
- [Elastic Common Schema (ECS)](https://www.elastic.co/guide/en/ecs/current/index.html)
- [Fleet and Elastic Agent](https://www.elastic.co/guide/en/fleet/current/index.html)
