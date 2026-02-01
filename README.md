# Cynan SBC

Cynan SBC is a high-performance, modern Session Border Controller (SBC) engineered in pure Rust. It is designed to provide robust security, signaling normalization, and media anchoring for Voice over IP (VoIP) infrastructures, serving as a state-of-the-art alternative to traditional SBC solutions.

## Architecture

The project is built as a modular Rust workspace:

- **`cynan-api`**: Management REST API for configuration and trunk control.
- **`cynan-sip`**: Real-time SIP signaling engine (B2BUA) with asynchronous session management.
- **`cynan-media`**: High-throughput RTP media plane for packet relaying and anchoring.
- **`cynan-core`**: Shared primitives, configuration models, and safety-focused error handling.
- **`cynan-dist`**: Decentralized state synchronization library using Zenoh mesh.
- **`cynan-bench`**: High-performance SIP load generator for cluster validation.
- **`cynan-arcrtc`**: Implementation of the **ArcRTC** signaling and key exchange protocol.

## Key Features

- **Pure Rust Implementation**: Memory safety and carrier-grade performance without the overhead of mixed-language architectures.
- **Post-Quantum Security (PQC)**: Integral support for NIST-standard algorithms including **ML-KEM-768** (Kyber) and **FN-DSA-512** (Falcon) for future-proof signaling and transport.
- **T.38 Fax Relay & Reliability**: Zero-copy transparent UDPTL relay with autonomous transition detection and real-time sequence gap analysis.
- **Regulatory Compliance Mode**: Native **HIPAA, SOC 2, PCI/DSS, and GLBA** compliant auditing with secure DTLS-wrapped media pathways for sensitive data.
- **Asynchronous Core**: Built on top of `Tokio` for high concurrency and low latency.
- **Modular Control**: Fully configurable via a REST API.
- **Media Anchoring**: Bi-directional RTP and UDPTL relaying.

## Installation

The easiest way to install Cynan SBC on a Linux server (Debian/Ubuntu/RHEL/CentOS) is using the automated installer.

 ```bash
 sudo ./install.sh
 ```

This script will:
1. Install all system dependencies (OpenSSL, Opus, Clang, PostgreSQL libs).
2. Install Rust toolchain (if missing).
3. Build the project in Release mode.
4. Install binaries to `/opt/cynan/bin`.
5. Setup and enable Systemd services (`cynan-sip`, `cynan-api`).
 
## Access & Deployment

**Cynan SBC** is a proprietary, carrier-grade security appliance. To maintain the integrity of our Post-Quantum Cryptography (PQC) and zero-copy media heuristics, the source code is **Source-Available** but not public.

### Obtaining the SBC
- **Binaries**: Pre-compiled, FIPS-compliant binaries for Ubuntu/RHEL are available for licensed partners.
- **Source Audit**: Secure source-code access for security auditing purposes (under NDA) can be requested via [Fastcomcorp Support](mailto:support@fastcomcorp.com).

### Quick Launch (Licensed Partners) - Coming Soon
Licensed users can deploy the SBC using our unified installer script:

```bash
# Download and run the Cynan Secure Installer
curl -sSL https://get.fastcomcorp.com/cynan-sbc | sudo bash
```

## Multi-Node Cluster Simulation

To simulate a masterless cluster on a single machine:

1. **Node A**:
   ```bash
   cargo run -p cynan-sip -- --addr 127.0.0.1:5060 --zenoh-listen tcp/127.0.0.1:7447
   ```

2. **Node B**:
   ```bash
   cargo run -p cynan-sip -- --addr 127.0.0.1:5061 --zenoh-connect tcp/127.0.0.1:7447 --zenoh-listen tcp/127.0.0.1:7448
   ```

3. **Verify Failover**:
   Use `cynan-bench` to send INVITEs to Node A and BYEs to Node B:
   ```bash
   cargo run -p cynan-bench -- --target 127.0.0.1:5060 --secondary-target 127.0.0.1:5061 --cps 10 --duration 5
   ```

## Technical Specifications

### RFC Compliance

Cynan SBC is engineered to adhere to the following IETF standards to ensure maximum interoperability:

- **Signaling (SIP)**:
    - **RFC 3261**: SIP: Session Initiation Protocol (Core Engine).
    - **RFC 3581**: An Extension to SIP for Symmetric Response Routing (`rport`).
    - **RFC 3264**: An Offer/Answer Model with SDP.
- **Media (RTP/RTCP)**:
    - **RFC 3550**: RTP: A Transport Protocol for Real-Time Applications.
    - **RFC 3551**: RTP Profile for Audio and Video Conferences with Minimal Control.
- **Security & Encryption**:
    - **RFC 4568**: Session Description Protocol (SDP) Security Descriptions for Media Streams (SDES-SRTP).
    - **RFC 3711**: The Secure Real-time Transport Protocol (SRTP).
- **Codecs**:
    - **RFC 6716**: Definition of the Opus Audio Codec.
    - **RFC 7587**: RTP Payload Format for output of the Opus Audio Codec.
- **Fax & Compliance**:
    - **RFC 3362**: Real-time Facsimile (T.38) over UDP (UDPTL).
    - **FIPS 140-3**: Compliance standards for cryptographic modules (AES-GCM-256).
    - **HIPAA/SOC2/HITECH**: Native audit hooks for PHI/PII protection.
- **Post-Quantum Security (PQC)**:
    - **NIST FIPS 203**: Module-Lattice-Based Key-Establishment Analysis (ML-KEM/Kyber).
    - **NIST FIPS 204**: Module-Lattice-Based Digital Signature Standard (ML-DSA/Falcon).
- **ArcRTC**:
    - **ArcSignaling**: Custom JSON-based low-latency signaling.
    - **ArcSRTP**: Enhanced SRTP key exchange using P-256 HKDF.

### Distributed Architecture (Zenoh)

For high availability and cluster-wide state synchronization, Cynan SBC utilizes **Zenoh** as its decentralized backbone. Unlike traditional centralized clusters, Cynan nodes communicate peer-to-peer to share active call sessions and media port availability.

- **Masterless Design**: No single point of failure.
- **Queryable State**: Nodes can recover session state from any active peer in the mesh.
- **Zero-Overhead Discovery**: Automatic peer discovery without manual configuration.

## Security & Hardening

Cynan SBC implements "Defense-in-Depth" by integrating firewall-grade security logic directly into the application layer.

### Dynamic IP Ban (Fail2Ban-style)
To protect against brute-force attacks and packet floods, the SIP engine enforces a strict "Three Strikes" policy:
- **Threshold**: **10 Security Violations** within a **1-minute window**.
- **Action**: The source IP is automatically added to a **Blocklist**.
- **Duration**: The ban remains active for **1 Hour** (60 minutes).
- **Violations**: A "violation" is triggered by:
    - Sending malformed SIP packets.
    - Sending non-UTF8 binary garbage.
    - Attempting to bypass Access Control Lists (ACLs).
    - Using known scanner User-Agents.

### Scanner Mitigation
The SBC actively fingerprints and drops connections from known vulnerability scanners:
- **User-Agent Filtering**: Traffic from `sipvicious`, `friendly-scanner`, and `pplsip` is instantly dropped.
- **Topology Hiding**: Internal headers (`X-Internal-IP`, `Record-Route`) are stripped from egress responses, and the `Server` header is masked to `CynanSBC` to prevent OS/Version fingerprinting.

## Developer & Troubleshooting Guide

### Troubleshooting 101

1.  **Signaling Issues**: Monitor the `cynan-sip` logs. Since we use an in-memory `DistributedStore`, check if the node is correctly joined to the Zenoh mesh.
2.  **Media Gaps**: If audio is one-way or missing, verify the `Symmetric RTP` logs in `cynan-media`. The SBC will automatically learn the correct egress IP/Port once it receives the first packet.
3.  **Database Sync**: `cynan-sip` refreshes its configuration from PostgreSQL every 60 seconds. Changes made via `cynan-api` might take up to a minute to reflect if not using manual triggers.

### Internal Logic Flow

1.  **Ingress**: `cynan-sip` receives INVITE → Checks ACLs → Parses SDP → Requests RTP port from `cynan-media`.
2.  **State Sync**: `CallManager` updates the `DistributedStore` (Zenoh broadcast).
3.  **Relay**: `cynan-media` anchors the media and provides bi-directional relaying.
4.  **Egress**: `cynan-sip` forwards the modified INVITE to the target Trunk.

## Third-Party Open Source Contributions

We would like to give credit and thanks to the following open-source projects that make Cynan SBC possible:

- **[Zenoh](https://zenoh.io/)**: For providing the high-performance decentralized communication layer.
- **[rsip](https://github.com/v0y4ger/rsip)**: For the robust SIP primitive parsing.
- **[Tokio](https://tokio.rs/)**: The foundation of our asynchronous multicore execution.
- **[Axum](https://github.com/tokio-rs/axum)**: For the modern and safe management API.
- **[sqlx](https://github.com/launchbadge/sqlx)**: For safe and fast PostgreSQL integration.
- **[webrtc-srtp](https://github.com/webrtc-rs/srtp)**: For the carrier-grade media encryption layer.

### Security Inspirations
- **[Fail2Ban](https://github.com/fail2ban/fail2ban)**: The "User-Agent Filtering" and "Dynamic IP Ban" concepts in `cynan-sip` are architecturally inspired by the Fail2Ban intrusion prevention framework.

## Authors

- **Fastcomcorp, LLC**

## Integration with Main Cynan

Cynan SBC is designed to work as a high-performance signaling and media engine within the [Fastcomcorp/Cynan](https://github.com/Fastcomcorp/Cynan) ecosystem.

- **Shared Persistence**: Connects to the main Cynan database for unified trunk and ACL management.
- **REST Control**: Fully manageable via the main Cynan dashboard through the `cynan-api` layer.
- **Mesh Ready**: Interoperates with other Cynan nodes using Zenoh for cluster-wide state.

For detailed instructions, see the [Integration Guide](INTEGRATION.md).

## License

**Proprietary to Fastcomcorp, LLC**

All rights reserved. This software and associated documentation files are the exclusive property of Fastcomcorp, LLC. Unauthorized copying, modification, distribution, or use of this software, via any medium, is strictly prohibited.
