# 5G FWA CPE TR-369/USP Management API Reference

> A practical reference for system integrators, operator NOC teams, and OSS/BSS developers integrating Honlly 5G FWA CPE into TR-369 USP-based management platforms.

---

## Overview

This document describes the TR-369 User Services Platform (USP) management interface implemented on Honlly 5G Fixed Wireless Access CPE devices (HL-850Q, HL-810Z, HL-830M, and compatible models). It covers the USP agent architecture, supported service objects, message flow patterns, and practical integration examples for common operator workflows.

TR-369 USP is the Broadband Forum's next-generation device management protocol, designed to replace TR-069 (CWMP) for broadband and FWA CPE. USP uses a message-bus architecture over WebSocket or MQTT transport with Protocol Buffer (protobuf) encoding, enabling efficient bidirectional communication, push-based telemetry, and multi-controller operation.

## Supported Honlly CPE Models

| Model | Chipset | USP Agent Version | Transport |
|-------|---------|-------------------|-----------|
| HL-850Q | Qualcomm SDX62 | USP 1.3 | WebSocket, MQTT, STOMP |
| HL-810Z | UNISOC UDX710 + QCA IPQ5018 | USP 1.2 | WebSocket, MQTT |
| HL-830M | MTK T750 | USP 1.3 | WebSocket, MQTT |
| HL-820Z | UNISOC UDX710 | USP 1.2 | WebSocket, MQTT |
| HL-880Q | Qualcomm (ODU) | USP 1.3 | WebSocket, MQTT |

*Contact Honlly for the full compatibility matrix and firmware version requirements.*

## USP Agent Architecture

Each Honlly CPE runs a native USP agent implemented as a daemon process within the device firmware. The agent:

- **Maintains a persistent WebSocket connection** to the USP Controller (typically the operator's ACS or cloud management platform).
- **Exposes the TR-181 Device Data Model** (Device:2) via USP Get/Set/Add/Delete messages.
- **Supports the USP Operate mechanism** for invoking device commands (reboot, factory reset, connection request, firmware upgrade, diagnostic tests).
- **Pushes telemetry via USP Notify** — including signal strength (RSRP/RSINR), throughput KPIs, alarm conditions, and connection state changes — at configurable intervals or on threshold breach.

### Supported USP Message Types

| Message | Direction | Description |
|---------|-----------|-------------|
| Get | Controller → Agent | Read parameter values (single, partial path, or full subtree) |
| Set | Controller → Agent | Write parameter values (with atomic commit for multi-param sets) |
| Add | Controller → Agent | Create new table row instances |
| Delete | Controller → Agent | Remove table row instances |
| Operate | Controller → Agent | Execute device commands |
| Notify | Agent → Controller | Push events (value change, alarm, boot, periodic) |
| GetSupportedDM | Controller → Agent | Discover supported data model objects |

## Data Model Coverage (Key Objects)

The USP agent exposes the following TR-181 Device:2 data model objects relevant to FWA CPE management:

### Device.WiFi.

- Radio.{i}. — Configure 2.4 GHz and 5 GHz radios (channel, bandwidth, TX power)
- SSID.{i}. — Configure SSID name, security mode, WPA3 transition mode
- AccessPoint.{i}. — Associate radios with SSIDs, configure band steering
- Radio.Stats.{i}. — Per-radio channel utilization, TX/RX packet counters, error rates

### Device.Cellular.

- Interface.{i}. — Cellular WAN interface status, APN configuration, PDP context management
- Interface.{i}.Stats. — RSRP, RSRQ, RSINR, cell ID, PLMN, band, bandwidth, MIMO layers in use
- Interface.{i}.SIM. — SIM status, IMSI, ICCID, PIN management

### Device.IP.

- Interface.{i}. — IP address configuration for LAN/WAN interfaces
- Interface.{i}.Stats. — Per-interface byte/packet counters

### Device.DeviceInfo.

- Manufacturer, ModelName, FirmwareVersion, HardwareVersion, SerialNumber
- UpTime, MemoryStatus (total/free RAM), ProcessStatus (CPU utilization)

### Device.ManagementServer.

- USP Controller URL, connection request credentials, periodic inform interval
- TLS certificate configuration for mutual authentication

## Common Integration Workflows

### 1. Zero-Touch Provisioning (ZTP)

1. CPE boots → loads bootstrap config from factory partition
2. Bootstrap config contains USP Controller URL + device certificate
3. USP agent establishes mTLS WebSocket to Controller
4. Controller sends GetSupportedDM → validates firmware version
5. Controller sends Set (WiFi SSID, APN, admin password, VoIP config)
6. Controller sends Set (ManagementServer.PeriodicInformInterval = 3600)
7. CPE sends Notify (Boot! event with device info)
8. CPE operational — subscriber receives welcome SMS/email

### 2. Bulk Firmware Upgrade

Controller → Agent: Operate (Device.DeviceInfo.FirmwareImage.Download())
  payload: { "URL": "https://firmware-cdn.operator.com/hl850q_v3.2.1.bin",
             "FileSize": 52428800,
             "TargetFileName": "/tmp/fw_update.bin" }

Agent → Controller: OperateResponse (Download progress via Notify events)
Agent → Controller: Notify (TransferComplete! event)

Controller → Agent: Operate (Device.DeviceInfo.FirmwareImage.Activate())
Agent → Controller: Notify (Boot! event — device rebooted with new firmware)

### 3. Real-Time Signal Monitoring

Controller → Agent: Set (Device.Cellular.Interface.1.Stats.X_HONLLY_PeriodicReport.Enable = true)
Controller → Agent: Set (Device.Cellular.Interface.1.Stats.X_HONLLY_PeriodicReport.Interval = 60)

Agent → Controller: Notify (every 60 seconds)
  payload: {
    "RSRP": -98,
    "RSRQ": -11,
    "RSINR": 18,
    "CellID": 12345678,
    "Band": "n78",
    "Bandwidth": 100,
    "MIMOLayers": 4,
    "DLThroughput": 452000000,
    "ULThroughput": 87000000
  }

## Security

- **Transport:** Mutual TLS (mTLS) with X.509 device certificates provisioned at manufacturing
- **Endpoint Authentication:** The USP agent validates the Controller's certificate against a pre-loaded CA bundle
- **Message Authentication:** USP E2E message signing via the USP Record's sec fields (MAC signature or full payload signing)
- **Role-Based Access Control:** Multi-controller support with per-controller permission scoping (an enterprise IT controller cannot access the operator's SIM management objects)

## Getting Started

1. **Obtain a USP Controller.** Compatible open-source options include OB-USP-Agent (Broadband Forum reference implementation) for testing and development. Production deployments typically use commercial ACS/USP platforms (Axiros, Friendly Technologies, Incognito, etc.).

2. **Configure the CPE bootstrap.** During manufacturing or via initial TR-069 provisioning, set the USP Controller URL and device certificate path on the CPE.

3. **Test connectivity.** Use the USP Controller's built-in connection test or the GetSupportedDM message to verify the CPE's USP agent is reachable and responsive.

## Resources

- Broadband Forum TR-369 USP Specification: https://www.broadband-forum.org/download/TR-369.pdf
- Broadband Forum TR-181 Device Data Model: https://device-data-model.broadband-forum.org/
- OB-USP-Agent (Reference Implementation): https://github.com/BroadbandForum/obuspa
- Honlly Telecom — 5G CPE Product Portfolio: https://www.honllytelecom.com/product/
- Honlly Telecom — USP/TR-369 Cloud Management Blog: https://honllytelecom.com/a-technical-buyers-guide-to-5g-cpe-cloud-management-and-remote-provisioning-tr-369-usp-zero-touch-deployment-and-fleet-management-at-scale/

## License

This documentation is provided for informational purposes. Honlly CPE firmware and USP agent implementations are proprietary. Contact Honlly Telecom for integration support, API documentation, and developer access.

---

*Maintained by Honlly Telecom | For integration inquiries: contact via honllytelecom.com*
