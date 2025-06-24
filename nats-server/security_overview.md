# Security Summary

## 1. Service Overview

**Main Purpose and Functionality**  
The service is a messaging server (NATS Server) supporting publish/subscribe and streaming paradigms, with features for clustering, client connections, monitoring, and JetStream persistence. It is designed for high performance and supports protocols such as MQTT and HTTP monitoring endpoints.

**Key Architectural Components**
- Core message broker with support for clustering, gateways, leafnodes, and JetStream (streaming and persistence).
- Authentication and authorization modules with support for multiple mechanisms (JWT, Nkey, username/password, tokens, TLS certificate mapping).
- Monitoring and observability endpoints exposing server, connection, and JetStream metrics via HTTP.
- Internal subsystems for event/advisory publishing, audit logging, and system statistics.
- Pluggable storage backends, including file and memory, with support for encryption at rest.
- Security audit completed by Trail of Bits and commissioned by OSTIF (April 2025).

**Technical Stack and Dependencies**
- Implemented in Go.
- Utilizes Go’s standard crypto, TLS, HTTP, and syscall libraries.
- Integrates with external authentication services (callouts).
- Supports bcrypt for password hashing.
- Uses pseudo-random number generation and secure nonce creation.
- Optional TPM support for key management (on Windows).
- JetStream storage with optional encryption (AES/ChaCha).

---

## 2. Authentication and Authorization

**Authentication Mechanisms**
- Username/password (with bcrypt hash support).
- Token-based authentication.
- Nkey authentication (public/private key pairs).
- JWT-based authentication for users and operators, supporting claim-based access and account scoping.
- TLS client certificate authentication with support for pinned certificates and subject mapping.
- External authentication via configurable callouts.
- Nonce-based challenge-response to prevent replay attacks, with cryptographically secure random nonces.

**Authorization Models and Policies**
- Role-based access control (RBAC) specified in configuration files, supporting fine-grained publish/subscribe permissions with allow/deny lists.
- Scoped permissions for JetStream APIs and MQTT operations.
- JWT claims enforce account-level limits (connections, subscriptions, payload sizes) and temporal access constraints.
- Cluster and route authentication/authorization with separate credentials.
- Import/export rules for streams and services with cycle detection and token-based approvals.

**Identity Management**
- Users defined in configuration files, JWTs, or external sources.
- JWT user and operator claims validated and parsed at connection time, including issuer, audience, subject, and expiration checks.
- TLS certificate subject mapping allows for identity linkage.
- Nkey and JWT credentials support for signing and proof of identity.
- Revocation and expiration checks on JWTs and activation tokens.
- Secure wiping of sensitive JWT data from memory after use.

**Session Handling**
- Session management for MQTT with persisted records in JetStream streams.
- Session restoration and duplicate detection for MQTT QoS 1/2.
- Timeouts for authentication, authorization, and idle connections.
- Nonce state managed with mutexes for thread safety.
- Explicit timers for authentication, session expiration, and ping/pong liveness checks.

**Access Control Implementation**
- Permissions enforced at connection and per-operation (publish/subscribe, API calls).
- Authorization filters support wildcards, allow/deny lists, and dynamic reply tracking.
- JetStream APIs and monitoring endpoints enforce account checks and subject validation.
- Configuration supports default and per-user permissions with inheritance.
- Advisory events and logging for permission violations and authentication failures.

---

## 3. Encryption and Data Protection

**Data Encryption at Rest**
- JetStream file storage supports encryption using server-configured keys (AES/ChaCha).
- Optional TPM integration (on Windows) for secure key storage and management.
- Sensitive JWT and credential data securely wiped from memory.

**Data Encryption in Transit**
- TLS support for all client, cluster, and monitoring endpoints, with configurable minimum TLS versions.
- Configurable cipher suites and curve preferences.
- Mutual TLS enforcement with client certificate verification, OCSP stapling, and revocation checking.
- Certificate pinning and subject/issuer mapping for enhanced trust.

**Key Management**
- Support for loading server keys from files or TPM modules (Windows only).
- Certificate and key files specified per endpoint; CA files and verification options configurable.
- OCSP-based revocation checks with cache management and logging.
- Cryptographically secure random number generation for nonces and tokens.

**Secure Configuration**
- Strict configuration parsing with input validation for permissions, subject patterns, and security options.
- Detection and warning for duplicate or conflicting user/account definitions.
- Extensive option validation for deprecated or conflicting security fields.
- Secure defaults for TLS and authorization timeouts.
- Panic recovery in configuration parsing to avoid partial/insecure states.

**Data Handling and Storage**
- In-memory and persistent storage of connection metadata, session state, and advisory logs.
- JetStream state and configuration versioning to prevent unauthorized or stale data propagation.
- Secure handling of sensitive data in logs (e.g., password redaction).
- Internal memory structures guarded with atomic operations or mutexes where appropriate.

---

## 4. Audit Logging and Monitoring

**Audit Logging Mechanisms**
- Comprehensive event/advisory publishing for authentication, authorization, connection lifecycle, JetStream/stream/consumer events, and system errors.
- Detailed logging of authentication failures, permission violations, and validation issues.
- Structured log formats for advisory and trace events, including timestamps, event types, and metadata.
- Internal tracking of closed client connections and connection history for forensic analysis.
- Warnings for plaintext passwords and insecure configurations.

**Log Formats and Structures**
- JSON marshaling for structured audit/advisory events.
- Per-connection and per-account logs including authentication details, JWT claims, and TLS info.
- Trace and debug logs for MQTT, JetStream, and protocol parsing with sampling controls.

**Log Retention Policies**
- Configurable logging output to file, syslog, or stdout with size limits and rotation support.
- PID and log file management for process auditing.
- No explicit log retention duration specified in code, but persistent log files and advisory streams for JetStream.

**Monitoring Systems**
- HTTP monitoring endpoints exposing connection, route, account, subscription, JetStream, gateway, and health stats with filtering, pagination, and query parameter validation.
- Metrics on resource usage (CPU, memory), connection/authentication status, and OCSP cache stats.
- JetStream API access and cluster state monitoring via advisories and internal events.
- System events for OCSP validation failures and certificate issues.

**Alert Mechanisms**
- Advisory events for critical authentication failures, certificate revocation, JetStream/stream/consumer events, and cluster changes.
- Internal rate-limiting of advisories to prevent alert flooding.
- Log-based alerts for warnings and errors related to security and compliance.

**Compliance Reporting**
- System and account audits through monitoring endpoints and advisory logs.
- Support for structured event publishing to external monitoring/audit systems.
- Compliance with certificate revocation checking (OCSP), secure credential handling, and audit-friendly logging.
- Third-party security audit conducted with published results (Trail of Bits, April 2025).

---