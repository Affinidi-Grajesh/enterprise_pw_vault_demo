# MCP over DIDComm: A New Standard for Secure AI Agent Communication

**A repeatable pattern for enterprise-grade AI agent integration with end-to-end encryption and decentralized identity**

---

## 🎯 The Problem

AI agents (like Claude Desktop, custom agents) need to access enterprise backends securely. Current approaches have limitations:

| Approach | Issue |
|----------|-------|
| **Direct API calls** | Credentials exposed in agent config |
| **HTTP/REST** | Network vulnerabilities, MITM attacks |
| **MCP over HTTP** | Passwords in plaintext on localhost |
| **Custom protocols** | No standardization, hard to audit |

---

## 💡 The Solution: MCP over DIDComm

Combine two powerful standards:
- **MCP (Model Context Protocol)**: Standard for AI agent ↔ tool communication
- **DIDComm**: Secure, encrypted messaging with decentralized identity

**Result**: Secure, standardized, verifiable AI agent communication

---

## 🏗️ Architecture Overview

### High-Level Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     AI AGENT LAYER                                      │
│                                                                          │
│  ┌──────────────┐         ┌──────────────────────────────┐            │
│  │   AI Agent   │         │  Agent Identity Manager      │            │
│  │   (Claude,   │────────>│  - DID (did:peer:2.Ez6LS...) │            │
│  │   Custom)    │         │  - Keys (OS Keychain)        │            │
│  └──────┬───────┘         └──────────────────────────────┘            │
│         │                                                               │
│         │ stdio/JSON-RPC                                               │
│         │                                                               │
└─────────┼───────────────────────────────────────────────────────────────┘
          │
          ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                   MCP CLIENT LAYER                                       │
│                                                                          │
│  ┌──────────────────────────────────────┐                              │
│  │  MCP Client (uses DIDComm transport) │                              │
│  │  - Reads agent DID from keychain     │                              │
│  │  - Wraps MCP in DIDComm envelopes    │                              │
│  │  - Sends to tool server DID          │                              │
│  └──────────────┬───────────────────────┘                              │
│                 │                                                        │
│                 │ DIDComm (encrypted)                                   │
│                 │ Message type: https://mcp.didcomm.org/jsonrpc/1.0     │
│                 │                                                        │
└─────────────────┼────────────────────────────────────────────────────────┘
                  │
                  ↓ [Through Mediator - Firewall Friendly]
                  │
┌─────────────────┼────────────────────────────────────────────────────────┐
│                 │          MCP TOOL SERVER LAYER                         │
│                 │                                                         │
│  ┌──────────────┴──────────────────────────────┐                        │
│  │  MCP Tool Server (listens via DIDComm)     │                        │
│  │  - Has own DID (did:peer:2.Ez6LS...)       │                        │
│  │  - Receives MCP wrapped in DIDComm         │                        │
│  │  - Unwraps, processes with ServerHandler   │                        │
│  │  - Returns MCP response via DIDComm        │                        │
│  └──────────────┬──────────────────────────────┘                        │
│                 │                                                         │
│                 │ Internal DIDComm (to backend)                          │
│                 │                                                         │
└─────────────────┼─────────────────────────────────────────────────────────┘
                  │
                  ↓
┌─────────────────┼─────────────────────────────────────────────────────────┐
│                 │         ENTERPRISE BACKEND                              │
│                 │                                                          │
│  ┌──────────────┴──────────────────────┐                                │
│  │  Password Service / Business Logic  │                                │
│  │  - Has own DID                       │                                │
│  │  - DIDComm only access               │                                │
│  │  - Pre-configured data               │                                │
│  └──────────────────────────────────────┘                                │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔄 Message Flow Diagram

### Step-by-Step Communication

```
Agent                MCP Client          Mediator         Tool Server         Backend
  │                     │                    │                │                 │
  │                     │                    │                │                 │
  ├─ 1. Initialize ────>│                    │                │                 │
  │   (stdio/JSON-RPC)  │                    │                │                 │
  │                     │                    │                │                 │
  │                     ├─ 2. DIDComm msg ──>│                │                 │
  │                     │   (encrypted)      │                │                 │
  │                     │                    │                │                 │
  │                     │                    ├─ 3. Route ────>│                 │
  │                     │                    │                │                 │
  │                     │                    │                │                 │
  │                     │                    │<─ 4. Response ─┤                 │
  │                     │                    │   (encrypted)  │                 │
  │                     │                    │                │                 │
  │                     │<─ 5. DIDComm msg ─┤                │                 │
  │<─ 6. MCP response ─┤                    │                │                 │
  │   (stdio/JSON-RPC)  │                    │                │                 │
  │                     │                    │                │                 │
  │                     │                    │                │                 │
  ├─ 7. Call tool ─────>│                    │                │                 │
  │   get_password      │                    │                │                 │
  │                     │                    │                │                 │
  │                     ├─ 8. DIDComm msg ──>│                │                 │
  │                     │   {jsonrpc:        │                │                 │
  │                     │    "tools/call"}   │                │                 │
  │                     │                    │                │                 │
  │                     │                    ├─ 9. Route ────>│                 │
  │                     │                    │                │                 │
  │                     │                    │                ├─ 10. Query ───>│
  │                     │                    │                │   (DIDComm)     │
  │                     │                    │                │                 │
  │                     │                    │                │<─ 11. Data ────┤
  │                     │                    │                │   (encrypted)   │
  │                     │                    │                │                 │
  │                     │                    │<─ 12. MCP ─────┤                 │
  │                     │                    │   response     │                 │
  │                     │                    │                │                 │
  │                     │<─ 13. DIDComm ────┤                │                 │
  │<─ 14. Result ──────┤                    │                │                 │
  │   Password!         │                    │                │                 │
  │                     │                    │                │                 │
```

### Legend
- **Solid lines** (───): Standard communication
- **DIDComm**: End-to-end encrypted, authenticated
- **stdio/JSON-RPC**: Local process communication
- **Mediator**: Routes DIDComm messages, cannot decrypt

---

## 📦 Component Diagram

### Three-Layer Architecture

```
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ LAYER 1: AI AGENT                                                     ┃
┃ ┌──────────────────────────────────────────────────────────────────┐ ┃
┃ │ Claude Desktop / Custom Agent                                    │ ┃
┃ │                                                                   │ ┃
┃ │ Responsibilities:                                                 │ ┃
┃ │ • User interaction                                                │ ┃
┃ │ • Generate MCP requests                                           │ ┃
┃ │ • Manage agent DID (create, store in OS keychain)                │ ┃
┃ │ • Configure MCP servers                                           │ ┃
┃ │                                                                   │ ┃
┃ │ Config:                                                           │ ┃
┃ │ {                                                                 │ ┃
┃ │   "mcpServers": {                                                 │ ┃
┃ │     "password-vault": {                                           │ ┃
┃ │       "transport": "didcomm",                                     │ ┃
┃ │       "toolServerDid": "did:peer:2.Ez6LS...",                    │ ┃
┃ │       "mediator": "did:web:mediator.example.com"                  │ ┃
┃ │     }                                                             │ ┃
┃ │   }                                                               │ ┃
┃ │ }                                                                 │ ┃
┃ └──────────────────────────────────────────────────────────────────┘ ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
                                    │
                                    │ stdio/JSON-RPC
                                    ↓
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ LAYER 2: MCP CLIENT (Agent-Side)                                     ┃
┃ ┌──────────────────────────────────────────────────────────────────┐ ┃
┃ │ MCP Client with DIDComm Transport                                │ ┃
┃ │                                                                   │ ┃
┃ │ Responsibilities:                                                 │ ┃
┃ │ • Implement MCP protocol (initialize, tools/list, tools/call)    │ ┃
┃ │ • Load agent DID from keychain                                   │ ┃
┃ │ • Wrap MCP JSON-RPC in DIDComm envelopes                         │ ┃
┃ │ • Encrypt using agent's keys                                      │ ┃
┃ │ • Send to tool server DID via mediator                           │ ┃
┃ │ • Receive and unwrap DIDComm responses                           │ ┃
┃ │                                                                   │ ┃
┃ │ Code:                                                             │ ┃
┃ │ let transport = DIDCommTransport::new(                            │ ┃
┃ │     agent_did,                                                    │ ┃
┃ │     agent_secrets,                                                │ ┃
┃ │     tool_server_did,                                              │ ┃
┃ │     mediator                                                      │ ┃
┃ │ );                                                                │ ┃
┃ │ let service = mcp_client.serve(transport).await?;                │ ┃
┃ └──────────────────────────────────────────────────────────────────┘ ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
                                    │
                                    │ DIDComm (E2E encrypted)
                                    │ Type: https://mcp.didcomm.org/jsonrpc/1.0
                                    ↓
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ LAYER 3: MCP TOOL SERVER (Enterprise-Side)                           ┃
┃ ┌──────────────────────────────────────────────────────────────────┐ ┃
┃ │ MCP Tool Server with DIDComm Listener                            │ ┃
┃ │                                                                   │ ┃
┃ │ Responsibilities:                                                 │ ┃
┃ │ • Listen for DIDComm messages                                     │ ┃
┃ │ • Verify sender DID (authentication)                              │ ┃
┃ │ • Decrypt DIDComm envelope                                        │ ┃
┃ │ • Extract MCP JSON-RPC payload                                    │ ┃
┃ │ • Process with standard ServerHandler                             │ ┃
┃ │ • Call backend services via DIDComm                               │ ┃
┃ │ • Wrap response in DIDComm                                        │ ┃
┃ │ • Send back to requesting agent DID                               │ ┃
┃ │                                                                   │ ┃
┃ │ Tools Exposed:                                                    │ ┃
┃ │ • get_password(key: string)                                       │ ┃
┃ │ • list_keys()                                                     │ ┃
┃ │ • [custom enterprise tools]                                       │ ┃
┃ └──────────────────────────────────────────────────────────────────┘ ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
                                    │
                                    │ DIDComm (business protocol)
                                    ↓
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ LAYER 4: ENTERPRISE BACKEND                                           ┃
┃ ┌──────────────────────────────────────────────────────────────────┐ ┃
┃ │ Password Service / Database / Business Logic                     │ ┃
┃ │                                                                   │ ┃
┃ │ • DIDComm-only access                                             │ ┃
┃ │ • No HTTP endpoints                                               │ ┃
┃ │ • Zero trust - all requests authenticated via DID                │ ┃
┃ └──────────────────────────────────────────────────────────────────┘ ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
```

---

## 🔐 Security Stack Visualization

### Defense in Depth

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SECURITY LAYERS                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Layer 1: Identity (DID)                                            │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │ • Decentralized Identifiers (did:peer)                     │    │
│  │ • No central authority                                      │    │
│  │ • Agent controls private keys                               │    │
│  │ • Cryptographically verifiable                              │    │
│  └────────────────────────────────────────────────────────────┘    │
│                            ↓                                         │
│  Layer 2: Authentication                                            │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │ • DID-based authentication (no passwords)                  │    │
│  │ • Every message cryptographically signed                    │    │
│  │ • Sender verification automatic                             │    │
│  │ • No shared secrets to compromise                           │    │
│  └────────────────────────────────────────────────────────────┘    │
│                            ↓                                         │
│  Layer 3: Encryption (DIDComm)                                      │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │ • End-to-end encryption                                     │    │
│  │ • Forward secrecy                                           │    │
│  │ • Mediator cannot decrypt                                   │    │
│  │ • Multiple encryption algorithms supported                  │    │
│  └────────────────────────────────────────────────────────────┘    │
│                            ↓                                         │
│  Layer 4: Protocol (MCP)                                            │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │ • Standard JSON-RPC 2.0                                     │    │
│  │ • Well-defined capabilities                                 │    │
│  │ • Tool-based access control                                 │    │
│  │ • Auditable requests/responses                              │    │
│  └────────────────────────────────────────────────────────────┘    │
│                            ↓                                         │
│  Layer 5: Key Management                                            │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │ • OS-level key storage (Keychain/DPAPI/Secret Service)     │    │
│  │ • Never in config files                                     │    │
│  │ • Encrypted at rest                                         │    │
│  │ • Hardware security module support                          │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 📊 Comparison: Traditional vs MCP over DIDComm

### Traditional HTTP/REST Approach

```
Agent ────[HTTP/API Key]───> Enterprise API
              │
              │ Issues:
              │ • API key in config file
              │ • No encryption (or TLS only)
              │ • Centralized authentication
              │ • IP-based access control
              │ • Credentials can be copied
```

### MCP over HTTP (Current)

```
Agent ──[stdio/JSON-RPC]──> MCP Server ──[HTTP plaintext]──> Bridge ──[DIDComm]──> Backend
                                              │
                                              │ Issues:
                                              │ • Passwords in plaintext
                                              │ • Local network exposure
                                              │ • No authentication
                                              │ • MITM possible
```

### MCP over DIDComm (Proposed)

```
Agent ──[stdio/JSON-RPC]──> MCP Client ──[DIDComm E2E encrypted]──> Tool Server ──[DIDComm]──> Backend
                                              │
                                              │ Benefits:
                                              │ ✅ End-to-end encryption
                                              │ ✅ DID-based auth
                                              │ ✅ No plaintext
                                              │ ✅ Decentralized
                                              │ ✅ Verifiable
```

---

## 🎯 Key Benefits Matrix

| Feature | Traditional API | MCP/HTTP | MCP/DIDComm |
|---------|----------------|----------|-------------|
| **Standard Protocol** | ❌ Custom | ✅ MCP | ✅ MCP |
| **Encryption** | ⚠️ TLS only | ❌ Plaintext | ✅ E2E DIDComm |
| **Authentication** | ⚠️ API keys | ❌ None | ✅ DID-based |
| **Decentralized** | ❌ Central auth | ❌ No | ✅ Yes |
| **Firewall Friendly** | ❌ Needs ports | ❌ Needs ports | ✅ Via mediator |
| **Credential Exposure** | ❌ In config | ❌ On network | ✅ Never exposed |
| **Verifiable** | ❌ No | ❌ No | ✅ Crypto signed |
| **Key Management** | ⚠️ Files | ⚠️ Files | ✅ OS keychain |
| **Zero Trust** | ⚠️ Partial | ❌ No | ✅ Yes |
| **Auditability** | ⚠️ Server logs | ⚠️ Distributed | ✅ Crypto proof |

---

## 🚀 Implementation Pattern

### Repeatable Steps for Any Enterprise Integration

#### Step 1: Define Your Tools

```json
{
  "tools": [
    {
      "name": "get_password",
      "description": "Retrieve password from vault",
      "inputSchema": {
        "type": "object",
        "properties": {
          "key": { "type": "string" }
        }
      }
    }
  ]
}
```

#### Step 2: Create MCP Tool Server

```rust
// Generic pattern - works for any backend
struct MyToolServer {
    our_did: String,
    our_secrets: Vec<Secret>,
    backend_did: String,
}

impl ServerHandler for MyToolServer {
    async fn call_tool(&self, request: CallToolRequestParam) -> Result<CallToolResult> {
        // 1. Validate request
        // 2. Forward to backend via DIDComm
        // 3. Return result
    }
}

// Listen via DIDComm
let transport = DIDCommTransport::new(...);
let service = server.serve(transport).await?;
```

#### Step 3: Configure Agent

```json
{
  "mcpServers": {
    "your-service": {
      "transport": "didcomm",
      "toolServerDid": "did:peer:2.Ez6LS...",
      "mediator": "did:web:mediator.example.com"
    }
  }
}
```

#### Step 4: Agent Initializes DID

```rust
// One-time setup
let (agent_did, agent_secrets) = generate_did(mediator)?;
save_to_keychain("agent_did", &agent_did)?;
save_to_keychain("agent_secrets", &agent_secrets)?;
```

**Done!** Agent can now securely call enterprise tools.

---

## 📈 Deployment Topologies

### Topology 1: Single Enterprise Backend

```
           ┌─────────┐
           │ Agent 1 │
           └────┬────┘
                │
┌─────────┐    │    ┌──────────────┐    ┌──────────┐
│ Agent 2 │────┼───>│ MCP Tool     │───>│ Password │
└─────────┘    │    │ Server       │    │ Service  │
               │    └──────────────┘    └──────────┘
┌─────────┐    │
│ Agent 3 │────┘
└─────────┘

Multiple agents → One tool server → One backend
```

### Topology 2: Multi-Service Enterprise

```
                    ┌──────────────┐    ┌──────────┐
                ┌──>│ MCP Tool     │───>│ Password │
                │   │ Server A     │    │ Service  │
                │   └──────────────┘    └──────────┘
┌─────────┐     │
│  Agent  │─────┤   ┌──────────────┐    ┌──────────┐
└─────────┘     ├──>│ MCP Tool     │───>│ Database │
                │   │ Server B     │    │ Service  │
                │   └──────────────┘    └──────────┘
                │
                │   ┌──────────────┐    ┌──────────┐
                └──>│ MCP Tool     │───>│ CRM      │
                    │ Server C     │    │ Service  │
                    └──────────────┘    └──────────┘

One agent → Multiple tool servers → Multiple backends
```

### Topology 3: Multi-Tenant SaaS

```
┌─────────────┐    ┌──────────────┐    ┌────────────────┐
│ Agent       │───>│ MCP Tool     │───>│ Tenant A       │
│ (Tenant A)  │    │ Server       │    │ Backend        │
└─────────────┘    │              │    └────────────────┘
                   │              │
┌─────────────┐    │  (Routes     │    ┌────────────────┐
│ Agent       │───>│   based on   │───>│ Tenant B       │
│ (Tenant B)  │    │   DID)       │    │ Backend        │
└─────────────┘    │              │    └────────────────┘
                   │              │
┌─────────────┐    │              │    ┌────────────────┐
│ Agent       │───>│              │───>│ Tenant C       │
│ (Tenant C)  │    │              │    │ Backend        │
└─────────────┘    └──────────────┘    └────────────────┘

Multi-tenant routing via DID-based identity
```

---

## 🔄 Message Format Specification

### DIDComm Envelope Structure

```json
{
  "type": "https://mcp.didcomm.org/jsonrpc/1.0",
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "from": "did:peer:2.Ez6LSghwSE437wnDE1pt3X6hVDUQzSjsHzinpX3XFvMjRAm7y",
  "to": "did:peer:2.Ez6LSbysY2xFMRpGMhb7tFTLMpeuPRaqaWM1yECx2AtzE3KCc",
  "created_time": 1698765432,
  "expires_time": 1698765732,
  "body": {
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "get_password",
      "arguments": {
        "key": "myapp"
      }
    }
  }
}
```

### Encryption Process

```
┌──────────────────┐
│ MCP JSON-RPC     │ 1. Start with standard MCP message
└────────┬─────────┘
         ↓
┌──────────────────┐
│ Wrap in DIDComm  │ 2. Add sender/receiver DIDs
│ Body             │
└────────┬─────────┘
         ↓
┌──────────────────┐
│ Sign with        │ 3. Cryptographically sign
│ Sender's Key     │
└────────┬─────────┘
         ↓
┌──────────────────┐
│ Encrypt with     │ 4. Encrypt for receiver
│ Receiver's Key   │
└────────┬─────────┘
         ↓
┌──────────────────┐
│ Send via         │ 5. Route through mediator
│ Mediator         │
└──────────────────┘
```

---

## 🎓 Integration Examples

### Example 1: Claude Desktop Integration

**Agent Side:**
```json
// ~/.config/Claude/claude_desktop_config.json
{
  "mcpServers": {
    "enterprise-vault": {
      "transport": "didcomm",
      "toolServerDid": "did:peer:2.Ez6LSbysY2xFMRpGMhb7tFTLMpeuPRaqaWM1yECx2AtzE3KCc",
      "mediator": "did:web:mediator.affinidi.com"
    }
  }
}
```

**Enterprise Side:**
```bash
# Start tool server
cargo run --bin mcp-tool-server -- \
  --did-config tool-server-config.json \
  --backend-did "did:peer:2.Ez6LS..."
```

**User Experience:**
```
User: "Claude, get me the password for myapp"
Claude: [Calls tool via DIDComm]
Result: "Password for 'myapp': secret123"
```

### Example 2: Custom Python Agent

```python
from mcp_didcomm import DIDCommTransport, MCPClient
from did_manager import AgentDID

# Load or create agent DID
agent = AgentDID.from_keychain()

# Create MCP client with DIDComm transport
transport = DIDCommTransport(
    agent_did=agent.did,
    agent_secrets=agent.secrets,
    tool_server_did="did:peer:2.Ez6LS...",
    mediator="did:web:mediator.example.com"
)

client = MCPClient(transport)

# Use tools
await client.initialize()
result = await client.call_tool("get_password", {"key": "myapp"})
print(result)
```

---

## 📊 Performance Characteristics

### Latency Breakdown

```
Total: ~200-400ms (typical)

┌─────────────────────────────────────┐
│ Component             │ Latency     │
├───────────────────────┼─────────────┤
│ Agent → MCP Client    │ <1ms        │ (in-process)
│ DIDComm Encryption    │ 5-10ms      │ (crypto ops)
│ Network to Mediator   │ 50-100ms    │ (depends on location)
│ Mediator Routing      │ 10-20ms     │ (lookup & forward)
│ Network to Tool       │ 50-100ms    │ (depends on location)
│ DIDComm Decryption    │ 5-10ms      │ (crypto ops)
│ Tool Processing       │ 10-50ms     │ (business logic)
│ Backend DIDComm       │ 50-100ms    │ (if needed)
│ Response Path         │ Same        │ (symmetric)
└─────────────────────────────────────┘

Compare to:
• Direct HTTPS API: 50-150ms
• VPN + API: 200-500ms
• Traditional MCP/HTTP: 10-50ms (local only)
```

---

## ✅ Checklist: Partner Integration Guide

### For Enterprise Partners

- [ ] **Define Your Tools** - What operations will AI agents perform?
- [ ] **Create DIDs** - Generate DID for tool server and backend
- [ ] **Implement MCP ServerHandler** - Use standard MCP interface
- [ ] **Add DIDComm Transport** - Replace HTTP with DIDComm
- [ ] **Deploy Tool Server** - Run with DIDComm listener
- [ ] **Document DIDs** - Share tool server DID with agent developers
- [ ] **Test Flow** - Verify end-to-end encryption works

### For Agent Developers

- [ ] **Create Agent DID** - Generate and store in OS keychain
- [ ] **Get Tool Server DID** - Obtain from enterprise partner
- [ ] **Configure MCP Client** - Use DIDComm transport
- [ ] **Test Tools** - Verify tool discovery and execution
- [ ] **Handle Errors** - Implement proper error handling
- [ ] **Log Interactions** - Audit trail for compliance

---

## 🌟 Real-World Use Cases

### Use Case 1: Password Management
- **Agent**: Claude Desktop
- **Tool**: `get_password(key)`
- **Backend**: Secure password vault
- **Benefit**: No passwords in config files

### Use Case 2: Customer Data Access
- **Agent**: Custom support bot
- **Tools**: `get_customer(id)`, `update_ticket(id, data)`
- **Backend**: CRM system
- **Benefit**: Zero-trust access, full audit trail

### Use Case 3: Multi-Cloud Operations
- **Agent**: DevOps assistant
- **Tools**: `deploy(service)`, `get_logs(service)`
- **Backend**: Kubernetes clusters across clouds
- **Benefit**: Unified interface, decentralized auth

### Use Case 4: Healthcare Data
- **Agent**: Clinical AI assistant
- **Tools**: `get_patient_data(id)`, `search_protocols(condition)`
- **Backend**: HIPAA-compliant database
- **Benefit**: Encryption + verifiable access for compliance

---

## 🔗 Standards & Specifications

### Based On

1. **MCP (Model Context Protocol)**
   - Spec: https://spec.modelcontextprotocol.io
   - JSON-RPC 2.0 based
   - Tool-based architecture

2. **DIDComm Messaging**
   - Spec: https://identity.foundation/didcomm-messaging/spec/
   - W3C DID standard
   - End-to-end encrypted messaging

3. **DID (Decentralized Identifiers)**
   - Spec: https://www.w3.org/TR/did-core/
   - W3C Recommendation
   - Decentralized identity

### New Contribution

**MCP over DIDComm Transport**
- Message Type: `https://mcp.didcomm.org/jsonrpc/1.0`
- Wraps MCP JSON-RPC in DIDComm envelopes
- Maintains full MCP protocol compatibility
- Adds DIDComm security properties

---

## 💼 Business Benefits

### For Enterprises
- ✅ **Security**: End-to-end encryption, zero-trust
- ✅ **Compliance**: Full audit trail, verifiable access
- ✅ **Cost**: No VPN, no certificate management
- ✅ **Flexibility**: Works across clouds, firewalls
- ✅ **Scalability**: Decentralized architecture

### For Agent Developers
- ✅ **Standards**: Build once, work everywhere
- ✅ **Security**: No credential management burden
- ✅ **Discovery**: Find tools via DID resolution
- ✅ **Trust**: Verify enterprise identity
- ✅ **Privacy**: User controls their DID

### For End Users
- ✅ **Security**: Credentials never exposed
- ✅ **Privacy**: Decentralized identity
- ✅ **Control**: Own your agent's identity
- ✅ **Transparency**: Auditable interactions
- ✅ **Reliability**: No single point of failure

---

## 📚 Next Steps

### To Learn More
- [SECURITY_HARDENING.md](SECURITY_HARDENING.md) - Full technical specification
- [ARCHITECTURE_FLOWS.md](ARCHITECTURE_FLOWS.md) - Detailed component flows
- [README.md](README.md) - Quick start guide

### To Implement
1. Clone repository: `git clone <repo>`
2. Review security proposal
3. Choose deployment topology
4. Follow integration checklist
5. Deploy and test

### To Contribute
- Open issues for questions
- Submit PRs for improvements
- Share your use case
- Help document patterns

---

## 🎉 Summary

**MCP over DIDComm = Secure + Standard + Decentralized AI Agent Communication**

This pattern provides:
- ✅ **Security** through DIDComm encryption
- ✅ **Standards** via MCP protocol
- ✅ **Decentralization** with DIDs
- ✅ **Simplicity** for both enterprises and agents
- ✅ **Scalability** for production deployments

**Ready to build secure AI agent integrations for your enterprise!**

---

*Built with ❤️ using [Affinidi TDK](https://github.com/affinidi/affinidi-tdk) and [rmcp](https://github.com/modelcontextprotocol/rust-sdk)*
