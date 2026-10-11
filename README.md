# AegisWire NIDS

**AegisWire** is an experimental, educational **Network Intrusion Detection System (NIDS)** written in **C17** for **Windows 10/11**. It captures Ethernet traffic through **Npcap**, parses IPv4/TCP headers and displays heuristic alerts in a native **Win32 GUI**.

> **Status: experimental prototype (v1.0).** The source code has not been built or validated on Windows with Npcap in this delivery. Do not deploy it as a production security control.

## Highlights

- Native C17 application with Win32 interface (interface selection, START, STOP, CLEAR LOG).
- Npcap live packet capture using a BPF filter (`ip and tcp`).
- Basic Ethernet / IPv4 / TCP parsing and packet counters.
- Heuristic detection of **TCP SYN port scans** (12 distinct destination ports within a 10-second tracking window).
- Heuristic detection of **SYN bursts** (150 SYN packets within a tracking window).
- Basic detection of repeated cleartext HTTP requests beginning with `POST /login` or `POST /signin` on TCP destination ports 80/8080.

These are **indicators**, not proof of compromise. Detection thresholds are currently hard-coded.

## Requirements

- Windows 10 or 11, 64-bit recommended.
- Npcap runtime installed (capture permissions may require administrator privileges).
- [Npcap SDK](https://npcap.com/#download) extracted to a local folder.
- Visual Studio C++ build tools with Windows SDK, and CMake 3.20+.

## Build (Visual Studio Developer PowerShell)

```powershell
cmake -S . -B build -A x64 -DNPCAP_SDK="C:/Npcap-SDK"
cmake --build build --config Release
```

The expected output is `build/Release/AegisWire.exe`. The Npcap runtime must be installed on the machine running the program. Adapt `NPCAP_SDK` to your extracted SDK path.

## Run

1. Install Npcap and compile the application.
2. Run `AegisWire.exe` with appropriate capture permissions.
3. Select a supported **Ethernet-type** capture interface.
4. Click **START** to begin capture; view counters and alerts.
5. Click **STOP** to end capture. Use **CLEAR LOG** to clear the visible log.

Only monitor networks you own or are explicitly authorized to assess.

## Detection details and limitations

| Signal | Current approach | Important limitation |
|---|---|---|
| SYN port scan | Counts unique TCP destination ports per source | Fixed-size hash tracker; collisions and shared alert cooldown can affect results |
| SYN flood | Counts SYN packets per source | No baseline, no per-destination aggregation; false positives possible |
| HTTP login burst | Matches request prefix in individual TCP segments | Does not reconstruct TCP streams or determine whether login failed; HTTPS is opaque |

Additional limitations: Ethernet/IPv4/TCP only; no VLAN-tag parsing; no IPv6, UDP or ARP analysis; no persistent logging; no signature rule engine; no TCP reassembly; no performance benchmarks; and no automated test suite yet. Packet capture visibility depends on interface and network topology.

## Repository layout

```text
AegisWire-NIDS/
├── src/main.c             # Capture, packet parsing, detection, Win32 GUI
├── CMakeLists.txt         # Windows build configuration
├── docs/ARCHITECTURE.md   # Architecture and data flow
├── docs/TESTING.md        # Manual verification checklist
├── SECURITY.md           # Security policy
├── CONTRIBUTING.md       # Contribution guidelines
├── LICENSE               # MIT license
└── .gitignore
```

## Roadmap

- [ ] Build and smoke-test on Windows 11 with Npcap.
- [ ] Separate capture, parser, detection and UI modules.
- [ ] Add PCAP replay and automated parser tests.
- [ ] Support VLAN, IPv6, UDP, ICMP and ARP.
- [ ] Add configuration files, structured JSON alerts and logging.
- [ ] Improve tracking accuracy, cooldown logic and GUI performance.
- [ ] Add Wazuh/SIEM export and benchmark detection accuracy.

## Responsible use

This project is for defensive learning and authorized laboratory monitoring. It is not a replacement for Snort, Suricata or a validated enterprise IDS.

## Credits

Inspired by the educational presentation *Minimalistic IDS (Intrusion Detection System) in C — Part 1* by **Licínio Ferreira**. AegisWire is a separately packaged prototype; this attribution does not imply endorsement or authorship by the presenter.

## License

MIT — see [LICENSE](LICENSE).
