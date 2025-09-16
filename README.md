Got it 👍 I’ll drop the content here in plain chat (no MD fences).

---

# RSA + Azure Key Vault + Kafka Messaging – Architecture Blueprint

This document explains how **App A (Pega)** and **App B (BDRP)** exchange Kafka messages securely using **RSA encryption** and **Azure Key Vault**. It is organized into chapters.

---

### Chapter 1 – Azure Infrastructure Setup

* Create **Subscription** and **Resource Group** (`rg-rsa-keyvault-prod`).
* Provision **Azure Key Vault** (`kv-rsa-msg-prod`), Premium tier for HSM-backed keys.
* Enable **Soft Delete + Purge Protection** and **Diagnostic Logs**.
* Configure networking:

  * App A (Pega) → **Private Endpoint**.
  * App B (BDRP) → **Public Endpoint + IP whitelist**.
* Set up identities: Managed Identity / Service Principal.
* Assign permissions: App A = `decrypt`, App B = `get`.
* Enable logging & monitoring (Log Analytics, alerts).

---

### Chapter 2 – RSA Key Pair Creation

* Generate **RSA-2048 (or 4096)** key pair inside Key Vault.
* Private key = **non-exportable**.
* Public key = exportable (PEM / JWK).
* Enable **rotation policy** (e.g., 12 months).
* Allow overlap during key rotation.
* Permissions: App A = `decrypt`, App B = `get`.

---

### Chapter 3 – Public Key Distribution (App B = BDRP)

* **Direct fetch**: App B uses Service Principal to call Key Vault REST API.
* **Managed Identity**: if App B runs in Rabobank’s Azure tenant.
* **Indirect sharing**: App A exports public key and shares securely (SFTP, Blob SAS, Teams PGP).
* Enforce **IP whitelisting**: allow only outbound NAT IPs from App B.

---

### Chapter 4 – App B (BDRP) Encrypts Kafka Messages

* Fetch public key (once or periodically).
* Encrypt sensitive fields with RSA public key.
* Publish encrypted payload to Kafka:

  ```json
  { "header": { "accountId": "12345" }, "payload": "ENCRYPTED_BLOB_BASE64" }
  ```

---

### Chapter 5 – App A (Pega) Consumes & Decrypts Kafka Messages

* Subscribe to Kafka topic, receive encrypted message.
* Extract `payload`, decode Base64 → byte array.
* Call **Key Vault Cryptography API** (algorithm = RSA-OAEP).
* Key Vault decrypts internally and returns plaintext.
* Process decrypted data in Pega workflow.

---

### Chapter 6 – Runtime Security & Governance

* **Identity governance**: App A = decrypt only, App B = get only.
* **Network governance**: Private Endpoint for App A, Public + Firewall for App B.
* **Key governance**: rotation, overlap, notifications to App B.
* **Audit & monitoring**: enable logs, track all operations, set alerts.

---

### ✅ End Result

* **App B (BDRP)** encrypts with the public key and publishes messages to Kafka.
* **App A (Pega)** consumes, decrypts using Key Vault private key, and processes the data.
* **Azure Key Vault** is the custodian of RSA keys (private key never leaves KV).
* Security ensured with **RBAC, IP whitelisting, rotation, monitoring**.

-
