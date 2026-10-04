# Pickko AdGuard 🛡️

On-device Android DNS firewall and ad-blocking engine powered by `LocalVpnService` and Capacitor. Intercepts DNS queries locally on UDP port 53 without routing user traffic through third-party VPN servers, mitigating trackers, telemetry, and malicious advertising scripts at the network layer with zero root requirements.

<div align="center">
  <img src="assets/brand/logo.png" alt="Pickko AdGuard Logo" width="280" />
</div>

<p align="center">
  <img src="https://img.shields.io/badge/Platform-Android%208.0%2B-38BDF8?style=flat-square&logo=android&logoColor=white" alt="Android" />
  <img src="https://img.shields.io/badge/Framework-Capacitor%208-38BDF8?style=flat-square&logo=capacitor&logoColor=white" alt="Capacitor" />
  <img src="https://img.shields.io/badge/Architecture-LocalVpnService-38BDF8?style=flat-square" alt="Architecture" />
  <img src="https://img.shields.io/badge/License-MIT-green?style=flat-square" alt="License" />
  <img src="https://img.shields.io/github/last-commit/pikadexofc/pickko-adguard?style=flat-square" alt="Last Commit" />
</p>

---

## 📐 System Architecture

Unlike traditional VPNs that route all device traffic through a remote proxy server, Pickko AdGuard establishes an **on-device loopback VPN tunnel** (`10.1.1.1/24`, IPv6 `fd00:1:fd00:1:fd00:1:fd00:1/128`). It selectively filters network packets at the transport layer, inspecting exclusively UDP port 53 (DNS) datagrams while allowing all other device traffic to route natively.

```
+-------------------------------------------------------------------------+
|                              Android Device                             |
|                                                                         |
|  +------------------+           +------------------------------------+  |
|  | Applications &   |  DNS Req  | Virtual TUN Interface              |  |
|  | System Services  | --------> | (10.1.1.1/24, MTU 1500)            |  |
|  +------------------+ (UDP 53)  +-----------------+------------------+  |
|                                                   |                     |
|                                                   | raw IP packet       |
|                                                   v                     |
|  +-------------------------------------------------------------------+  |
|  | LocalVpnService.java (Worker Thread Pool)                         |  |
|  |                                                                   |  |
|  | 1. Extract IP header (IPv4 IHL / IPv6 RFC 2460)                   |  |
|  | 2. Filter Protocol 17 (UDP) & Dest Port 53                        |  |
|  | 3. Byte-by-byte DNS question domain extraction                    |  |
|  | 4. In-Memory Radix & LRU Evaluation (AD_DOMAINS + 2000-entry LRU) |  |
|  +-------------------+-----------------------------------------------+  |
|                      |                                                  |
|       Match (Blocked)|                                No Match (Clean)  |
|                      v                                                  v
|  +------------------------------------+    +-------------------------+  |
|  | Synthetic RFC 1035 NXDOMAIN Packet |    | protect(DatagramSocket) |  |
|  | (RCODE 3 Name Error Injection)     |    | Bypass local VPN tunnel |  |
|  +-------------------+----------------+    +------------+------------+  |
|                      |                                  |               |
|                      | inject response                  | forward query |
+----------------------|----------------------------------|---------------+
                       v                                  v
              Immediate Resolution              Upstream DNS Resolver
              into TUN Interface             (1.1.1.1, 9.9.9.9, 94.140.14.14)
```

---

## ⚡ Core Technical Mechanisms

### 1. Raw TUN Packet Capture & Header Demuxing
The service binds a virtual file descriptor via `VpnService.Builder`:
* Configures local gateway `10.1.1.1`, DNS endpoint `10.1.1.2`, and IPv6 `fd00:1:fd00:1:fd00:1:fd00:1`.
* Reads raw IP packets directly from `ParcelFileDescriptor` into a 4-thread worker pool (`ExecutorService`).
* Dynamically parses IPv4 (IHL offset) and IPv6 (RFC 2460 fixed 40-byte header) structures to isolate UDP datagrams targeting destination port 53.

### 2. Byte-by-Byte DNS Host Parsing & Radix Matching
* DNS question sections are parsed directly from raw byte buffers without third-party parsing dependencies.
* Domain names are evaluated against an in-memory blocklist (`AD_DOMAINS`) utilizing suffix and subdomain tree walking.
* Lookups are accelerated by an `LruCache<String, Boolean>` (2,000 capacity) for sub-millisecond filtering latency.

### 3. Synthetic RFC-Compliant NXDOMAIN Injection
* When a requested domain matches a known ad network, tracker, or telemetry endpoint, `LocalVpnService` constructs a synthetic RFC 1035 response packet.
* Sets response header flags `0x8183` (`QR=1`, `AA=0`, `TC=0`, `RD=1`, `RA=1`, `RCODE=3` - Non-Existent Domain).
* Swaps source and destination IPv4/IPv6 addresses and UDP port numbers, recalculates IPv4 checksums, and writes the packet directly back into the TUN `FileOutputStream`.
* Host applications receive an immediate `NXDOMAIN` response, aborting tracking requests with zero network transit delay.

### 4. Upstream Forwarding via Protected Sockets
* Legitimate DNS queries are forwarded upstream to user-configured resolvers:
  * **AdGuard DNS** (`94.140.14.14` / `94.140.14.15`)
  * **Cloudflare** (`1.1.1.1`)
  * **Quad9** (`9.9.9.9`)
  * **NextDNS** (`45.90.28.0`)
* `protect(DatagramSocket)` is invoked on all upstream sockets to prevent recursive routing loops through the local VPN tunnel.
* Resolved responses are cached in a 500-entry `DNS_RESULT_CACHE` for instantaneous resolution of frequent queries.

### 5. Capacitor Native Bridge & UI Integration
* **`VpnPlugin.java`**: A custom `@CapacitorPlugin(name = "VpnPlugin")` exposes lifecycle controls (`enableShield`, `disableShield`, `getRecentActivity`, `setProvider`) to the frontend.
* Negotiates Android system consent via `VpnService.prepare(Context)`.
* **Frontend Shell**: React 18, TypeScript, and Tailwind CSS provide a responsive interface displaying real-time query counters, blocked metrics, and traffic telemetry buffers.

---

## 🔐 Android System Integration & Permissions

Pickko AdGuard integrates directly with Android's system networking services:

| Permission / Component | Declaration | Architectural Purpose |
|------------------------|-------------|-----------------------|
| `BIND_VPN_SERVICE` | `<service android:permission="android.permission.BIND_VPN_SERVICE">` | Enforces OS-level security binding for `LocalVpnService`. |
| `FOREGROUND_SERVICE_VPN` | `<uses-permission android:name="android.permission.FOREGROUND_SERVICE_VPN" />` | Guarantees continuous background execution under Android 14+ foreground service policies. |
| `RECEIVE_BOOT_COMPLETED` | `<receiver android:name=".BootReceiver">` | Restores shield protection automatically upon device restart. |
| `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` | `<uses-permission android:name="..." />` | Prevents aggressive OEM task killers from killing the DNS listener during deep sleep. |

---

## 🛠️ Build & Installation

### Prerequisites
* **Node.js**: v18.0.0 or higher
* **Java Development Kit (JDK)**: OpenJDK 17 or higher
* **Android SDK**: API Level 34 (Android 14) with Build-Tools 34.0.0+

### Compilation Steps

1. **Install Web Dependencies**:
   ```bash
   npm install
   ```

2. **Compile Web Assets**:
   ```bash
   npm run build
   ```

3. **Synchronize Capacitor Android Project**:
   ```bash
   npx cap sync android
   ```

4. **Build Android APK**:
   ```bash
   cd android
   ./gradlew assembleRelease
   ```
   *Generated binary will be located at `android/app/build/outputs/apk/release/app-release.apk`.*

---

## 🛡️ Privacy & Performance Invariants

* **100% On-Device Processing**: Blocklist evaluation and synthetic packet construction occur exclusively in local memory.
* **Zero Payload Inspection**: Only UDP port 53 packets are inspected. HTTP, HTTPS, WebSocket, and application data bypass filtering unmodified.
* **No Cloud Telemetry**: Real-time activity logs exist strictly in a 15-entry volatile ring buffer in RAM. Zero telemetry or analytical events are recorded or transmitted.
* **No Root Required**: Operates entirely within standard Android user-space APIs.

---

<div align="center">
  <p><b>PixelPie Media</b> • Engineered by Pickko</p>
  <p><i>High-performance, local-first system utilities for Android.</i></p>
</div>
