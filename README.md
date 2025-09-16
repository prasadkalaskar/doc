# RSA + Azure Key Vault + Kafka Messaging – Architecture Blueprint

This document describes how **App A (Pega)** and **App B (BDRP)** securely exchange Kafka messages using **RSA encryption** and **Azure Key Vault**.

---

## Chapter 1 – Azure Infrastructure Setup (App A = Pega)

- **Subscription & Resource Group**  
  - Provision a dedicated subscription.  
  - Create a Resource Group (e.g., `rg-rsa-keyvault-prod`).  

- **Key Vault Provisioning**  
  - Create Azure Key Vault (e.g., `kv-rsa-msg-prod`).  
  - Use Premium tier for HSM-backed keys.  
  - Enable Soft Delete + Purge Protection.  
  - Enable diagnostic logs for audit.  

- **Networking**  
  - Public Endpoint: restrict with firewall/IP whitelisting.  
  - Private Endpoint: recommended for App A (Pega).  
  - Decision: App A → Private Endpoint, App B → Public Endpoint + IP whitelist.  

- **Identity Setup**  
  - App A (Pega): Managed Identity (if in Azure) or Service Principal.  
  - App B (BDRP): existing Service Principal in Rabobank Azure AD.  

- **Access Control**  
  - App A (Pega): `decrypt` permission.  
  - App B (BDRP): `get` permission (public key only).  
  - Roles: App A → `Key Vault Crypto User`, App B → `Key Vault Reader`.  

- **Logging & Monitoring**  
  - Enable Diagnostic Settings to Log Analytics / Azure Monitor.  
  - Audit each call (decrypt/get).  
  - Alerts for unauthorized access.  

---

## Chapter 2 – RSA Key Pair Creation

- **Key Generation**  
  - Generate RSA-2048 (or 4096) key inside Key Vault.  
  - Private key = non-exportable.  
  - Public key = exportable (PEM / JWK).  

- **Versioning & Rotation**  
  - Enable automatic rotation policy (e.g., 12 months).  
  - Maintain overlapping versions during cutover.  
  - Notify App B (BDRP) when new public key is available.  

- **Permissions**  
  - App A (Pega): `decrypt`.  
  - App B (BDRP): `get`.  
  - No team touches the private key.  

---

## Chapter 3 – Public Key Distribution (App B = BDRP)

- **Direct Fetch from Key Vault**  
  - App B authenticates with its Service Principal in Rabobank Azure AD.  
  - Calls REST API:  
    ```http
    GET https://kv-rsa-msg-prod.vault.azure.net/keys/myRSAKey?api-version=7.4
    ```  
  - RBAC + IP firewall enforced.  

- **Managed Identity**  
  - If App B runs workloads in Rabobank Azure tenant, assign a Managed Identity.  
  - Grant `get` permission on Key Vault.  

- **Indirect Sharing**  
  - App A exports public key once.  
  - Share via secure channel (Teams PGP, SFTP, Azure Blob SAS).  
  - Reshare on rotation.  

- **IP Whitelisting**  
  - Request outbound egress NAT IP(s) from App B architects.  
  - Add IPs (single or CIDR) to Key Vault firewall.  
  - Combine with RBAC for layered security.  

---

## Chapter 4 – App B (BDRP) Encrypts Kafka Messages

- **Fetching Public Key**  
  - Retrieve once or periodically.  
  - Store securely.  

- **Encrypting Payload**  
  - Encrypt sensitive fields with RSA public key.  
  - Produce Base64 blob.


## Chapter 5 – App A (Pega) Consumes & Decrypts Kafka Messages

- **Kafka Consumption**  
  - App A (Pega) subscribes to the Kafka topic and receives the encrypted message.  

- **Extract & Prepare Ciphertext**  
  - Extract the encrypted payload (`payload` field) from the Kafka message.  
  - Decode the Base64 string into a byte array for decryption.  

- **Decrypt via Key Vault**  
  - App A (Pega) calls the **Azure Key Vault Cryptography API** with:  
    - Key URI (e.g., `https://kv-rsa-msg-prod.vault.azure.net/keys/myRSAKey/<version>`)  
    - Algorithm (`RSA-OAEP`)  
    - Ciphertext byte array  
  - Azure Key Vault uses the **private key internally** within its HSM and returns the decrypted plaintext.  

- **Business Processing**  
  - Convert the plaintext bytes into a string/JSON.  
  - Pass the decrypted data back into the Pega workflow for business logic.  
  - Implement error handling for:  
    - Key version mismatch  
    - Corrupted ciphertext  
    - Unauthorized access errors  

---

## Chapter 6 – Runtime Security & Governance

- **Identity Governance**  
  - App A (Pega): `decrypt` only.  
  - App B (BDRP): `get` only.  

- **Network Governance**  
  - App A (Pega): access Key Vault via **Private Endpoint**.  
  - App B (BDRP): access Key Vault via **Public Endpoint + IP whitelist**.  

- **Key Governance**  
  - Define a regular key rotation schedule (e.g., 12 months).  
  - Allow temporary overlap between old and new keys to prevent downtime.  
  - Notify App B (BDRP) when a new public key is published.  

- **Audit & Monitoring**  
  - Enable Key Vault diagnostic logs.  
  - Monitor who accessed `decrypt` and `get` operations.  
  - Create alerts for unauthorized or failed access attempts.  

---

## Sequence Diagram

```plantuml
@startuml
title RSA + Azure Key Vault + Kafka Messaging Flow

actor "App B (BDRP)" as B
participant "Kafka Topic" as K
actor "App A (Pega)" as A
participant "Azure Key Vault" as KV

B -> KV: (Optional) Fetch Public Key (GET /keys/myRSA)
KV --> B: Public Key (RSA)

B -> B: Encrypt payload with Public Key
B -> K: Publish Kafka Message {"payload":"ENCRYPTED_BLOB"}

A -> K: Subscribe & Read Message
K --> A: Kafka Message with Encrypted Payload

A -> A: Extract & Base64-decode Encrypted Blob
A -> KV: Decrypt(ciphertext, key=myRSA)
KV --> A: Plaintext Payload

A -> A: Process decrypted data in Pega workflow

@e
    

- **Publishing Kafka Message**  
  ```json
  {
    "header": { "accountId": "12345" },
    "payload": "ENCRYPTED_BLOB_BASE64"
  }
