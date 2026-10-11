# AegisWire NIDS

**AegisWire** is an experimental, educational **Network Intrusion Detection System (NIDS)** written in **C17** for **Windows 10/11**. It captures packets using **Npcap**, inspects Ethernet/IPv4/TCP headers and shows heuristic alerts in a native **Win32 GUI**.

> [!WARNING]
> **Experimental prototype v1.0 — not validated.** The source code has not yet been successfully built or functionally tested on Windows with Npcap for this release. Do not deploy it as a production security control.

## Table of contents

1. [Features](#features)
2. [Architecture](#architecture)
3. [Prerequisites](#prerequisites)
4. [Installation](#installation)
5. [Build](#build)
6. [Run](#run)
7. [Detection logic](#detection-logic)
8. [Authorized laboratory tests](#authorized-laboratory-tests)
9. [Troubleshooting](#troubleshooting)
10. [Known limitations](#known-limitations)
11. [Repository layout](#repository-layout)
12. [Roadmap](#roadmap)
13. [Credits and license](#credits-and-license)

## Features

- C17 application with native Win32 GUI: interface selection, **START**, **STOP**, **CLEAR LOG**.
- Live packet capture via Npcap with the BPF filter `ip and tcp`.
- Basic Ethernet, IPv4 and TCP header parsing and packet counters.
- Heuristic detection of TCP SYN scans, SYN bursts and repeated cleartext HTTP login request prefixes.
- Alert messages and counters displayed in the GUI.

These alerts are **indicators, not proof of an attack**. Thresholds are currently hard-coded.

## Architecture

```text
Windows network adapter
        |
  Npcap capture driver
        |
  BPF: ip and tcp
        |
  Capture worker thread
        |
  Ethernet / IPv4 / TCP parsing
        |
  Per-source heuristic counters
        |
  Win32 messages -> GUI log + counters
```

The prototype currently keeps the capture, parsing, detection and GUI code in `src/main.c`. See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## Prerequisites

| Component | Purpose |
| --- | --- |
| Windows 10/11 | Target operating system |
| [Npcap runtime](https://npcap.com/#download) | Driver and runtime libraries for packet capture |
| [Npcap SDK](https://npcap.com/#download) | Headers and import libraries for compilation |
| [Visual Studio Build Tools](https://visualstudio.microsoft.com/downloads/) | MSVC compiler, linker and Windows SDK |
| [CMake 3.20+](https://cmake.org/download/) | Configure and build the application |
| Git (optional) | Clone the repository |

The documented build targets **x64**. The Npcap runtime and Npcap SDK are **different components**: install the runtime and separately extract the SDK.

## Installation

### Step 1 — Install Npcap

1. Download the Windows Npcap installer from the official website.
2. Run the installer and follow its prompts.
3. Note any option that restricts capture to administrators; your runtime permissions depend on the installation settings.
4. Restart if the installer requests it.

### Step 2 — Extract the Npcap SDK

1. Download the SDK from the official Npcap website.
2. Extract it to a folder such as `C:\Npcap-SDK`.
3. Check that the selected folder directly contains the expected files:

```text
C:\Npcap-SDK\Include\pcap.h
C:\Npcap-SDK\Lib\x64\wpcap.lib
C:\Npcap-SDK\Lib\x64\Packet.lib
```

If extraction creates an extra nested directory, use the **inner folder containing `Include` and `Lib`** as the SDK path.

### Step 3 — Install Visual Studio Build Tools

1. Install Visual Studio Build Tools from Microsoft's official download page.
2. Select the **Desktop development with C++** workload.
3. Ensure the **MSVC x64/x86 build tools** and a **Windows 10/11 SDK** are selected.
4. Open **Developer PowerShell for Visual Studio** from the Start menu.

### Step 4 — Install and verify CMake

Install CMake 3.20 or newer. In Developer PowerShell, run:

```powershell
cmake --version
cl
Test-Path 'C:\Npcap-SDK\Include\pcap.h'
Test-Path 'C:\Npcap-SDK\Lib\x64\wpcap.lib'
Test-Path 'C:\Npcap-SDK\Lib\x64\Packet.lib'
```

`cl` should display the compiler banner (it may also complain that no source file was specified). Each `Test-Path` should return `True` if the SDK uses the example path. The `Test-Path` commands are **PowerShell-specific**.

## Build

### Step 1 — Obtain the source

Download the source ZIP using the green **Code** button on GitHub, or clone it:

```powershell
git clone https://github.com/PedroGoncalves28286/AegisWire-NIDS.git
cd AegisWire-NIDS
```

For a ZIP download, extract it and open Developer PowerShell in the folder containing `CMakeLists.txt`.

### Step 2 — Configure and compile

With **Visual Studio 2022 Build Tools** installed:

```powershell
cmake -S . -B build -G 'Visual Studio 17 2022' -A x64 -DNPCAP_SDK='C:/Npcap-SDK'
cmake --build build --config Release
```

If you have a different Visual Studio version, select the appropriate generator; run `cmake -G` to list available generators. If your default generator is already Visual Studio, you may omit the explicit `-G` argument. **Do not use `-A x64` with generators such as Ninja**, which do not support that option.

**Expected executable, if compilation succeeds:**

```text
build/Release/AegisWire.exe
```

The current `CMakeLists.txt` checks for `Include/pcap.h`, selects `Lib/x64` for 64-bit builds, compiles `src/main.c` and links `wpcap`, `Packet` and `ws2_32`. A successful build has **not yet been demonstrated**. If changing the generator or architecture, delete the old `build` directory before reconfiguring.

## Run

1. Ensure the **Npcap runtime** is installed.
2. After a successful build, run `build/Release/AegisWire.exe` with appropriate capture permissions.
3. Select a network interface from the drop-down menu. The prototype accepts only **Ethernet-type** interfaces (`DLT_EN10MB`).
4. Click **START** to begin monitoring.
5. Observe **Packets**, **Alerts** and the log pane.
6. Click **STOP** to stop capture; **CLEAR LOG** clears the visible log text.

> [!NOTE]
> The interface must actually receive the traffic being tested. On switched or virtual networks, traffic between other machines may not be visible without an appropriate network configuration or mirrored traffic.

## Detection logic

The current code uses a fixed-size per-source-IP tracker with a **10-second tracking window**.

| Signal | Trigger | Limitation |
| --- | --- | --- |
| Possible TCP SYN port scan | **12 distinct TCP destination ports** from one source in the window | Hash collisions and shared alert cooldown can affect results |
| Possible SYN flood | **150 SYN packets** from one source in the window | No traffic baseline or per-destination aggregation |
| Possible HTTP login burst | **5** request starts matching `POST /login` or `POST /signin` on destination port **80/8080** | Counts prefixes in individual TCP segments; does not verify failed logins |

The HTTP detector only inspects **unencrypted traffic** and does not reconstruct TCP streams. HTTPS request content cannot be read by this packet sniffer. A single shared cooldown can suppress alerts of another type.

## Authorized laboratory tests

Use only networks and hosts that you own or are **explicitly authorized** to monitor or test. An isolated virtual network is recommended.

1. **Build check:** record Windows version, MSVC version, Npcap runtime/SDK version, warnings and linker output.
2. **GUI check:** confirm interface enumeration and START/STOP/CLEAR LOG controls.
3. **Capture check:** generate ordinary authorized TCP traffic on the selected interface and observe packet counters.
4. **Scan heuristic check:** in an isolated lab, generate a limited, authorized scan of distinct TCP ports on a test host and compare observed alerts with the threshold.
5. **HTTP check:** send repeated unencrypted requests to a local test `/login` or `/signin` endpoint and check for the heuristic alert.
6. **Stability check:** stop and exit during idle and active capture; record any freezes or errors.
7. **Robustness:** plan future tests for truncated and malformed PCAP data before making security or reliability claims.

Record **actual** test outcomes and screenshots only after executing the tests. See [docs/TESTING.md](docs/TESTING.md). **No passing test results are claimed in this repository.**

## Troubleshooting

| Problem | What to check |
| --- | --- |
| `cmake` not recognized | Install CMake, add it to PATH and reopen the terminal |
| `cl` not recognized | Open Developer PowerShell / x64 Native Tools Command Prompt |
| `Pass -DNPCAP_SDK=...` | Point to the SDK folder containing `Include/pcap.h` |
| `wpcap.lib` or `Packet.lib` not found | Verify SDK `Lib/x64` directory and architecture |
| `-A` platform not supported | Use a Visual Studio generator instead of Ninja |
| DLL missing at runtime | Install/repair the official Npcap runtime; do not download DLLs from unofficial sites |
| No interfaces listed | Verify Npcap installation and permissions |
| “Ethernet interfaces only” | Select a compatible Ethernet-type capture interface |
| Packet counter does not increase | Check selected interface, BPF filter and actual traffic visibility |
| No alerts | Verify thresholds, tracking window and visible traffic |
| GUI freezes on STOP or exit | Record the steps; the prototype's capture shutdown requires validation and may need code fixes |

**A successful compile does not prove correct or safe runtime behavior.**

## Known limitations

- Experimental prototype; no successful Windows/Npcap build or functional validation has been reported for this release.
- Ethernet/IPv4/TCP only; no VLAN, IPv6, UDP, ICMP or ARP parsing.
- No TCP stream reassembly, IP fragment reconstruction, or TLS inspection.
- No external signature engine, rule files or editable thresholds.
- Fixed-size tracking table with collisions and shared alert cooldown.
- No persistent event logs, JSON export, SIEM integration or automated tests.
- No performance benchmarks, false-positive measurements or memory-safety audit.
- GUI message handling, capture shutdown and malformed input behavior still require testing.
- Captured network data may be sensitive; do not publish private payloads or credentials.

## Repository layout

```text
AegisWire-NIDS/
├── .gitignore
├── CMakeLists.txt
├── CONTRIBUTING.md
├── LICENSE
├── README.md
├── SECURITY.md
├── docs/
│   ├── ARCHITECTURE.md
│   └── TESTING.md
└── src/
    └── main.c
```

## Roadmap

- [ ] Build and smoke-test on Windows 11 with Npcap.
- [ ] Resolve compiler, linker, shutdown and threading defects found in testing.
- [ ] Add offline PCAP replay and automated parser tests.
- [ ] Split capture, parser, detection and GUI into independent modules.
- [ ] Support VLAN, IPv6, UDP, ICMP and ARP.
- [ ] Add configurable rules, JSON alerts and persistent logs.
- [ ] Improve tracking accuracy and alert cooldown behavior.
- [ ] Integrate with Wazuh and other SIEM platforms.
- [ ] Publish reproducible performance and detection benchmarks.
- [ ] Add verified application screenshots.

## Contributing and responsible use

Contributions and bug reports are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) and [SECURITY.md](SECURITY.md).

**Monitor only systems and networks you own or have explicit authorization to assess.** AegisWire is not a substitute for Snort, Suricata or a validated enterprise IDS.

## Credits and license

Maintainer: [PedroGoncalves28286](https://github.com/PedroGoncalves28286).

Inspired by the educational presentation *Minimalistic IDS (Intrusion Detection System) in C — Part 1* by **Licínio Ferreira**. This attribution does not imply the presenter's endorsement or authorship of AegisWire.

Licensed under the **MIT License** — see [LICENSE](LICENSE).
