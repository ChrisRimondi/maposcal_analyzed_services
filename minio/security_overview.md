# Security Summary for MinIO Service

## 1. Service Overview

**Main purpose and functionality:**  
MinIO is an object storage service compatible with Amazon S3 APIs, offering high-performance, scalable storage with advanced features such as server-side encryption, identity federation, access management, and audit logging. It supports a variety of authentication and authorization mechanisms, enabling integration with corporate identity providers and external policy engines.

**Key architectural components:**  
- **Object Storage Layer:** Handles storage, retrieval, and management of objects, including erasure coding and encryption capabilities.
- **Identity and Access Management (IAM):** Built-in and pluggable authentication/authorization mechanisms, supporting users, groups, policies, and external identity providers.
- **Security Token Service (STS):** Issues temporary credentials via various authentication flows (OpenID Connect, LDAP/AD, TLS certificates, custom tokens).
- **Access Management Plugin:** Delegates authorization decisions to external HTTP(S) endpoints for every API call.
- **Encryption Engine:** Provides server-side encryption (SSE-S3, SSE-C) and integrates with external Key Management Systems (KMS).
- **Audit and Metrics Subsystem:** Collects audit logs, operational metrics, and supports compliance monitoring.
- **Monitoring and Metrics:** Prometheus-compatible metrics endpoints with configurable authentication.

**Technical stack and dependencies:**  
- Written in Go, leveraging Go standard libraries and select third-party packages (e.g., `github.com/golang-jwt/jwt/v4` for JWT handling).
- Supports integration with external systems: OpenID/OIDC providers, LDAP/AD, KMS, OPA (Open Policy Agent), Prometheus, etcd.
- Uses cryptographic primitives from Go's crypto library for credential generation, signature validation, and data encryption.

---

## 2. Authentication and Authorization

**Authentication mechanisms:**  
- **AWS Signature V4:** Supports signature-based authentication for S3-compatible APIs.
- **STS Temporary Credentials:** Issues temporary access keys, secret keys, and session tokens via multiple flows:
  - **AssumeRole (Signature-based):** Authenticates using existing MinIO credentials.
  - **AssumeRoleWithWebIdentity:** Authenticates with JWTs from OIDC-compatible providers (e.g., Keycloak, Dex, Casdoor).
  - **AssumeRoleWithClientGrants:** Authenticates via client credential grants (JWT access tokens from identity providers).
  - **AssumeRoleWithLDAPIdentity:** Authenticates AD/LDAP users using username/password.
  - **AssumeRoleWithCertificate:** Authenticates via client X.509/TLS certificates.
  - **AssumeRoleWithCustomToken:** Authenticates via opaque tokens verified by a custom Identity Management Plugin webhook.
- **Default Credentials:** Supports static access and secret keys for bootstrap and root access.

**Authorization models and policies:**  
- **MinIO IAM Policies:** Named access policies defined within MinIO and associated with users or groups.
- **Group-based Access:** LDAP/AD group memberships influence policy assignments.
- **Role-based Access:** Temporary credentials are associated with policies via role ARNs or claims in identity tokens.
- **OPA Integration:** Optional use of Open Policy Agent via webhook for external, dynamic policy evaluation.
- **Access Management Plugin:** Delegates all access control decisions to an external HTTP(S) endpoint, overriding internal IAM when enabled.

**Identity management:**  
- **Built-in Identities:** Users, groups, and service accounts managed internally (except when LDAP/AD is enabled).
- **External Identities:** Integration with AD/LDAP, OIDC, custom identity plugins, and X.509 certificates.
- **Automatic LDAP Sync:** Regularly synchronizes LDAP directory state, revoking credentials for removed users or altered group memberships.

**Session handling:**  
- **Temporary Credentials:** All STS flows issue credentials with explicit expiration (default 1 hour, configurable up to 365 days).
- **Session Tokens:** JWTs are generated and signed per session, containing claims and expiration.
- **Revocation:** LDAP sync and IAM policy changes can revoke or update active sessions; credentials for removed users are purged.

**Access control implementation:**  
- **Policy Attachment:** Policies can be attached directly to users or groups, or inferred from JWT claims or role ARNs.
- **Policy Evaluation:** Requests are evaluated against the union of applicable policies before granting access.
- **External Authorization:** When Access Management Plugin is active, every authenticated request is forwarded for allow/deny decision.

---

## 3. Encryption and Data Protection

**Data encryption at rest:**  
- **Server-Side Encryption (SSE):** Supports SSE-S3 (with master key or KMS) and SSE-C (client-provided keys).
- **Per-Object Keys:** Each object is encrypted with a unique, randomly generated object key.
- **Key Wrapping:** Object keys are never stored in plaintext; they are encrypted (sealed) with a key encryption key (KEK).
- **KMS Integration:** For SSE-S3, MinIO integrates with external KMS for key generation and unsealing.
- **Config/IAM Encryption:** Supports encrypting configuration and IAM assets using KMS-provided keys if enabled; otherwise, these are stored in plaintext.

**Data encryption in transit:**  
- **TLS Enforcement:** All management and API endpoints support TLS; client certificate authentication is available for STS.
- **Minimum TLS Version:** Defaults to TLS 1.2 or higher for secure communications.
- **Cipher Suite Preferences:** Can be configured for secure cipher suite selection.

**Key management:**  
- **Key Hierarchy:** Supports master keys, key encryption keys (KEK), and object encryption keys (OEK).
- **KMS APIs:** Expects KMS to provide key generation and unsealing functions for encrypted objects.
- **Key Rotation and Storage:** Details of key rotation depend on KMS implementation; MinIO does not store plaintext keys.

**Secure configuration:**  
- **Environment Variables & APIs:** Sensitive configuration (e.g., plugin endpoints, auth tokens, KMS settings) can be set via environment or admin API.
- **Config Integrity:** Configuration files are saved with SHA-256 hashes to ensure integrity.
- **Validation:** Strict input validation for encryption parameters, policy assignments, and endpoint configurations.

**Data handling and storage:**  
- **Erasure Coding:** Provides redundancy and protection against data loss.
- **Input Validation:** Strong validation on object sizes, multipart uploads, versioning, and encryption headers.
- **Metadata Protection:** Encryption metadata (IVs, sealed keys) stored alongside objects; never exposes raw keys.

---

## 4. Audit Logging and Monitoring

**Audit logging mechanisms:**  
- **Audit Logs:** Captures all API requests and authorization events, including user claims and policy actions.
- **Audit Metrics:** `/audit` endpoint exposes metrics related to audit functionality.
- **Message Queue Stats:** Collects metrics on audit log delivery (e.g., failed, total, pending messages).

**Log formats and structures:**  
- **Structured Logs:** Logs include contextual information such as user claims, actions, and outcomes.
- **Log Destinations:** Supports file and console log outputs; log configuration is part of the server config.

**Log retention policies:**  
- **Configurable Retention:** Log retention is configurable via server settings; not detailed in the available context.

**Monitoring systems:**  
- **Prometheus Integration:** Provides metrics endpoints for Prometheus, with authentication modes (`jwt` or `public`).
- **IAM Metrics:** Exposes IAM sync durations, successes, and failures as metrics.
- **Operational Metrics:** Exposes audit and object storage metrics for performance and compliance monitoring.

**Alert mechanisms:**  
- **Metric-based Alerts:** Prometheus metrics can be used to configure external alerting.
- **Audit Failures:** Failures in audit log delivery are tracked via metrics.

**Compliance reporting:**  
- **Audit Trail:** Comprehensive audit log and metrics support traceability and compliance verification.
- **Policy/Config Changes:** Changes to IAM policies, group memberships, and relevant config settings are logged.

---