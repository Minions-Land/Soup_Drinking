# Remote Connection (Bridge) & Server

> 🗺️ **Navigation**: [← README](../README.md) | [zh-CN version](../zh-CN/07-远程连接.md)  
> 📁 [Reference Index](../README.md#reference) | [Tutorial](../tutorial/) | [Toolkit](../toolkit/)

> Sources: `src/bridge/`, `src/server/`, `src/upstreamproxy/`, `src/assistant/`

## 1. Bridge Architecture

The "Bridge" connects local Claude Code instances to Anthropic's remote control infrastructure (claude.ai/code).

### Two Generations

| Version | Name | Flow |
|---------|------|------|
| V1 | Env-based (Legacy) | Register → Poll for work → Acknowledge → Heartbeat |
| V2 | Env-less (Direct) | Direct `/bridge` endpoint → SSE/POST |

### Key Components

| File | Purpose |
|------|---------|
| `bridgeApi.ts` | HTTP client for Environments & Sessions APIs |
| `bridgeMain.ts` | V1 environment-based flow |
| `remoteBridgeCore.ts` | V2 env-less direct flow |
| `replBridge.ts` | REPL-specific bridge orchestration |
| `bridgeMessaging.ts` | Message serialization, dedup (`BoundedUUIDSet`), control requests |
| `bridgePointer.ts` | Local file pointer for session/env ID recovery |
| `bridgeConfig.ts` | Bridge configuration |
| `bridgeUI.ts` | Bridge status UI |
| `bridgePermissionCallbacks.ts` | Permission routing |

## 2. Session Management

```
Session Lifecycle:
  Create (POST /v1/sessions) → Active → Archive (cleanup)
```

- **Metadata**: Project dir, git repo URL, branch, human-readable title
- **Recovery**: `bridgePointer` enables crash recovery by persisting session IDs locally
- **Perpetual**: Sessions can survive restarts; "reconnect-in-place" to existing environment

## 3. Transport Mechanisms

| Transport | Protocol | Direction | Used In |
|-----------|----------|-----------|---------|
| SSE | Server-Sent Events | Server→Client | V2 reads |
| WebSocket | WS | Bidirectional | V1, Direct Connect |
| HTTP POST | HTTP | Client→Server | V2 writes (Session Ingress) |
| Hybrid | WS+POST | Mixed | V1 combined |

Sequence number management (`lastTransportSequenceNum`) for stream resumption.

## 4. Authentication & Security

| Layer | Mechanism | Purpose |
|-------|-----------|---------|
| OAuth | System keychain tokens | CLI-to-Backend auth |
| JWT (Worker Tokens) | Short-lived tokens via `/bridge` | V2 session auth |
| Work Secrets | Opaque connection strings | V1 environment credentials |
| Trusted Device | `X-Trusted-Device-Token` header | Device verification |
| Token Refresh | `TokenRefreshScheduler` | Proactive JWT refresh (5min before expiry) |
| Process Security | `prctl(PR_SET_DUMPABLE, 0)` | Prevent memory scraping (Linux) |

## 5. Upstream Proxy

For Claude Code running in secure containers (CCR) to reach external services:

```
Tool (curl/npm) → HTTP CONNECT → Local Relay
  → WebSocket → Anthropic Server
  → TLS MITM → Destination
```

| File | Purpose |
|------|---------|
| `upstreamproxy.ts` | Container-side proxy initialization |
| `relay.ts` | TCP-to-WebSocket relay on localhost |

- **MITM**: Server terminates WS, does TLS MITM with downloaded CA cert
- **`NO_PROXY`**: Internal/common domains (GitHub, npm) bypass proxy

## 6. Server Module

`src/server/directConnectManager.ts` — "Direct Connect" sessions:
- Simplified WebSocket communication
- Control request handling (permissions)
- Interrupt signal management
- Used for local debugging and SDK integrations

## 7. Session History

`src/assistant/sessionHistory.ts`:
- `fetchLatestEvents()` — Paged session events from backend
- `fetchOlderEvents()` — History pagination
- Message high-water mark tracking
