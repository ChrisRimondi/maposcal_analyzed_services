# Security Summary for MinIO Service

## 1. Service Overview

### Main Purpose and Functionality
MinIO is a high-performance, distributed object storage service compatible with the Amazon S3 API. It is designed for cloud-native environments, supporting features such as server-side encryption, audit logging, identity federation, and fine-grained access control.

### Key Architectural Components
- **Object Storage Engine**: Manages data storage, erasure coding, and replication.
- **IAM Subsystem**: Provides authentication, authorization, user/group management, and policy enforcement.
- **STS (Security Token Service)**: Issues temporary credentials through multiple identity providers (OpenID, LDAP, custom plugins, TLS certificates, etc.).
- **Audit and Metrics System**: Records and exports audit logs and metrics to various backends (console, HTTP webhooks, Kafka).
- **Encryption/KMS Integration**: Supports server-side encryption (SSE-S3, SSE-C) and can integrate with external Key Management Systems.
- **Notification and Logging Targets**: Supports pluggable logging and notification endpoints including HTTP, Kafka, and console.

### Technical Stack and Dependencies
- **Primary Language**: Go
- **External Dependencies**: 
  - Kafka (for audit log delivery)
  - LDAP/AD servers (for identity federation)
  - Key Management Service (KMS) for encryption key management
  - Prometheus (for metrics)
- **Authentication Libraries**: JWT, OIDC, OAuth2, X.509 TLS
- **Logging**: JSON and structured logging, pluggable targets

---

## 2. Authentication and Authorization

### Authentication Mechanisms
- **Static Access/Secret Keys**: Default/minimum credentials for admin/root user access.
- **AWS Signature V4**: Used for API authentication (headers, query params, POST policies) with HMAC-SHA256 signatures and replay protection.
- **STS (Temporary Credentials)**:
  - **OpenID Connect/OAuth2**: JWT-based authentication via providers like Keycloak, Dex, Casdoor.
  - **LDAP/Active Directory**: User/password authentication, group synchronization, and credential issuance.
  - **Custom Identity Plugins**: Webhook-based external identity validation with opaque tokens.
  - **X.509/TLS Client Certificates**: Direct authentication using client certificates.
  - **Client Grants**: OAuth2 client credentials grant flow for machine-to-machine authentication.
- **Prometheus Metrics**: Supports `jwt` or `public` (no auth) modes for metric scraping endpoints.

### Authorization Models and Policies
- **IAM Policies**: JSON-based, named policies defining permissions at user/group/service account level.
- **Role Policy Association**: For STS users, policies are associated by:
  - Role ARN (for OpenID, custom plugins)
  - Claims in JWT tokens
  - LDAP group membership
- **Access Management Plugin**: Allows delegation of authorization decisions to an external webhook (e.g., OPA).
- **Bucket Policies**: S3-compatible policies for bucket/object-level access control.
- **ACLs**: Dummy implementation, only supports "private" with owner full control.

### Identity Management
- **Internal Users/Groups**: Managed via MinIO IAM subsystem.
- **LDAP/AD**: External source of truth for users/groups when configured.
- **STS Service Accounts**: Support for temporary and service accounts with expiration and session tokens.
- **Custom Identity Plugins**: External systems can provide identity assertions and policies via webhooks.

### Session Handling
- **STS Tokens**: Temporary credentials with configurable expiration (default 1 hour, min 15min, max 365 days).
- **JWT Tokens**: Used for session tokens, signed with server-side secret.
- **Session Token Validation**: Token expiration is enforced; expired tokens are purged.

### Access Control Implementation
- **Policy Evaluation**: Requests are authorized based on attached IAM/bucket policies, user/group membership, and external plugin responses.
- **Signature Validation**: Signature V4 ensures request authenticity and prevents tampering.
- **External Plugins**: Can override built-in policy evaluation when enabled.

---

## 3. Encryption and Data Protection

### Data Encryption at Rest
- **Server-Side Encryption (SSE-S3/SSE-C)**:
  - Each object is encrypted with a randomly generated ObjectKey, never stored in plaintext.
  - ObjectKey is sealed with a Key Encryption Key (KEK) derived from either a master key, KMS, or client-provided key.
  - Metadata includes IV, sealed key, and optionally KeyID for KMS integration.
- **Config/IAM Encryption**:
  - Configuration files and IAM assets can be encrypted using KMS-provided keys.

### Data Encryption in Transit
- **TLS/HTTPS**: Supported for all API endpoints, including audit log and notification targets.
- **mTLS**: Optional mutual TLS authentication for HTTP/Kafka endpoints.
- **TLS Client Auth**: Configurable for Kafka with certificate/key support.
- **TLS Skip Verification**: Can be enabled/disabled; defaults to verification enabled.

### Key Management
- **KMS Integration**: Used for root of trust in SSE-S3; supports key generation and key unsealing.
- **Client-Provided Keys**: SSE-C allows clients to supply encryption keys per object.
- **Key Rotation**: Supported via batch rotation jobs with encrypted key metadata.

### Secure Configuration
- **Credential Validation**: Enforces minimum lengths and disallows reserved characters.
- **Config Validation**: Structured parsing and validation of all configuration and credential fields.
- **Secrets Management**: Sensitive fields (tokens, passwords) are marked as secret in configuration.

### Data Handling and Storage
- **Erasure Coding**: Ensures durability and integrity of stored objects.
- **Bitrot Verification**: Periodic integrity checks using checksums.
- **Trash/Purge Mechanism**: Deleted objects may be moved to trash before permanent removal, depending on disk pressure.
- **Input Validation**: Enforced on volume, path, object names, and request headers.

---

## 4. Audit Logging and Monitoring

### Audit Logging Mechanisms
- **Structured Logging**: Captures comprehensive request/response metadata including:
  - API operation, bucket/object, user identity, request/response headers, timing, request IDs, deployment ID, claims, and tags.
- **Audit Targets**: Pluggable targets such as console, HTTP webhooks, Kafka; supports JSON format for logs.
- **Contextual Logging**: Includes IAM and session claims, service account info, and error details.

### Log Formats and Structures
- **JSON Format**: All audit logs are exported in structured JSON, capturing version, time, event, trigger, API details, remote host, request/response headers, and tags.
- **Audit Entry**: Includes context such as deployment ID, request ID, user agent, and claims for traceability.

### Log Retention Policies
- **Configurable Queues**: Logs can be queued in memory or on disk for reliable delivery, with batch sizes and retention periods configurable.
- **Durable Persistence**: Kafka and HTTP targets can be configured for persistent delivery and queueing.
- **No explicit retention policy**: in code, but can be enforced via external log management systems.

### Monitoring Systems
- **Prometheus Integration**: Exposes metrics for IAM syncs, authentication plugin success/failure, audit delivery status, and operational health.
- **Audit Metrics**: Tracks failed, queued, and total audit messages via `/audit` metrics endpoint.

### Alert Mechanisms
- **Operational Metrics**: Monitoring of audit log delivery failures, IAM sync failures, authentication plugin health, and storage health through Prometheus.
- **Configurable Alerts**: Relies on external systems (e.g., Prometheus Alertmanager, Kafka consumers) for alerting based on exposed metrics.

### Compliance Reporting
- **Audit Trails**: Detailed, structured logs support forensic analysis and compliance investigations.
- **Access/Change Logging**: Logs include all authenticated API calls, configuration changes, and policy updates.
- **Metrics**: IAM and audit metrics provide evidence of policy enforcement, access attempts, and system health for compliance audits.

---