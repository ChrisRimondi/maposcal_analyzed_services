# Security Summary for NATS Server

## 1. Service Overview

**Main purpose and functionality:**  
The NATS Server is a cloud-native messaging system designed for high-performance publish/subscribe, request/reply, and streaming messaging. It supports multiple protocols (including MQTT) and integrates with JetStream for persistent messaging, streaming, and event processing.

**Key architectural components:**
- Core server supporting subjects, publish/subscribe, and request/reply.
- Pluggable authentication and authorization framework.
- Support for user accounts, multi-tenancy, and permissioned access.
- JetStream subsystem for streaming, persistence, and consumer management.
- Monitoring and metrics via HTTP endpoints and internal advisories.
- Clustering with secure routing, leader election, and state replication.
- Logging and audit subsystems with flexible backend options.
- Support for encrypted communication via TLS.

**Technical stack and dependencies:**
- Implemented in Go.
- Uses Go standard library for networking, cryptography, and concurrency.
- Employs bcrypt for password hashing.
- Integrates with standard logging (file, syslog, Windows event log).
- Utilizes cryptographic primitives for JWT, NKey, and OCSP handling.
- Supports external authentication callouts and directory/HTTP-based account resolvers.

## 2. Authentication and Authorization

**Authentication mechanisms:**
- Supports multiple authentication methods:
  - Username/password (with bcrypt hashing).
  - Token-based authentication.
  - JWT/NKey authentication for users and accounts.
  - TLS client certificate mapping.
  - External authentication callouts (with JWT/nkey and encrypted challenge/response).
- Enforces nonce-based challenge-response for NKey/JWT to prevent replay attacks.
- Integration with OCSP/CRL for certificate validation.
- Custom authentication hooks are supported.

**Authorization models and policies:**
- Role-based permissions at user and account levels.
- Fine-grained publish/subscribe permissions, including allow/deny lists and wildcard matching.
- Service import/export permissions with subject scoping, cycle detection, and activation token support.
- Response permissions with optional TTL and message limits.
- Scoped response templates for dynamic reply authorization.
- System supports default permissions and role templates.
- Cluster and route authorization with per-route credentials and account mapping.

**Identity management:**
- User and account definitions in configuration, supporting multiple accounts (multi-tenant).
- JWT-based account and user representations, including revocation and expiration.
- Support for account resolvers (directory or HTTP) to fetch and validate JWTs.
- User revocation and account expiration enforced via timers and state flags.

**Session handling:**
- MQTT and JetStream sessions are persisted and managed using streams and durable consumers.
- Session uniqueness and clean session semantics enforced.
- Session context includes TLS info and authentication details for auditing.

**Access control implementation:**
- Permission checks performed on each publish/subscribe operation.
- Deny rules take precedence over allow.
- Account and user bindings enforced at connection and operation time.
- Clustered environments enforce permissions across routes and peer nodes.
- Export/import cycles and scope mismatches are detected and blocked.
- Authentication failures generate advisory events and are logged for auditing.

## 3. Encryption and Data Protection

**Data encryption at rest:**
- JetStream supports encrypted file storage using selectable ciphers.
- Windows platforms support TPM-based key management for hardware-rooted encryption.
- Encryption keys are sealed and stored with TPM or on disk (protected by TPM password).
- JetStream supports encrypted message storage (cipher configuration present).

**Data encryption in transit:**
- TLS is supported for all client and inter-server connections.
- Configurable TLS with certificate pinning, client certificate verification, and custom cipher suites.
- OCSP stapling and CRL endpoints supported for certificate revocation checking.
- Optional enforcement of client certificate authentication.
- Compression negotiation for inter-server routes, with fallback to uncompressed as needed.

**Key management:**
- TPM-backed key generation and sealing on Windows for JetStream encryption.
- Directory permissions (0750) set for persisted keys.
- Key rotation and sealing policies enforced via TPM session controls.
- NKey seeds are handled securely and intended for restricted access.

**Secure configuration:**
- Strong input validation for configuration files (duplicate checks, value ranges, subject formats).
- Warnings and errors logged for insecure configurations (e.g., plaintext passwords).
- Default file permissions for logs set to 0640.
- TLS and account configuration strictly parsed and validated on startup.

**Data handling and storage:**
- JetStream and MQTT use streams for persistent storage, with configurable retention and limits.
- Message deduplication via message IDs and duplicate windows.
- File and memory storage types available, with state snapshotting and recovery features.
- All sensitive operations (e.g., snapshot, restore, delete) require permission checks and are audited.

## 4. Audit Logging and Monitoring

**Audit logging mechanisms:**
- Supports logging to file, syslog, Windows event log, and standard error.
- Log rotation with configurable size and backup limits.
- Debug, trace, info, warn, error, and fatal levels available.
- Logging of authentication failures, permission violations, configuration reloads, and server events.
- Internal advisories and event publishing for JetStream, account, and client activities.
- Rate limiting and structured event logging to prevent flooding and support compliance.

**Log formats and structures:**
- Structured log messages with timestamp, PID, severity, and formatted content.
- Audit advisories and JetStream events encoded as JSON for machine parsing.
- Logs include context (user, account, client ID, subject, event type) for traceability.

**Log retention policies:**
- Configurable via log file rotation and backup purging.
- Default log file permissions set to restrict unauthorized access.
- Audit events retained in JetStream streams as configured.

**Monitoring systems:**
- Extensive HTTP monitoring endpoints for connections, routes, accounts, JetStream, and raft clusters.
- Metrics include authentication status, user/account details, JWT and tag exposure, and certificate info.
- Internal event system tracks server health, client connections, and configuration reloads.
- Advisory publishing for administrative, lifecycle, and cluster leadership events.
- Connection lifecycle and slow consumer tracking via ring buffers.

**Alert mechanisms:**
- Fatal errors (e.g., failed log rotation, syslog connection) trigger process termination or panic for immediate alerting.
- Log warnings and errors for failed authentication, authorization, or configuration issues.
- Event advisories published for significant incidents (e.g., account expiration, stream/consumer changes).

**Compliance reporting:**
- Audit trails maintained via advisories, logging, and persistent event streams.
- Detailed error codes and API error objects support traceability and consistent compliance reporting.
- Monitoring endpoints provide compliance-relevant introspection data (revocation status, account validity, etc.).
- Log access and retention controls depend on external file system and log management solution configuration.

---

This summary is based strictly on the provided code and documentation context.