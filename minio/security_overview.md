# Security Summary

## 1. Service Overview

### Main purpose and functionality

The service is MinIO, an object storage system built for compatibility with the Amazon S3 API. Its primary purpose is to provide scalable, high-performance storage with support for secure authentication, authorization, encryption, and compliance features. It supports identity federation, fine-grained access control, and advanced encryption for data at rest and in transit.

### Key architectural components

- **Object Storage Engine:** Manages buckets, objects, multipart uploads, versioning, and replication.
- **Identity and Access Management (IAM):** Handles users, groups, policies, and integration with external identity providers.
- **Security Token Service (STS):** Issues temporary credentials via various authentication mechanisms (LDAP, OIDC, client certificates, custom tokens).
- **Encryption Subsystem:** Provides server-side encryption with customer-provided keys (SSE-C), internal keys (SSE-S3), or external Key Management Service (KMS).
- **Audit Logging and Metrics:** Tracks access, configuration changes, and compliance metrics.
- **Access Management Plugins:** Supports external policy engines (e.g., OPA) and custom authorization via HTTP(S) webhooks.

### Technical stack and dependencies

- **Language:** Go
- **Core Libraries:** Standard Go libraries, cryptographic packages, and the MinIO internal libraries (e.g., `auth`, `crypto`).
- **Third-party integrations:** 
  - Identity providers: Keycloak, Dex, Casdoor, LDAP/AD
  - Policy engines: OPA (Open Policy Agent)
  - Metrics: Prometheus
  - KMS backends for key management

---

## 2. Authentication and Authorization

### Authentication mechanisms

MinIO supports multiple authentication mechanisms:

- **AWS Signature Version 4:** Used for S3-compatible API authentication (Authorization header, Query parameters, POST policies). Validates the request using HMAC and per-user access/secret keys.
- **STS APIs:** Provides temporary credentials after authenticating via:
  - **LDAP/AD:** Credentials verified against external directory. Temporary credentials issued upon successful authentication.
  - **Web Identity (OIDC):** Accepts JWT tokens from OIDC-compatible identity providers. JWTs are validated as part of the STS request.
  - **Client Grants:** Accepts access tokens from OAuth2 client grants.
  - **Custom Token Plugin:** Delegates token validation to an external webhook. The plugin responds with user and policy information, allowing MinIO to issue temporary credentials.
  - **TLS/X.509:** Supports client certificate-based authentication when enabled.
- **Root and Service Accounts:** Default credentials and service accounts managed locally or via IAM.

### Authorization models and policies

- **IAM Policies:** Access is controlled via named policies, which can be attached to users and groups.
- **Group-Based Authorization:** For LDAP/AD, policies can be assigned at the user or group level, with group memberships synced automatically.
- **Role Policies (STS):** Role-based policies can be attached to federated identities or roles, e.g., from OIDC providers or custom plugins.
- **Policy Enforcement:** MinIO evaluates all applicable policies for a user (including group and direct assignments) to allow or deny requests.
- **Access Management Plugin:** Optionally, all authorization decisions can be delegated to an external webhook, such as OPA, for centralized policy management.

### Identity management

- **Internal IAM:** Manages users, groups, and policies unless external identity sources are configured.
- **External Integration:** When LDAP/AD is configured, only external users (plus root) are allowed; internal user/group management is disabled except for information queries.
- **Identity Federation:** Supports OIDC, OAuth2, and custom token providers for federated access, with policy mapping based on role policies or token claims.

### Session handling

- **Temporary Credentials:** All STS-based authentications issue temporary credentials, consisting of access key, secret key, session token, and expiration timestamp.
- **Expiration:** Default credential lifetime is 1 hour, configurable between 15 minutes and 365 days for client grants.
- **Session Tokens:** JWT-based tokens signed using secret keys, containing claims such as access key, groups, and expiration.
- **Session Revocation:** For LDAP/AD, MinIO synchronizes group and user membership changes and revokes credentials if a user is deleted or removed from groups.

### Access control implementation

- **Policy Evaluation:** Policies are evaluated on every request using the IAM engine or delegated to the Access Management Plugin (e.g., OPA).
- **API Authorization:** Each API call checks the applicable policies to allow or deny the requested action, including for bucket and object operations.
- **Metrics Access:** Prometheus metrics endpoints can be configured for public or JWT-authenticated access.

---

## 3. Encryption and Data Protection

### Data encryption at rest

- **Server-Side Encryption (SSE):** All objects can be encrypted with unique, randomly generated keys (ObjectKey).
  - **SSE-C:** Uses client-provided keys to derive the encryption key for each object.
  - **SSE-S3:** Uses a master key or an external KMS. Object keys are derived from the master key or KMS-provided key.
- **Key Wrapping:** Object keys are never stored in plaintext. They are sealed (encrypted) with a Key Encryption Key (KEK) derived from either the client key (SSE-C) or the master/KMS key (SSE-S3).
- **Metadata:** Encrypted object keys, initialization vectors (IVs), and other relevant information are stored in object metadata.
- **IAM and Config Encryption:** MinIO supports encrypting configuration and IAM assets using KMS-provided keys; if KMS is disabled, this data is stored as plaintext.

### Data encryption in transit

- **TLS/SSL:** All network communication can be secured using TLS. Client certificate authentication is supported for mutual TLS (mTLS).
- **External Plugin Communications:** Webhooks and KMS endpoints can be configured to use HTTPS for secure communication.

### Key management

- **KMS Integration:** For SSE-S3, MinIO integrates with external KMS systems for key generation, sealing, and unsealing operations.
- **Key Lifecycle:** Object keys are generated per object and never stored unencrypted. Sealed keys are only decryptable with the appropriate KEK from KMS or client input.
- **Key Metadata:** For KMS-backed encryption, object metadata stores the key ID and encrypted key from the KMS.

### Secure configuration

- **Sensitive Parameters:** Sensitive configuration parameters (e.g., webhook tokens, KMS credentials) are handled via environment variables or secure configuration APIs.
- **Parameter Validation:** Input validation is performed for user keys, tokens, and session credentials (minimum and maximum length checks, presence of reserved characters).
- **Default Credentials:** Default access and secret keys are defined but can be overridden and should be secured appropriately.

### Data handling and storage

- **Integrity:** Checksums and hash algorithms are used for data and metadata integrity.
- **Versioning and Replication:** Object versioning, replication status, and delete markers are tracked for compliance and forensic analysis.
- **Erasure Coding:** Supports data redundancy and healing, with atomic updates to object metadata and audit logging for healing events.

---

## 4. Audit Logging and Monitoring

### Audit logging mechanisms

- **Action Auditing:** All access to sensitive operations (e.g., bucket encryption configuration, healing, policy changes) is logged, including user identity and request context.
- **Audit Metrics:** Dedicated metrics (e.g., `/audit`) track audit-related events and are exposed for monitoring.

### Log formats and structures

- **Structured Logging:** Logs include request identifiers, user identities, and contextual metadata for traceability.
- **Audit Event Fields:** For sensitive operations, logs capture before/after states, error codes, and external plugin responses when relevant.

### Log retention policies

- **Retention:** Log retention policies are not detailed in code, but audit logs are structured to support external log management systems for compliance.
- **Configurability:** Log verbosity and retention may be controlled via server configuration or external log aggregation tools.

### Monitoring systems

- **Prometheus Integration:** MinIO exports rich metrics for Prometheus, including IAM sync stats, audit metrics, and operational health.
- **Metric Authentication:** Prometheus endpoints can require JWT authentication or be made public, configurable via environment variable.

### Alert mechanisms

- **Error Logging:** Internal and upstream errors are logged with high severity, supporting alerting when audit or security events occur.
- **Audit Event Exposure:** Audit events and metrics can be used by external monitoring and SIEM solutions for alerting and compliance enforcement.

### Compliance reporting

- **Audit Trails:** Detailed and structured audit logs support compliance with regulatory requirements for data access and change tracking.
- **Policy Enforcement Logs:** Integration with external policy engines (e.g., OPA) and management plugins allows for externalized compliance reporting and attestation.
- **Access Revocation Logging:** Changes in user/group memberships and credential revocations (especially for LDAP/AD) are logged for auditability.

---