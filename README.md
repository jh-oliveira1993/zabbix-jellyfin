# Jellyfin Media Server — Zabbix 7.0 HTTP Monitor

> **Production-grade, agentless monitoring template for Jellyfin Media Server using Zabbix 7.0 native HTTP Agent — zero external scripts, zero cron jobs, and 100% native preprocessing.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Zabbix 7.0](https://img.shields.io/badge/Zabbix-7.0%20LTS-red.svg)](https://www.zabbix.com/)
[![Jellyfin 10.9+](https://img.shields.io/badge/Jellyfin-10.9%2B%20%7C%2010.10%2B%20%7C%2010.11%2B-purple.svg)](https://jellyfin.org/)
[![HTTP Agent](https://img.shields.io/badge/Architecture-Native%20HTTP%20Agent-brightgreen.svg)](#3-architecture)

---

## Table of Contents

1. [Overview](#1-overview)
2. [Key Features & Observability Tiers](#2-key-features--observability-tiers)
3. [Compatibility](#3-compatibility)
4. [Architecture](#4-architecture)
5. [Data Point & Metric Catalogue](#5-data-point--metric-catalogue)
   - 5.1 [Tier 1: Health & Public System Information](#51-tier-1-health--public-system-information-no-auth)
   - 5.2 [Tier 2: Prometheus Metrics & .NET Runtime](#52-tier-2-prometheus-metrics--net-runtime)
   - 5.3 [Tier 3: REST API & Library Insights](#53-tier-3-rest-api--library-insights-requires-api-key)
6. [Prerequisites & Jellyfin Preparation](#6-prerequisites--jellyfin-preparation)
   - 6.1 [Enabling the Prometheus Endpoint](#61-enabling-the-prometheus-endpoint)
   - 6.2 [Creating an API Key for Session & Library Metrics](#62-creating-an-api-key-for-session--library-metrics)
7. [Installation — Step by Step](#7-installation--step-by-step)
   - 7.1 [Importing the Template into Zabbix](#71-importing-the-template-into-zabbix)
   - 7.2 [Creating the Host in Zabbix](#72-creating-the-host-in-zabbix)
   - 7.3 [Configuring User Macros](#73-configuring-user-macros)
   - 7.4 [Forcing Configuration Cache Reload](#74-forcing-configuration-cache-reload)
8. [Template Reference](#8-template-reference)
   - 8.1 [User Macros](#81-user-macros)
   - 8.2 [Master Items & Polling Intervals](#82-master-items--polling-intervals)
9. [Triggers & Operational Alerts](#9-triggers--operational-alerts)
10. [Troubleshooting](#10-troubleshooting)
11. [Known Considerations](#11-known-considerations)
12. [License](#12-license)

---

## 1. Overview

Jellyfin is a popular open-source media system that powers home streaming of movies, TV shows, music, and live TV. While simple uptime checks can tell if the web UI is alive, production homelabs and media servers require deep visibility into:
- .NET runtime health (Garbage Collector heap sizes, pause ratios, ThreadPool saturation).
- Web server throughput (Kestrel active connections, queue length, HTTP error rates).
- Playback activity (active streams, direct play vs. CPU/GPU-intensive transcoding).
- Library growth (tracking inventory of movies, series, episodes, and audio tracks).

Traditional monitoring approaches often rely on external shell/Python scripts, custom Docker sidecars, or cron jobs that invoke `zabbix_sender`. While functional, those methods introduce extra dependencies, maintenance overhead, and failure points.

This project implements a **100% native Zabbix 7.0 HTTP Agent template** (`Jellyfin by HTTP`). The Zabbix Server or Zabbix Proxy directly queries Jellyfin's HTTP endpoints and processes the data using native preprocessing:
- **`PROMETHEUS_PATTERN`** to ingest and parse Prometheus exposition metrics from `/metrics`.
- **`JSONPATH`** and **`JAVASCRIPT`** to extract structured metadata, stream stats, and counts.
- **`REGEX`** for health endpoint validation.

---

## 2. Key Features & Observability Tiers

The template is organized into **three distinct tiers** inside a single YAML file:

```
┌────────────────────────────────────────────────────────────────────────┐
│                        JELLYFIN BY HTTP TEMPLATE                       │
├────────────────────────────────────────────────────────────────────────┤
│  TIER 1: Base Health & Version (Zero Auth Required)                    │
│  - /health              -> Instant Healthy/Unhealthy check             │
│  - /System/Info/Public  -> Version, Server Name, ID, Product Name      │
├────────────────────────────────────────────────────────────────────────┤
│  TIER 2: Prometheus & .NET Runtime (Zero Auth, Metrics Enabled)        │
│  - /metrics             -> CPU, Memory working set, GC Gens 0/1/2/LOH, │
│                            GC pause ratio & fragmentation, ThreadPool, │
│                            Kestrel connections, request rates, sockets │
├────────────────────────────────────────────────────────────────────────┤
│  TIER 3: REST API & Library Insights (Requires API Key)                │
│  - /Sessions            -> Active sessions, playing, transcoding,      │
│                            direct play streams                         │
│  - /Items/Counts        -> Movies, Series, Episodes, Songs, Albums     │
└────────────────────────────────────────────────────────────────────────┘
```

- **Zero External Dependencies:** No Python, no curl wrappers, no custom cron jobs.
- **Low Overhead via Dependent Items:** Only 6 master HTTP queries are executed; all 43 telemetry metrics are derived efficiently in Zabbix memory without extra network overhead.
- **Strict Zabbix 7.0 Compatibility:** Built and validated on Zabbix 7.0 LTS with compliant UUIDv4 identifiers and modern trigger expressions.

---

## 3. Compatibility

| Component | Supported Versions | Notes |
|---|---|---|
| **Zabbix Server / Proxy** | **7.0 LTS+** | Tested on 7.0.22. Uses Zabbix 7.0 YAML export syntax and UUIDv4 format. |
| **Jellyfin Media Server** | **10.9.x, 10.10.x, 10.11.x** | Tested on 10.11.11 (Linux x86_64 Docker container). |
| **Collector Architecture** | Zabbix Server or Zabbix Proxy | Works identically when polled directly by Zabbix Server or via local Zabbix Proxies. |

---

## 4. Architecture

```
┌───────────────────────────────────────────────────────────────────────────┐
│                           HOMELAB / LOCAL LAN                             │
│                                                                           │
│   ┌────────────────────────────────┐                                      │
│   │     Jellyfin Media Server      │                                      │
│   │     Docker Container (:8096)   │                                      │
│   │                                │                                      │
│   │  ├─ /health                    │                                      │
│   │  ├─ /System/Info/Public        │                                      │
│   │  ├─ /metrics                   │                                      │
│   │  ├─ /Sessions                  │                                      │
│   │  └─ /Items/Counts              │                                      │
│   └────────────────▲───────────────┘                                      │
│                    │                                                      │
│                    │ Native HTTP Agent GET                                │
│                    │ (LAN IP: e.g. 192.168.1.100:8096)                    │
│                    │                                                      │
│   ┌────────────────┴───────────────┐                                      │
│   │       Zabbix Proxy 7.0         │                                      │
│   │       (Docker / SQLite3)       │                                      │
│   │   zbx-proxy-local  │                                      │
│   └────────────────┬───────────────┘                                      │
└────────────────────┼──────────────────────────────────────────────────────┘
                     │
                     │ Encrypted TLS-PSK Tunnel
                     ▼
      ┌─────────────────────────────┐
      │   Zabbix Server 7.0 LTS     │
      │   (Oracle Cloud / VPS)      │
      │   Web Frontend & Alerting   │
      └─────────────────────────────┘
```

---

## 5. Data Point & Metric Catalogue

The template defines **49 items** (6 Master HTTP Agents + 43 Dependent Items).

### 5.1 Tier 1: Health & Public System Information (No Auth)

| Item Name | Key | Type | Preprocessing | Description |
|---|---|---|---|---|
| **Raw Health Check Response** | `jellyfin.health.raw` | HTTP Agent | None | Raw response from `GET {$JELLYFIN.URL}/health`. |
| **Server Health Status** | `jellyfin.health` | Dependent | `REGEX: ^(\S+) -> ` | Status string (`Healthy`, `Unhealthy`). |
| **Raw System Info (Public)** | `jellyfin.sysinfo.raw` | HTTP Agent | None | Raw JSON from `GET {$JELLYFIN.URL}/System/Info/Public`. |
| **Server Version** | `jellyfin.version` | Dependent | `JSONPATH: $.Version` | Running Jellyfin server version (e.g., `10.11.11`). |
| **Server Name** | `jellyfin.server.name` | Dependent | `JSONPATH: $.ServerName` | Configured Jellyfin hostname/server name. |
| **Server ID** | `jellyfin.server.id` | Dependent | `JSONPATH: $.Id` | Unique instance GUID. |
| **Product Name** | `jellyfin.product` | Dependent | `JSONPATH: $.ProductName` | Product identifier (e.g., `Jellyfin Server`). |

---

### 5.2 Tier 2: Prometheus Metrics & .NET Runtime

Scraped from `GET {$JELLYFIN.URL}/metrics` via master item `jellyfin.metrics.raw`.

#### Process & CPU
| Item Name | Key | Unit | Prometheus Metric |
|---|---|---|---|
| **Process Working Set Memory** | `jellyfin.process.working_set` | B | `process_working_set_bytes` |
| **Process CPU Usage** | `jellyfin.process.cpu_usage` | % | `system_runtime_cpu_usage` |
| **Process CPU Time Total** | `jellyfin.process.cpu_seconds_total` | s | `process_cpu_seconds_total` |
| **Available CPU Cores** | `jellyfin.process.cpu_count` | - | `process_cpu_count` |

#### .NET Garbage Collector (GC)
| Item Name | Key | Unit | Prometheus Metric |
|---|---|---|---|
| **GC Heap Size Gen 0** | `jellyfin.gc.heap_gen0` | B | `dotnet_gc_heap_size_bytes{gc_generation="0"}` |
| **GC Heap Size Gen 1** | `jellyfin.gc.heap_gen1` | B | `dotnet_gc_heap_size_bytes{gc_generation="1"}` |
| **GC Heap Size Gen 2** | `jellyfin.gc.heap_gen2` | B | `dotnet_gc_heap_size_bytes{gc_generation="2"}` |
| **GC Large Object Heap Size** | `jellyfin.gc.heap_loh` | B | `dotnet_gc_heap_size_bytes{gc_generation="loh"}` |
| **GC Collections Gen 0** | `jellyfin.gc.collections_gen0` | - | `dotnet_gc_collection_count_total{gc_generation="0"}` |
| **GC Collections Gen 1** | `jellyfin.gc.collections_gen1` | - | `dotnet_gc_collection_count_total{gc_generation="1"}` |
| **GC Collections Gen 2** | `jellyfin.gc.collections_gen2` | - | `dotnet_gc_collection_count_total{gc_generation="2"}` |
| **GC Pause Ratio** | `jellyfin.gc.pause_ratio` | % | `dotnet_gc_pause_ratio` |
| **GC Fragmentation** | `jellyfin.gc.fragmentation` | % | `system_runtime_gc_fragmentation` |

#### ThreadPool & Concurrency
| Item Name | Key | Unit | Prometheus Metric |
|---|---|---|---|
| **ThreadPool Active Threads** | `jellyfin.threadpool.threads` | - | `dotnet_threadpool_num_threads` |
| **ThreadPool Queue Length** | `jellyfin.threadpool.queue_length` | - | `dotnet_threadpool_queue_length_sum` |
| **ThreadPool Completed Work Items**| `jellyfin.threadpool.completed_total` | - | `dotnet_threadpool_throughput_total` |

#### Web Server (ASP.NET Core & Kestrel)
| Item Name | Key | Unit | Prometheus Metric |
|---|---|---|---|
| **HTTP Active Requests** | `jellyfin.http.requests_active` | - | `microsoft_aspnetcore_hosting_current_requests` |
| **HTTP Failed Requests Total** | `jellyfin.http.requests_failed` | - | `microsoft_aspnetcore_hosting_failed_requests` |
| **HTTP Total Requests** | `jellyfin.http.requests_total` | - | `microsoft_aspnetcore_hosting_total_requests` |
| **HTTP Requests per Second** | `jellyfin.http.requests_per_second` | - | `microsoft_aspnetcore_hosting_requests_per_second_total` |
| **Kestrel Active Connections** | `jellyfin.kestrel.connections_active` | - | `microsoft_aspnetcore_server_kestrel_current_connections` |
| **Kestrel Total Connections** | `jellyfin.kestrel.connections_total` | - | `microsoft_aspnetcore_server_kestrel_total_connections` |
| **Kestrel Connection Rate** | `jellyfin.kestrel.connections_per_second` | - | `microsoft_aspnetcore_server_kestrel_connections_per_second_total` |
| **Kestrel Request Queue Length** | `jellyfin.kestrel.request_queue_length` | - | `microsoft_aspnetcore_server_kestrel_request_queue_length` |

#### Sockets & Network Throughput
| Item Name | Key | Unit | Prometheus Metric |
|---|---|---|---|
| **Sockets Incoming Connections Total** | `jellyfin.sockets.incoming_total` | - | `dotnet_sockets_connections_established_incoming_total` |
| **Sockets Outgoing Connections Total** | `jellyfin.sockets.outgoing_total` | - | `dotnet_sockets_connections_established_outgoing_total` |
| **Network Bytes Received** | `jellyfin.sockets.bytes_received` | B | `system_net_sockets_bytes_received` |
| **Network Bytes Sent** | `jellyfin.sockets.bytes_sent` | B | `system_net_sockets_bytes_sent` |

---

### 5.3 Tier 3: REST API & Library Insights (Requires API Key)

#### Sessions & Active Playback
Scraped from `GET {$JELLYFIN.URL}/Sessions` and `/Sessions?activeWithinSeconds=30`:

| Item Name | Key | Preprocessing | Description |
|---|---|---|---|
| **Raw Sessions Data** | `jellyfin.sessions.raw` | None | Raw session list JSON. |
| **Active Sessions Total** | `jellyfin.sessions.count` | `JAVASCRIPT: return JSON.parse(value).length;` | Total connected client sessions. |
| **Raw Active Playing Sessions** | `jellyfin.sessions.playing.raw` | None | Raw playing sessions JSON. |
| **Sessions Currently Playing** | `jellyfin.sessions.playing` | `JAVASCRIPT` (filter `NowPlayingItem != null`) | Sessions actively rendering audio/video. |
| **Sessions Transcoding** | `jellyfin.sessions.transcode` | `JAVASCRIPT` (filter `PlayMethod === "Transcode"`) | Streams requiring server-side transcoding. |
| **Sessions Direct Playing** | `jellyfin.sessions.direct_play` | `JAVASCRIPT` (filter `PlayMethod === "DirectPlay"`) | Streams playing directly without transcode. |

#### Media Library Inventory
Scraped every 6 hours from `GET {$JELLYFIN.URL}/Items/Counts`:

| Item Name | Key | Preprocessing | Description |
|---|---|---|---|
| **Raw Library Item Counts** | `jellyfin.library.counts.raw` | None | Raw JSON library summary. |
| **Library Movie Count** | `jellyfin.library.movies` | `JSONPATH: $.MovieCount` | Total movies catalogued. |
| **Library Series Count** | `jellyfin.library.series` | `JSONPATH: $.SeriesCount` | Total TV series catalogued. |
| **Library Episode Count** | `jellyfin.library.episodes` | `JSONPATH: $.EpisodeCount` | Total TV episodes catalogued. |
| **Library Song Count** | `jellyfin.library.songs` | `JSONPATH: $.SongCount` | Total music tracks catalogued. |
| **Library Album Count** | `jellyfin.library.albums` | `JSONPATH: $.AlbumCount` | Total music albums catalogued. |
| **Library Music Video Count**| `jellyfin.library.music_videos` | `JSONPATH: $.MusicVideoCount` | Total music videos catalogued. |

---

## 6. Prerequisites & Jellyfin Preparation

### 6.1 Enabling the Prometheus Endpoint

By default, Jellyfin disables Prometheus metrics. To enable them:

1. Stop your Jellyfin container or service.
2. Edit `config/system.xml` (located in your Jellyfin data directory, e.g. `/DELLHD/jellyfin/config/config/system.xml`).
3. Locate the `<EnableMetrics>` tag (near line 5) and set it to `true`:
   ```xml
   <EnableMetrics>true</EnableMetrics>
   ```
4. Restart Jellyfin:
   ```bash
   docker restart jellyfin
   ```
5. Verify the endpoint responds with metrics:
   ```bash
   curl -s http://<JELLYFIN_IP>:8096/metrics | head -n 10
   ```

### 6.2 Creating an API Key for Session & Library Metrics

To allow Zabbix to read sessions and library item counts:

1. Open the Jellyfin Web UI as an Administrator.
2. Navigate to **Dashboard** → **Advanced** → **API Keys**.
3. Click **+** (Add API Key), enter the name `Zabbix Monitoring`, and click **OK**.
4. Copy the generated Access Token (a 32-character hexadecimal string).

---

## 7. Installation — Step by Step

### 7.1 Importing the Template into Zabbix

1. In Zabbix Web UI, navigate to **Data collection** → **Templates**.
2. Click **Import** (top right).
3. Choose the file [`zabbix/zabbix_template_jellyfin_by_http.yaml`](zabbix/zabbix_template_jellyfin_by_http.yaml).
4. Ensure the import rules for **Templates**, **Template groups**, **Items**, and **Triggers** have **Create missing** and **Update existing** checked.
5. Click **Import**.

### 7.2 Creating the Host in Zabbix

1. Navigate to **Data collection** → **Hosts** → **Create host**.
2. Fill in the host settings:
   - **Host name:** `Jellyfin Media Server` (or descriptive name).
   - **Templates:** Link `Jellyfin by HTTP`.
   - **Host groups:** Select `Applications` (or your preferred group).
   - **Monitored by:** Select **Proxy** (if monitoring through a local Zabbix Proxy) or **Server**.
   - **Proxy:** Select your local proxy (e.g. `zbx-proxy-local`).
   - **Interfaces:** Add an **Agent** interface with the LAN IP of your host (e.g. `192.168.1.100`, port `8096`).
3. Click **Add**.

### 7.3 Configuring User Macros

On the newly created host, switch to the **Macros** tab and configure:

| Macro | Value | Description |
|---|---|---|
| `{$JELLYFIN.URL}` | `http://192.168.1.100:8096` | Base URL to your Jellyfin instance (no trailing slash). |
| `{$JELLYFIN.API.KEY}` | `YOUR_JELLYFIN_API_KEY_HERE` | API Key generated in §6.2. |

### 7.4 Forcing Configuration Cache Reload

If monitoring via a Zabbix Proxy, reload the proxy configuration cache to start polling immediately without waiting for the default config frequency:

```bash
# On the Docker host running the Zabbix Proxy:
docker exec zbx-proxy zabbix_proxy -R config_cache_reload
```

---

## 8. Template Reference

### 8.1 User Macros

| Macro | Default Value | Description |
|---|---|---|
| `{$JELLYFIN.URL}` | `http://localhost:8096` | Base HTTP URL to Jellyfin. Must be overridden on the host level. |
| `{$JELLYFIN.API.KEY}` | *(empty)* | Admin API Key for Tier 3 items. If omitted, Tier 3 items will stay unsupported. |
| `{$JELLYFIN.HEALTH.INTERVAL}` | `1m` | Polling frequency for health check and runtime metrics. |
| `{$JELLYFIN.LIBRARY.INTERVAL}` | `6h` | Polling frequency for library item counts. |
| `{$JELLYFIN.MEM.WARN}` | `1073741824` (1 GiB) | Working set memory WARNING threshold in bytes. |
| `{$JELLYFIN.MEM.HIGH}` | `2147483648` (2 GiB) | Working set memory HIGH threshold in bytes. |
| `{$JELLYFIN.CPU.WARN}` | `70` (%) | CPU usage WARNING threshold. |
| `{$JELLYFIN.CPU.HIGH}` | `90` (%) | CPU usage HIGH threshold. |
| `{$JELLYFIN.SESSIONS.WARN}` | `10` | Active session count threshold for INFO notification. |
| `{$JELLYFIN.TRANSCODE.WARN}`| `3` | Active transcode streams threshold for WARNING notification. |

### 8.2 Master Items & Polling Intervals

| Master Item Key | Endpoint | Interval | Purpose |
|---|---|---|---|
| `jellyfin.health.raw` | `{$JELLYFIN.URL}/health` | `1m` | Health status. |
| `jellyfin.sysinfo.raw` | `{$JELLYFIN.URL}/System/Info/Public` | `15m` | Version and system attributes. |
| `jellyfin.metrics.raw` | `{$JELLYFIN.URL}/metrics` | `1m` | Prometheus performance metrics. |
| `jellyfin.sessions.raw` | `{$JELLYFIN.URL}/Sessions` | `1m` | Active sessions overview. |
| `jellyfin.sessions.playing.raw` | `{$JELLYFIN.URL}/Sessions?activeWithinSeconds=30` | `1m` | Active playback sessions and transcode detection. |
| `jellyfin.library.counts.raw` | `{$JELLYFIN.URL}/Items/Counts` | `6h` | Media collection counts. |

---

## 9. Triggers & Operational Alerts

| Severity | Trigger Name | Expression | Operational Guidance |
|---|---|---|---|
| **DISASTER** | **Jellyfin Server Unhealthy** | `last(/Jellyfin by HTTP/jellyfin.health)<>"Healthy"` | The `/health` endpoint returned a non-Healthy response. Check if the Jellyfin container crashed, ran out of memory, or if storage is unmounted. |
| **HIGH** | **Jellyfin Server Memory Usage is HIGH** | `min(/Jellyfin by HTTP/jellyfin.process.working_set,5m)>{$JELLYFIN.MEM.HIGH}` | Working set exceeded 2 GiB for 5 minutes. Check for memory leaks or high library scan activity. |
| **WARNING** | **Jellyfin Server Memory Usage is WARNING** | `min(/Jellyfin by HTTP/jellyfin.process.working_set,5m)>{$JELLYFIN.MEM.WARN}` | Working set exceeded 1 GiB for 5 minutes (suppressed if HIGH fires). |
| **HIGH** | **Jellyfin Server CPU Usage is HIGH** | `min(/Jellyfin by HTTP/jellyfin.process.cpu_usage,5m)>{$JELLYFIN.CPU.HIGH}` | Process CPU consumption sustained above 90% for 5 min. Usually caused by software video transcoding. |
| **WARNING** | **Jellyfin Server CPU Usage is WARNING** | `min(/Jellyfin by HTTP/jellyfin.process.cpu_usage,5m)>{$JELLYFIN.CPU.WARN}` | Process CPU consumption sustained above 70% for 5 min (suppressed if HIGH fires). |
| **WARNING** | **Jellyfin High Rate of HTTP Failed Requests** | `change(/Jellyfin by HTTP/jellyfin.http.requests_failed)>10` | More than 10 failed HTTP requests (5xx) occurred within the last polling interval. |
| **WARNING** | **Jellyfin ThreadPool Queue Backlog** | `min(/Jellyfin by HTTP/jellyfin.threadpool.queue_length,5m)>50` | .NET ThreadPool has >50 work items queued. The CPU is saturated with background tasks. |
| **WARNING** | **Jellyfin Kestrel Request Queue Backlog** | `min(/Jellyfin by HTTP/jellyfin.kestrel.request_queue_length,3m)>10` | Incoming HTTP requests are queuing in Kestrel, indicating delayed request handling. |
| **WARNING** | **High concurrent transcode load** | `min(/Jellyfin by HTTP/jellyfin.sessions.transcode,5m)>{$JELLYFIN.TRANSCODE.WARN}` | More than 3 concurrent transcode sessions active. Verify if hardware acceleration (VA-API / NVENC / QuickSync) is working properly. |
| **INFO** | **Jellyfin version changed to {ITEM.VALUE}** | `last(/Jellyfin by HTTP/jellyfin.version)<>last(/Jellyfin by HTTP/jellyfin.version,#2)` | Jellyfin was updated to a new version. |
| **INFO** | **High active session count** | `min(/Jellyfin by HTTP/jellyfin.sessions.count,5m)>{$JELLYFIN.SESSIONS.WARN}` | More than 10 client sessions are connected simultaneously. |
| **INFO** | **Library movie count changed to {ITEM.VALUE}** | `last(/Jellyfin by HTTP/jellyfin.library.movies)<>last(#2)` | A movie was added or removed from the library. |
| **INFO** | **Library series count changed to {ITEM.VALUE}** | `last(/Jellyfin by HTTP/jellyfin.library.series)<>last(#2)` | A TV series was added or removed from the library. |

---

## 10. Troubleshooting

### Problem: `Cannot connect to [[192.168.x.x]:8096]: connection timed out` from Zabbix Proxy
- **Cause:** When Zabbix Proxy runs in a Docker bridge container on the same host as Jellyfin, connecting to the host's LAN IP may fail if Docker hairpinning / `br_netfilter` drops outbound-to-host packets.
- **Solution:** 
  1. Verify the current IP address of your host machine (`ip addr show`). If the host obtained a new DHCP lease, update `{$JELLYFIN.URL}`.
  2. Alternatively, use the Docker bridge gateway IP (e.g. `http://172.17.0.1:8096`) or attach `zbx-proxy` and `jellyfin` to the same Docker network and use `http://jellyfin:8096`.

### Problem: `jellyfin.metrics.raw` returns 404 or empty data
- **Cause:** Prometheus metrics are not enabled in Jellyfin.
- **Solution:** Follow [§6.1](#61-enabling-the-prometheus-endpoint) to set `<EnableMetrics>true</EnableMetrics>` in `system.xml` and restart the container.

### Problem: Tier 3 items return `401 Unauthorized`
- **Cause:** The `{$JELLYFIN.API.KEY}` macro is empty or invalid.
- **Solution:** Create an API Key in Jellyfin Dashboard → API Keys and paste it into the host's `{$JELLYFIN.API.KEY}` macro.

---

## 11. Known Considerations

- **Library Item Polling:** The `/Items/Counts` endpoint traverses library indices. The default polling interval is set to **6 hours** (`6h`) to minimize unnecessary disk and database overhead on large libraries.
- **Transcode vs. Direct Play:** The session parser distinguishes between `DirectPlay` and `Transcode` streams. Direct streaming uses negligible CPU, whereas software transcoding uses 100% of allocated cores. If you observe high CPU triggers alongside transcode alerts, configure GPU passthrough (QuickSync, VA-API, or NVENC).

---

## 12. License

This project is licensed under the [MIT License](LICENSE).
Copyright (c) 2026 José Henrique.
