# Security Summary

## 1. Service Overview

- **Main Purpose and Functionality**
  - The service is a high-performance messaging system supporting publish/subscribe, queue groups, streaming, and MQTT interoperability.
  - It includes advanced features such as JetStream (streaming and persistence), clustering, and multi-tenant accounts.
  - The service is designed for reliability, extensibility, and secure communication between distributed systems.

- **Key Architectural Components**
  - **Core Server**: Manages client connections, message routing, authorization, and clustering.
  - **JetStream**: Provides persistent streams, consumers, and advanced message delivery guarantees.
  - **Account System**: Supports multi-tenant isolation and scoped permissions.
  - **Websocket/HTTP2 Layer**: Allows access via WebSockets and plans for HTTP2 support.
  - **TLS/Certificate Store Integration**: Provides secure transport using standard files and OS certificate stores (notably Windows).
  - **Audit and Advisory System**: Publishes internal events and advisories for monitoring and compliance.

- **Technical Stack and Dependencies**
  - **Language**: Go (multiple files under Apache License 2.0).
  - **Crypto Libraries**: Standard Go crypto, `golang.org/x/crypto`, nkeys, JWT.
  - **HTTP/WS**: Go standard library, with WebSocket frame handling and compression.
  - **Compression**: `github.com/klauspost/compress/flate` for websocket compression.
  - **Other Dependencies**: `github.com/nats-io/jwt/v2`, `github.com/nats-io/nkeys`, `github.com/nats-io/nats.go`.
  - **Operating System Integration**: Windows certificate store support; utilities for BSD/macOS process stats.
  - **Third-party Audit**: Security audit performed by Trail of Bits (April 2025).

---

## 2. Authentication and Authorization

- **Authentication Mechanisms**
  - **JWT-Based Authentication**: Uses ed25519-signed JWTs for operators, accounts, and users. JWTs are parsed and validated for each connection.
  - **NKEY Authentication**: Supports NKEYs for cryptographic identity (ed25519 key pairs).
  - **Username/Password**: Supports both plaintext and bcrypt-hashed passwords.
  - **Token Authentication**: Accepts tokens as an authentication mechanism.
  - **TLS Client Certificates**: Integrates with OS certificate stores (Windows) to authenticate using X.509 certificates.
  - **Websocket Authentication**: JWT and credential cookies supported for websocket clients.

- **Authorization Models and Policies**
  - **Account-based Authorization**: Each account is isolated with its own permissions and subject space.
  - **User/Role Permissions**: Users can have granular publish/subscribe allow/deny lists.
  - **Resource Limits**: Enforces limits on connections, subscriptions, payload sizes, and more.
  - **Multi-Tenant Isolation**: Subject spaces and streams are isolated per account.
  - **Cluster and Route Authorization**: Cluster and leaf node connections are authenticated and authorized using credentials or tokens.

- **Identity Management**
  - **JWT Claims Validation**: Parses and validates claims for operator, account, and user identities.
  - **NKEY Seed Management**: Seeds and keys for users are treated as sensitive and must be protected.
  - **Certificate Identity**: X.509 subject and issuer matching, OCSP support for peer validation.

- **Session Handling**
  - **Session Persistence**: MQTT and streaming sessions are persisted in streams with per-session records.
  - **Session Policies**: Session limits (e.g., MaxMsgsPer) are defined for MQTT streams.
  - **Connection Tracking**: Activity time and connection metadata (start time, uptime, total connections) are tracked.

- **Access Control Implementation**
  - **Permission Enforcement**: Publish/subscribe permissions enforced per user/account.
  - **Dynamic Access Checks**: On each operation, access is checked against current permissions and account status.
  - **Connection Closure on Violation**: Protocol escalations and permission violations can result in connection closure (see TODOs).
  - **Clustered Authorization**: Cluster and gateway connections verify peer identity and permissions.

---

## 3. Encryption and Data Protection

- **Data Encryption at Rest**
  - **JetStream Persistence**: Message data, session state, and retained messages are stored, with no explicit mention of encryption at rest in the provided context.
  - **Sensitive Material Handling**: NKEY seeds and JWTs are recommended to be treated as secrets.

- **Data Encryption in Transit**
  - **TLS Support**: Full support for SSL/TLS, with configurable ciphers and curve preferences.
  - **Websocket TLS**: Websocket endpoints require TLS by default unless explicitly disabled.
  - **Certificate Store Integration**: Can use OS-backed certificate stores for private key and certificate management (notably on Windows).
  - **Cipher Suite Management**: Explicit cipher suite and curve preference configuration is available for TLS connections.

- **Key Management**
  - **NKEY and JWT Management**: Strong cryptographic keys for JWTs and NKEYs, with routines for secure seed wiping.
  - **OS Certificate Store**: On Windows, the service can retrieve and use private keys securely from the system store, not requiring direct access to key files.
  - **Certificate Matching**: Supports certificate selection via subject, issuer, or thumbprint.
  - **Error Handling**: Robust error reporting for key management failures (e.g., inability to extract private key, unsupported algorithms).

- **Secure Configuration**
  - **Configuration Validation**: Extensive config validation to prevent conflicts, enforce correct types, and ensure proper setup of authentication and authorization.
  - **TLS Options**: Enforces the use of TLS for websocket endpoints unless explicitly allowed.
  - **Blacklist/ERR Escalation**: TODOs indicate planned features for escalating permission errors to connection termination.

- **Data Handling and Storage**
  - **Stream Policies**: Streams have limits and retention policies (e.g., limits on retained messages, maximum messages per subject).
  - **Session and State Storage**: Session and retained messages are managed in dedicated streams.
  - **Input Validation**: Subject, payload, and protocol validation are enforced during client operations.

---

## 4. Audit Logging and Monitoring

- **Audit Logging Mechanisms**
  - **Advisory Events**: Publishes JetStream advisories and metrics (e.g., stream/consumer actions, message acknowledgments, leader elections) as internal messages.
  - **Connection and Activity Tracking**: Tracks and exposes connection activity, dropped messages, and resource usage via monitoring endpoints.

- **Log Formats and Structures**
  - **Structured JSON**: Advisories and metrics are marshaled as JSON objects with typed event schemas.
  - **Metrics and State**: Monitoring endpoints expose connection, stream, and server stats in a structured format.

- **Log Retention Policies**
  - **Stream Retention**: Advisory and metric events are published into streams and are subject to stream retention and limits policies.
  - **No Explicit Log Rotation**: No mention of file-based log retention or rotation in provided context.

- **Monitoring Systems**
  - **Internal Monitoring Endpoints**: `/varz`, `/connz`, `/routez` endpoints for live state and metrics.
  - **Metrics Tracking**: Tracks server uptime, connection count, dropped messages, and more.

- **Alert Mechanisms**
  - **Advisory Publication**: Internal advisories can be consumed by monitoring or alerting systems.
  - **Resource Limit Alerts**: Events are published when resource limits are hit (e.g., API queue limit, server out of space).
  - **Error and Warning Logging**: Warnings and errors are logged for failed advisories and configuration issues.

- **Compliance Reporting**
  - **Third-party Audit**: A security review was performed by Trail of Bits (April 2025) and is publicly available.
  - **Structured Event History**: JetStream advisories and metrics provide an auditable event history for compliance.
  - **Configuration Validation**: Strict validation and error reporting for security-sensitive configuration errors.

---

**Note:** All information is derived from the provided code and documentation context; no external assumptions are made.