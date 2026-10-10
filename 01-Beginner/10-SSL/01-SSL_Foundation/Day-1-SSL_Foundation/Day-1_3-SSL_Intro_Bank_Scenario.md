# Real Banking Scenario: SSL/TLS Handshake in Payment Architecture

In BankCell01's PaymentCluster, secure communication is critical during UPI transaction processing. This document outlines the end-to-end handshake flow, encryption parameters, and failure modes encountered in production.

---

## Architecture Overview

When a customer initiates a UPI transaction, the traffic routes through an IBM HTTP Server (IHS) reverse proxy termination layer before reaching backend core banking services.

```
+------------------+         HTTPS (Port 443)         +----------------------+
| Customer Browser | -------------------------------> | IHS (BankCell01)     |
|                  | <------------------------------- | PaymentCluster       |
+------------------+     SSL/TLS 1.2+ Handshake       +----------------------+
```

---

## Transaction Handshake Flow

The TLS session negotiation takes place within milliseconds prior to transmitting payment payloads:

1. **Connection Initiation**  
   The browser establishes a TCP connection to `https://pay.bankingapp.com` over port `443` (IBM HTTP Server).

2. **Server Identity Presentation**  
   IHS presents its X.509 SSL certificate, which contains the public key and server hostname information.

3. **Certificate Verification**  
   The browser verifies the certificate chain against trusted Root/Intermediate CAs (e.g., VeriSign/DigiCert).

4. **Session Key Exchange**  
   A pre-master secret / session key is exchanged using asymmetric encryption (**RSA-2048**).

5. **Payload Encryption**  
   Sensitive transaction parameters (amount, virtual payment address / VPA, account number, IFSC) flow encrypted via symmetric encryption (**AES-256**).

> [!NOTE]  
> The entire handshake completes in milliseconds. From the user interface, only the standard security indicator (🔒 padlock icon) is visible.

---

## Cryptographic Specifications

| Parameter | Mechanism / Standard | Purpose |
| :--- | :--- | :--- |
| **Edge Listener** | IHS Port `443` | Reverse proxy edge termination |
| **Trust Validation** | DigiCert / VeriSign CA Chain | Public trust verification |
| **Key Exchange** | RSA-2048 (Asymmetric) | Secure symmetric key negotiation |
| **Data Encryption** | AES-256-GCM / CBC (Symmetric) | Wire encryption for payment payload |

---

## Failure Modes & Production Risks

Operational and configuration anomalies at the reverse proxy or certificate level can lead to severe service disruption:

* **Key Pair Mismatch (Cluster Misconfiguration)**  
  * **Symptom:** Private key exists on Server A, but load-balanced traffic hits Server B without the synchronized key pair.  
  * **Impact:** Handshake termination failure (`SSL_ERROR_DECRYPT_ERROR_ALERT`).

* **Certificate Expiry**  
  * **Symptom:** Leaf or intermediate certificate lapses past its validity window.  
  * **Impact:** Browser displays high-risk untrusted warnings; payment abandonment surges and customer support load escalates.

* **Cipher Suite Discrepancy**  
  * **Symptom:** Mismatched TLS protocol versions or deprecated cipher configurations between client and IHS.  
  * **Impact:** `SSL_ERROR_NO_CYPHER_OVERLAP` / connection reset with no fallback path.

> [!TIP]  
> Enforce automated certificate lifecycle management (e.g., ACME or internal PKI monitoring), keep CMS key databases (`.kdb`) synchronized across clustered IHS nodes, and pin cipher suites to modern TLS recommendations.