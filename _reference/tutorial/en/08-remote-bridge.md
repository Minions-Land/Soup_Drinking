🌐 [中文版](../../tutorial/zh-CN/08-远程连接.md) | [← README](../../README.en.md)  
🗺️ **Navigation**: [← Ch.7](07-command-plugin-system.md) | **Chapter 8** | [Ch.9 →](09-design-patterns-summary.md)  
📁 [Tutorial EN](../../tutorial/en/) | [Toolkit EN](../../toolkit/en/) | [Reference](../../en/)

---

# Chapter 8: Remote Connection -- Bridging Local and Cloud

> Previous: [Chapter 7: Commands & Plugins](07-command-plugin-system.md) | Next: [Chapter 9: Design Patterns Summary](09-design-patterns-summary.md)

You have been using Claude Code on your laptop. It is fast, private, and powerful. Now imagine you are on your phone, or on a different computer, and you want to continue exactly where you left off -- same project, same conversation, same tools. How does a local CLI tool become accessible from anywhere?

This is the problem the **Bridge** solves. It connects your local Claude Code instance to `claude.ai/code`, turning your local terminal into a remotely controllable agent. Understanding this system teaches broadly applicable patterns for building resilient distributed architectures.

## The Core Idea: Local Execution, Remote Control

The Bridge does not move your code to the cloud. Your files stay on your machine. Your tools run locally. What the Bridge does is create a secure communication channel so that a remote interface (like claude.ai in your browser) can send commands to your local Claude Code and receive results back.

Think of it like a remote desktop, but only for your AI coding assistant. The keyboard and screen are in the cloud; the actual work happens on your machine.

The key source files live in `src/bridge/`:

| File | Purpose |
|------|---------|
| `bridgeApi.ts` | HTTP client for Environments and Sessions APIs |
| `bridgeMain.ts` | V1 environment-based connection flow |
| `remoteBridgeCore.ts` | V2 direct connection flow |
| `bridgeMessaging.ts` | Message serialization, deduplication, control requests |
| `bridgePointer.ts` | Local file storing session/environment IDs for crash recovery |

## Two Generations of Architecture

The Bridge has evolved through two generations, each solving the problem differently.

### V1: Environment-Based (Legacy)

The first generation used a concept called "environments." Your local CLI would register itself as an available environment with Anthropic's servers, then continuously poll for incoming work:

```
V1 Flow:
  Local CLI -> Register environment -> Poll for work -> Acknowledge
       ^                                                    |
       +----------------- Heartbeat -----------------------+
```

When work arrived (a user message from claude.ai), the CLI would process it, send back results, and resume polling. This is a classic **long-polling** pattern -- simple and reliable, but inherently wasteful because the CLI polls even when nothing is happening.

V1 used **Work Secrets** -- opaque connection strings that served as credentials for the environment.

### V2: Environment-Less (Direct)

The current generation eliminates the environment abstraction entirely:

```
V2 Flow:
  Local CLI -> POST /bridge -> Receive JWT token
            -> Open SSE stream (server pushes events to client)
            -> POST messages (client sends results to server)
```

V2 is simpler and more efficient. The CLI connects directly to a `/bridge` endpoint, receives a short-lived JWT, opens an SSE (Server-Sent Events) stream for receiving messages, and uses HTTP POST for sending messages back. No polling, no environment registration, no wasted requests.

## Session Lifecycle

Whether V1 or V2, every remote connection revolves around a **session**. A session represents one continuous interaction between a remote user and their local Claude Code:

```
Session Lifecycle:
  Create (POST /v1/sessions) -> Active (messages flowing) -> Archive (cleanup)
```

Sessions carry rich metadata: the project directory, git repo URL, current branch, and a human-readable title. This metadata lets the remote interface show you "Working on `my-app`, branch `feature/auth`" rather than just a session number.

### Crash Recovery: The Bridge Pointer

The most important design decision is **crash recovery**. The `bridgePointer.ts` module persists session IDs and environment IDs to a local file on disk. If Claude Code crashes or your laptop restarts:

1. On restart, Claude Code reads the pointer file
2. It finds the previous session ID
3. It reconnects to the same session instead of creating a new one
4. The conversation continues as if nothing happened

This "reconnect-in-place" strategy is essential. The person on claude.ai sees a brief disconnection, not a lost conversation.

## Transport Mechanisms: How Bytes Move

The Bridge uses multiple transport protocols, each optimized for different scenarios:

| Transport | Protocol | Direction | When Used |
|-----------|----------|-----------|-----------|
| **SSE** | Server-Sent Events | Server to Client | V2 reads |
| **WebSocket** | WS | Bidirectional | V1, Direct Connect |
| **HTTP POST** | HTTP | Client to Server | V2 writes |
| **Hybrid** | WS + POST | Mixed | V1 combined |

**Why not just WebSocket for everything?** Because reads and writes have different reliability requirements. Server-to-client messages benefit from SSE's built-in automatic reconnection. Client-to-server messages work better as individual HTTP POSTs that can be retried independently if they fail.

The `HybridTransport` combines both: it reads from a WebSocket (for low-latency server pushes) but writes via HTTP POST (for reliable delivery with retries). This acknowledges a fundamental asymmetry in distributed communication.

### Stream Resumption via Sequence Numbers

Every transport tracks a **sequence number** (`lastTransportSequenceNum`). If a connection drops, the client tells the server: "I last received message #47 -- send me everything from #48 onward." This enables seamless stream resumption without replaying the entire conversation.

## Authentication: Layers of Security

Remote control of a developer's machine is a high-stakes capability. The Bridge implements five overlapping authentication layers:

**OAuth (System Keychain)**: The foundational layer. Claude Code stores OAuth tokens in your system keychain (macOS Keychain, Linux secret service). These prove you are a legitimate Claude Code user.

**JWT Worker Tokens**: For V2 sessions, the `/bridge` endpoint issues short-lived JWT tokens scoped to a specific session. They expire quickly (typically minutes), limiting the blast radius of a compromised token.

**Work Secrets**: V1's simpler approach -- opaque connection strings that serve as both identifier and credential for an environment.

**Trusted Device Tokens**: An `X-Trusted-Device-Token` HTTP header that verifies the physical device, adding security beyond user credentials.

**Process-Level Security**: On Linux, the Bridge calls `prctl(PR_SET_DUMPABLE, 0)` to prevent other processes from reading its memory, protecting tokens from being scraped by malicious software.

## Token Refresh: Proactive, Not Reactive

The `TokenRefreshScheduler` class illustrates an important distributed systems principle. A naive approach waits until a request fails with "401 Unauthorized," then refreshes. This causes visible hiccups.

Claude Code's approach is proactive: calculate when the JWT will expire, subtract 5 minutes, and refresh before it is needed.

```
Token issued at T=0, expires at T=60min
  -> Scheduler sets refresh timer for T=55min
  -> At T=55min: new token fetched in background
  -> Old token still valid until T=60min
  -> Seamless transition, zero downtime
```

This pattern -- proactive credential renewal -- is applicable to any system that uses expiring tokens.

## The Upstream Proxy: Secure Containers

When Claude Code runs inside a secure container (Anthropic's cloud-hosted Claude Control REPL environment), it cannot directly access the internet. The upstream proxy in `src/upstreamproxy/` solves this:

```
Tool (curl/npm) -> HTTP CONNECT -> Local Relay (relay.ts)
  -> WebSocket -> Anthropic Server -> TLS MITM -> Destination
```

The `relay.ts` module creates a TCP-to-WebSocket relay on localhost. Tools think they are talking to a normal HTTP proxy. The relay tunnels traffic through WebSocket to Anthropic's server, which forwards it to the destination. Trusted domains (GitHub, npm registries) bypass the proxy via `NO_PROXY`.

## Structured IO: The Message Protocol

All communication uses **NDJSON** (Newline-Delimited JSON) through `src/cli/structuredIO.ts`. Two message types are critical:

- **`control_request`**: The local CLI needs a decision. "This tool wants to write to `/etc/hosts`. Allow?"
- **`control_response`**: The remote user's answer returns. "Yes, allow it."

NDJSON is ideal: human-readable for debugging, stream-friendly (parse each line as it arrives), and requires no framing beyond newlines.

## Session Recovery: Five Mechanisms Working Together

The Bridge uses five complementary mechanisms for resilience:

1. **Pointer files** -- Session IDs written to disk survive process crashes and machine restarts.
2. **Sequence numbers** -- Every message is numbered. On reconnection, only missed messages are replayed.
3. **Message deduplication** -- `BoundedUUIDSet` in `bridgeMessaging.ts` tracks recent message IDs and drops duplicates from network retries.
4. **Heartbeat management** -- Detects stale connections and triggers reconnection before the user notices.
5. **Epoch tracking** -- Ensures only the latest connection instance is active, preventing ghost sessions.

No single mechanism provides full resilience. Together, they create the illusion of a stable connection over an inherently unstable network.

## Key Takeaway: Resilient Distributed Architecture

> Build remote connections with the assumption that everything will fail. Use pointer files for crash recovery, sequence numbers for stream resumption, bounded UUID sets for deduplication, proactive token refresh for authentication continuity, and multiple transport protocols for network adaptability. Reliability in distributed systems is not a single feature -- it is a collection of small, complementary mechanisms that together create the illusion of stability.

The Bridge teaches us that resilience is not about preventing failures. It is about recovering from them so quickly and transparently that users never notice they happened.
---
📁 [← Tutorial Index](../../README.en.md#tutorial) | 🌐 [中文版](../../tutorial/zh-CN/08-远程连接.md) | [Ch.9 →](09-design-patterns-summary.md)
