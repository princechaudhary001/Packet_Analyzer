# 🔍 DPI Engine — Deep Packet Inspection System

A C++ Deep Packet Inspection (DPI) engine that reads network captures, classifies traffic by application (YouTube, Netflix, TikTok, etc.), enforces blocking rules, and writes a filtered output PCAP. Ships with a full-stack web dashboard: upload a `.pcap`, set rules, and get instant traffic analytics with charts, flow tables, and download links.

---

## ✨ Features

- **Protocol parsing** — Ethernet/IPv4/TCP/UDP from raw PCAP bytes, zero external libraries
- **TLS SNI extraction** — identifies encrypted HTTPS destinations (YouTube, Facebook, 20+ apps) from the plaintext Client Hello field
- **HTTP Host header extraction** — classifies unencrypted HTTP traffic
- **Flow-based blocking** — block by app, source IP, or domain substring; all packets of a matched flow are dropped
- **Two engine variants** — single-threaded for simplicity, multi-threaded (LB → FP pipeline) for performance
- **Structured JSON output** — `--json <path>` flag emits a machine-readable report from both engines
- **Web dashboard** — FastAPI backend + single-file HTML/JS frontend; upload a pcap, run the engine, view charts and flow tables, download the filtered pcap

---

## 📸 Architecture

```
┌─────────────┐    ┌──────────────────────────────┐    ┌─────────────┐
│  input.pcap │───►│  DPI Engine (C++)            │───►│ output.pcap │
└─────────────┘    │  · Parse headers             │    └─────────────┘
                   │  · Extract TLS SNI           │    ┌─────────────┐
                   │  · Classify to app           │───►│ report.json │
                   │  · Apply blocking rules      │    └─────────────┘
                   └──────────────────────────────┘
                                  ▲
                   ┌──────────────┴──────────────┐
                   │  Web Dashboard (optional)   │
                   │  FastAPI backend + HTML UI  │
                   └─────────────────────────────┘
```

**Multi-threaded engine pipeline:**
```
Reader → [LB0, LB1, ...] → [FP0, FP1, FP2, FP3, ...] → Output Writer
           (hash by 5-tuple for consistent flow assignment)
```

---

## 📁 Project Structure

```
Packet_analyzer/
├── include/                    # C++ headers
│   ├── types.h                 # FiveTuple, AppType, BlockingRules
│   ├── pcap_reader.h           # PCAP file I/O
│   ├── packet_parser.h         # Ethernet/IP/TCP/UDP parsing
│   ├── sni_extractor.h         # TLS SNI + HTTP Host extraction
│   ├── json_writer.h           # Lightweight JSON serializer (no deps)
│   ├── thread_safe_queue.h     # Lock-free queue for MT engine
│   ├── load_balancer.h         # LB thread (MT engine)
│   └── fast_path.h             # FP thread (MT engine)
│
├── src/
│   ├── main_working.cpp        # ★ Simple single-threaded engine
│   ├── dpi_mt.cpp              # ★ Multi-threaded LB/FP engine
│   ├── packet_parser.cpp
│   ├── sni_extractor.cpp
│   ├── pcap_reader.cpp
│   └── types.cpp
│
├── web/
│   ├── backend/
│   │   ├── main.py             # FastAPI: upload → engine → JSON
│   │   ├── requirements.txt
│   │   └── bin/                # Place compiled binaries here
│   ├── frontend/
│   │   └── index.html          # Single-file dashboard (Chart.js)
│   └── README.md               # Web dashboard docs
│
├── generate_test_pcap.py       # Generates test_dpi.pcap
├── test_dpi.pcap               # Sample capture (multi-protocol)
└── CMakeLists.txt
```

---

## 🚀 Quick Start

### 1. Build the engines

```bash
# Simple engine
g++ -std=c++17 -O2 -I include -o dpi_simple \
    src/main_working.cpp src/pcap_reader.cpp src/packet_parser.cpp \
    src/sni_extractor.cpp src/types.cpp

# Multi-threaded engine
g++ -std=c++17 -pthread -O2 -I include -o dpi_engine \
    src/dpi_mt.cpp src/pcap_reader.cpp src/packet_parser.cpp \
    src/sni_extractor.cpp src/types.cpp
```

Or with CMake:
```bash
mkdir build && cd build && cmake .. && cmake --build .
```

### 2. Run on a capture

```bash
# Basic analysis
./dpi_simple test_dpi.pcap output.pcap

# Block YouTube and a specific IP, export JSON report
./dpi_engine test_dpi.pcap output.pcap \
    --block-app YouTube \
    --block-ip 192.168.1.50 \
    --block-domain tiktok \
    --json report.json \
    --lbs 2 --fps 2
```

### 3. Start the web dashboard

```bash
# Copy binaries to the web backend
cp dpi_simple web/backend/bin/
cp dpi_engine web/backend/bin/

# Install and run
cd web/backend
pip install -r requirements.txt
uvicorn main:app --port 8123
```

Open **http://127.0.0.1:8123** — drag in a pcap, add block rules, click Analyze.

---

## 🛠 CLI Reference

Both engines share the same flags:

| Flag | Description |
|---|---|
| `--block-app <name>` | Block app by name (YouTube, Netflix, TikTok, Facebook, etc.) |
| `--block-ip <ip>` | Block all traffic from a source IP |
| `--block-domain <str>` | Block any flow whose SNI contains this substring |
| `--json <path>` | Write structured JSON report to this path |

Additional flags for `dpi_engine` (multi-threaded):

| Flag | Default | Description |
|---|---|---|
| `--lbs <n>` | 2 | Number of Load Balancer threads |
| `--fps <n>` | 2 | Number of Fast Path threads per LB |

---

## 📊 JSON Report Format

```json
{
  "engine": "multi-threaded",
  "total_packets": 77,
  "forwarded": 75,
  "dropped": 2,
  "app_breakdown": [
    { "app": "HTTPS", "count": 39, "percent": 50.6 },
    { "app": "YouTube", "count": 4, "percent": 5.2 }
  ],
  "detected_domains": [
    { "domain": "www.youtube.com", "app": "YouTube" }
  ],
  "blocked_events": [
    { "src_ip": "10.0.0.5", "dest_ip": "142.250.185.206", "app": "YouTube", "sni": "www.youtube.com" }
  ],
  "flows": [
    { "src_ip": "10.0.0.5", "dest_ip": "31.13.64.35", "src_port": 52341,
      "dest_port": 443, "protocol": "TCP", "app": "Facebook",
      "sni": "www.facebook.com", "packets": 6, "bytes": 1240, "blocked": false }
  ],
  "load_balancers": [{ "id": 0, "dispatched": 53 }],
  "fast_paths": [{ "id": 0, "processed": 53 }]
}
```

---

## 🌐 Web Dashboard API

The FastAPI backend exposes three endpoints:

| Endpoint | Description |
|---|---|
| `POST /api/analyze` | Upload pcap + rules → returns JSON report |
| `GET /api/runs/{id}/output.pcap` | Download filtered output pcap |
| `GET /api/runs/{id}/report.json` | Download raw JSON report |
| `GET /api/health` | Check which engine binaries are available |

`POST /api/analyze` form fields: `pcap` (file), `engine` (`simple`/`multi-threaded`), `block_apps`, `block_ips`, `block_domains` (comma-separated strings), `lbs`, `fps` (integers).

---

## 🧠 How It Works

### TLS SNI Extraction
Even though HTTPS traffic is encrypted, the destination domain is sent in plaintext during the TLS handshake (the Client Hello message). We parse the raw bytes to find the SNI (Server Name Indication) extension:

```
TLS Client Hello → Extensions → Type 0x0000 (SNI) → "www.youtube.com"
```

### Flow-Based Blocking
We track flows by their 5-tuple (src IP, dst IP, src port, dst port, protocol). Once a flow is classified (e.g., as YouTube), every subsequent packet belonging to it is dropped — not just the one containing the Client Hello.

### Multi-threaded Pipeline
The MT engine distributes flows across threads using consistent hashing on the 5-tuple: `hash(flow) % num_lbs` picks a Load Balancer, then `hash(flow) % fps_per_lb` picks a Fast Path. This guarantees all packets of one flow land on the same FP thread, keeping flow state safe without locking.

---

## 🧪 Detected Applications

YouTube, Netflix, TikTok, Facebook, Instagram, Twitter, WhatsApp, Google, GitHub, Amazon, Reddit, Twitch, Spotify, Discord, Zoom, Microsoft, Apple, Cloudflare, Akamai, plus generic HTTP/HTTPS/DNS.

---

## 📋 Requirements

- C++17 compiler (g++ ≥ 7 or clang++ ≥ 5)
- POSIX threads (`-pthread`) for the MT engine
- Python 3.8+ with `fastapi`, `uvicorn`, `python-multipart` for the web dashboard
- No other external dependencies — PCAP parsing is built from scratch

---

## 📝 License

MIT License — see [LICENSE](LICENSE)
