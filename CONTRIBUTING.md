# Contributing

Contributions are welcome — bug fixes, new app signatures, protocol support (QUIC, DoH/DoT), or dashboard improvements.

## Getting started

1. Fork the repository and clone your fork
2. Create a branch: `git checkout -b feature/your-feature`
3. Make your changes and test them (see below)
4. Push and open a Pull Request

## Testing your changes

Generate a fresh test capture and run both engines:

```bash
python3 generate_test_pcap.py      # creates test_dpi.pcap

# Build and run simple engine
g++ -std=c++17 -O2 -I include -o dpi_simple \
    src/main_working.cpp src/pcap_reader.cpp src/packet_parser.cpp \
    src/sni_extractor.cpp src/types.cpp
./dpi_simple test_dpi.pcap /tmp/out.pcap --json /tmp/report.json
python3 -m json.tool /tmp/report.json   # should parse cleanly

# Build and run multi-threaded engine
g++ -std=c++17 -pthread -O2 -I include -o dpi_engine \
    src/dpi_mt.cpp src/pcap_reader.cpp src/packet_parser.cpp \
    src/sni_extractor.cpp src/types.cpp
./dpi_engine test_dpi.pcap /tmp/out_mt.pcap --json /tmp/report_mt.json
```

## Adding a new application signature

In `src/types.cpp`, find `sniToAppType()` and add a new pattern:

```cpp
if (sni.find("twitch.tv") != std::string::npos) return AppType::TWITCH;
```

Also add the enum value to `AppType` in `include/types.h` and the string mapping in `appTypeToString()`.

## Code style

- C++17, no external dependencies for the core engine
- Match the surrounding indentation and naming style
- Keep each component focused — parser parses, extractor extracts, etc.
