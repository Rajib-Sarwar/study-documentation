# Technical Architecture Document: Secure Mobile Payment & Hardware Subsystem

**Author:** Chowdhury Md Rajib Sarwar

**Role Focus:** Senior Software Engineer (SE5) – Systems & Architecture

**Architecture Focus:** Hardware Integration • PCI-DSS Security • State Machine Design • Bluetooth LE

## 1. Executive Summary

To enable in-field payment processing for a high-volume mobility platform, I architected a decoupled, delegation-based MVC subsystem integrating the Worldpay triPOS® Mobile SDK and BBPOS Chipper hardware. This system successfully bridges the native iOS Bluetooth stack with proprietary backend payment APIs to process EMV (chip), contactless (NFC), and swipe transactions. By engineering a strict 3-way transaction handshake, dynamic credential mapping, and QuickChip optimization, the architecture guarantees absolute PCI-DSS compliance while reducing physical transaction latency to under two seconds.

## 2. High-Level Architecture

The subsystem utilizes a strict unidirectional data flow, ensuring that physical hardware events (Bluetooth/EMV) are safely mapped through native controllers before executing secure backend settlements.

```text
┌────────────────────────┐      ┌─────────────────────────┐      ┌────────────────────────┐
│ BBPOS Hardware Reader  │      │ iOS Bluetooth LE Stack  │      │ Worldpay triPOS SDK    │
│ (EMV / NFC / Swipe)    ├─────►│ (CoreBluetooth)         ├─────►│ (sharedVtp Singleton)  │
└────────────────────────┘      └─────────────────────────┘      └──────────┬─────────────┘
                                                                            │ (Encrypted Payload)
┌───────────────────────────────────────────────────────────────────────────▼─────────────┐
│                                iOS CLIENT PAYMENT SUBSYSTEM                             │
│                                                                                         │
│  ┌──────────────────────┐      ┌────────────────────────┐      ┌─────────────────────┐  │
│  │ triPOSConfig         │◄─────┤ Base Controller Layer  ├─────►│ Device Manager      │  │
│  │ (Dynamic Keys/EMV)   │      │ (SDK Lifecycle / HUD)  │      │ (BLE Scan & Pair)   │  │
│  └──────────────────────┘      └──────────┬─────────────┘      └─────────────────────┘  │
│                                           │ (Inherits)                                  │
│                                           ▼                                             │
│                                ┌────────────────────────┐                               │
│                                │ Payment Controller     │                               │
│                                │ (3-Way Handshake flow) │                               │
│                                └──────────┬─────────────┘                               │
└───────────────────────────────────────────┼─────────────────────────────────────────────┘
                                            │ (Reference # & Settlement)
                                            ▼
                                 ┌────────────────────────┐
                                 │ Backend API Gateway    │
                                 │ (Proprietary APIs)     │
                                 └────────────────────────┘

```

## 3. Pillar 1: Dynamic Configuration & Security Initialization

**The Challenge:** Hardcoding merchant credentials or maintaining persistent access tokens on a mobile client introduces severe PCI compliance risks and makes scaling across multiple white-label tenants impossible.

**The Solution:** I engineered a dynamic configuration manager (`triPOSConfig`). Instead of hardcoding API keys, the client performs a secure backend handshake (`getTriPOSDeviceConfig`) to dynamically parse Acceptor IDs, Account Tokens, and Terminal IDs. The configuration layer strictly enforces `VTPDeviceTypeBbPos`, blocks manual keyed entry, and isolates all initialization logic within `ExpressBaseViewController`.

**Business Impact:** The application achieves absolute PCI-DSS compliance by ensuring raw cardholder data (PAN, CVV, Track 2) is never serialized or stored on the device. Data is encrypted at the point of interaction (POI) on the hardware reader and transmitted directly to the Worldpay gateway.

## 4. Pillar 2: Hardware State & Connection Management

**The Challenge:** Bluetooth Low Energy (BLE) connections in field environments are inherently unstable. The mobile client must gracefully handle unexpected disconnects, device discovery, and real-time hardware updates without hanging the UI.

**The Solution:** I built `ExpressDeviceManageViewController` to monitor CoreBluetooth state changes via `BluetoothStateProtocol`. It dynamically categorizes scan results and handles Over-The-Air (OTA) hardware firmware updates. I integrated `JGProgressHUD` into the Base Controller to strictly block user interactions during critical I/O operations (e.g., config injection or firmware pushes), preventing race conditions.

**Business Impact:** Drivers experience a seamless, self-healing connection process. If a physical disconnect occurs during a transaction, robust error-mapping (`ErrorToMessageConverter`) safely guides the driver to retry or switch payment methods without abandoning the active session.

## 5. Pillar 3: The 3-Way Transaction State Machine

**The Challenge:** Processing payments in a mobility environment requires handling both temporary holds (Pre-Authorization) and immediate charges (Direct Sale). Network drops during a charge can result in double-charging or lost revenue.

**The Solution:** I architected `ExpressPaymentController` around a deterministic, 3-way handshake protocol:

1. **Prepare:** The client hits `startTransactionUrl` to lock the transaction and obtain a unique backend `reference_number`.
2. **Authorize/Charge:** The client commands the triPOS SDK to execute `processAuthorizationRequest` or `processSaleRequest`.
3. **Settle:** Upon gateway approval, the client maps the payload into a `TransactionCompleteObj` and posts to `completeTransactionUrl` to finalize the backend state.

**Performance Optimization:** I enabled Worldpay's QuickChip protocol (`quickChipEnabled = YES`). This allows the EMV handshake to complete on the hardware reader in under 2 seconds without waiting for the full gateway network round-trip, significantly accelerating the checkout process for drivers.
