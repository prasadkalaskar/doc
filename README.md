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

- **Publishing Kafka Message**  
  ```json
  {
    "header": { "accountId": "12345" },
    "payload": "ENCRYPTED_BLOB_BASE64"
  }
