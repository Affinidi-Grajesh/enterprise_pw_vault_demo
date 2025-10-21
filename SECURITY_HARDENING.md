# Security Hardening Proposal

This document outlines security improvements for the MCP architecture, focusing on eliminating network vulnerabilities and proper key management.

## 🎯 Executive Summary

**Current Issue:** MCP server communicates with bridge via plaintext HTTP, exposing passwords on the network.

**Recommended Solution:** **MCP over DIDComm Transport**
- Native DIDComm encryption (end-to-end)
- DID-based authentication (no certificates needed)
- Standard MCP protocol compatibility
- Agent owns its DID and keys
- Works through firewalls via mediator
- Decentralized and verifiable

**Alternative Solutions:**
1. **In-Process MCP Server** - Maximum security, zero network exposure
2. **HTTPS + mTLS** - Traditional approach, certificate-based

**Key Innovation:** Extend MCP to support DIDComm as a native transport layer (like stdio, SSE, HTTP), enabling secure, decentralized AI agent communication.

---

## 🔒 Current Security Model (As-Is)

### Architecture
```
┌──────────────┐   stdio      ┌──────────────┐   HTTP       ┌──────────────┐   DIDComm    ┌──────────────┐
│  AI Agent    │  JSON-RPC    │  MCP Server  │  plaintext   │   DIDComm    │  encrypted   │  Password    │
│  (Claude)    │ ──────────── │ (separate    │ ──────────── │   Bridge     │ ──────────── │  Service     │
│              │              │  process)    │   network    │ (persistent) │              │              │
└──────────────┘              └──────────────┘              └──────────────┘              └──────────────┘
```

### Current Vulnerabilities

#### ❌ **Critical: Unencrypted HTTP**
```rust
// Current implementation in mcp-server
let response = self.client
    .post(format!("{}/bridge", self.bridge_url))  // HTTP plaintext!
    .json(&request)                                // Passwords in clear text
    .send()
    .await?;
```

**Risk:**
- Passwords transmitted in plaintext over HTTP
- Vulnerable to man-in-the-middle attacks
- Local network sniffing can capture credentials
- No authentication between MCP server and bridge

#### ❌ **Moderate: Process Separation**
- MCP server runs as separate process
- Inter-process communication via network stack
- Potential for process injection/debugging
- No shared memory protection

#### ❌ **Moderate: Key Management Split**
- Bridge owns persistent DID and keys
- Agent has no control over its identity
- Trust delegation to bridge process
- Cannot audit bridge behavior

---

## 🛡️ Proposed Security Model (To-Be)

### Option 1: In-Process MCP Server (Recommended)

#### Architecture
```
┌─────────────────────────────────────────┐   DIDComm    ┌──────────────┐
│  AI Agent Process                       │  encrypted   │  Password    │
│  ┌──────────────┐   ┌──────────────┐   │ ──────────── │  Service     │
│  │  AI Agent    │──>│  MCP Server  │   │              │              │
│  │  (Claude)    │   │  (in-process │───┼──────────────>              │
│  │              │   │   library)   │   │              │              │
│  └──────────────┘   └──────────────┘   │              │              │
│         │                  │            │              │              │
│         │                  ↓            │              │              │
│         │         ┌──────────────┐      │              │              │
│         └────────>│  Agent DID   │      │              │              │
│                   │  Key Manager │      │              │              │
│                   └──────────────┘      │              │              │
└─────────────────────────────────────────┘              └──────────────┘

         Single Process Boundary
         No Network Calls
         OS-Level Memory Protection
```

#### Benefits
- ✅ **No network exposure** - MCP server runs in agent's process
- ✅ **OS-level protection** - Memory isolation via process boundaries
- ✅ **No HTTP** - Direct function calls
- ✅ **Agent owns DID** - Full control over identity and keys
- ✅ **Reduced attack surface** - Eliminate bridge as separate service
- ✅ **Simpler deployment** - One process, one configuration
- ✅ **Better performance** - No network latency

#### Implementation Changes

**1. Convert MCP Server to Library**
```rust
// src/lib.rs - New library interface
pub mod mcp_server;

pub use mcp_server::PasswordVaultMCPServer;

/// Configuration for the MCP server
pub struct MCPServerConfig {
    /// Agent's DID (created and managed by agent)
    pub agent_did: String,

    /// Agent's DID secrets (stored by agent)
    pub agent_secrets: Vec<Secret>,

    /// Password service DID
    pub service_did: String,

    /// Mediator DID
    pub mediator_did: String,
}

impl PasswordVaultMCPServer {
    /// Create new MCP server instance (in-process)
    pub fn new(config: MCPServerConfig) -> Result<Self> {
        // Initialize DIDComm directly
        // No HTTP client needed
        Ok(Self {
            agent_did: config.agent_did,
            agent_secrets: config.agent_secrets,
            service_did: config.service_did,
            atm: Arc::new(RwLock::new(None)),
        })
    }

    /// Direct DIDComm call (no HTTP)
    async fn get_password_direct(&self, key: String) -> Result<Option<String>> {
        // Direct DIDComm message to service
        // No network call to bridge
        // End-to-end encryption
    }
}
```

**2. Agent Integration Example (Claude Desktop)**
```rust
// Claude Desktop would embed MCP server as library
use enterprise_pw_vault::{PasswordVaultMCPServer, MCPServerConfig};

// Agent manages its own DID
let (agent_did, agent_secrets) = generate_did(mediator_did)?;

// Save to agent's secure storage
save_to_keychain("agent_did", &agent_did)?;
save_to_keychain("agent_secrets", &agent_secrets)?;

// Create in-process MCP server
let mcp_server = PasswordVaultMCPServer::new(MCPServerConfig {
    agent_did,
    agent_secrets,
    service_did: "did:peer:2.Ez6LS...".to_string(),
    mediator_did: DEFAULT_MEDIATOR.to_string(),
})?;

// MCP server runs in same process as Claude
// No network calls for password operations
```

**3. Security Properties**
```
Agent Process Memory Layout:
┌─────────────────────────────────────┐
│  Agent Code                         │
│                                     │
│  ┌───────────────────────────────┐ │
│  │  MCP Server (in-process)      │ │
│  │  - DIDComm client             │ │
│  │  - No HTTP                    │ │
│  │  - Direct function calls      │ │
│  └───────────────────────────────┘ │
│                                     │
│  ┌───────────────────────────────┐ │
│  │  Agent DID Key Manager        │ │
│  │  - Private keys               │ │
│  │  - Encrypted at rest          │ │
│  │  - OS keychain integration    │ │
│  └───────────────────────────────┘ │
│                                     │
└─────────────────────────────────────┘
        ↕ OS Process Boundary
   No data leaves process except
   encrypted DIDComm messages
```

---

### Option 2: MCP over DIDComm Transport (Recommended Alternative)

Native DIDComm transport for MCP protocol - eliminates HTTP/HTTPS entirely while keeping standard MCP.

#### Architecture
```
┌──────────────┐   stdio      ┌──────────────┐   DIDComm    ┌──────────────┐   DIDComm    ┌──────────────┐
│  AI Agent    │  JSON-RPC    │  MCP Server  │  MCP-over-   │   MCP Tool   │  Business    │  Password    │
│  (Claude)    │ ──────────── │ (has agent   │ ─DIDComm──── │   Server     │ ─DIDComm──── │  Service     │
│              │              │  DID)        │  encrypted   │ (has DID)    │  encrypted   │              │
└──────────────┘              └──────────────┘              └──────────────┘              └──────────────┘
                                      │                              │
                                      │      MCP Protocol            │
                                      │   (initialize, tools/list,   │
                                      │    tools/call, etc.)         │
                                      │   Wrapped in DIDComm         │
                                      └──────────────────────────────┘
```

#### Concept: MCP as a DIDComm Protocol

Just like MCP supports multiple transports (stdio, SSE, HTTP), we add **DIDComm** as a native transport:

**Standard MCP Transports:**
- `stdio` - JSON-RPC over stdin/stdout
- `sse` - Server-Sent Events over HTTP
- `http` - HTTP with SSE upgrade

**New Transport:**
- `didcomm` - JSON-RPC messages wrapped in DIDComm encrypted messages

#### Benefits
- ✅ **Native encryption** - DIDComm provides end-to-end encryption
- ✅ **Authentication** - DID-based authentication built-in
- ✅ **Decentralized** - No central authority required
- ✅ **Standard MCP** - Same protocol, different transport
- ✅ **No HTTP** - Eliminates HTTP vulnerabilities
- ✅ **Agent owns DID** - Full control over identity
- ✅ **Backward compatible** - Existing MCP tools work
- ✅ **Verifiable** - All messages cryptographically signed

#### Implementation

**1. DIDComm Transport for MCP**

```rust
// src/mcp/transport/didcomm.rs

use rmcp::transport::{Transport, TransportMessage};
use affinidi_tdk::didcomm::*;
use anyhow::Result;

/// DIDComm transport for MCP protocol
pub struct DIDCommTransport {
    /// Our DID (agent)
    our_did: String,

    /// Our secrets
    our_secrets: Vec<Secret>,

    /// Remote DID (tool server)
    remote_did: String,

    /// ATM instance
    atm: Arc<ATM>,

    /// Profile
    profile: Arc<ATMProfile>,
}

impl DIDCommTransport {
    pub async fn new(
        agent_did: String,
        agent_secrets: Vec<Secret>,
        tool_server_did: String,
        mediator: String,
    ) -> Result<Self> {
        // Initialize ATM (Affinidi Trusted Messaging)
        let secrets_resolver = SecretsResolver::new(agent_secrets.clone());
        let config = ATMConfig {
            mediator_did: mediator.clone(),
            atm_api: None,
            atm_did: None,
            secrets_resolver,
        };

        let atm = Arc::new(ATM::new(config).await?);

        // Register with mediator
        let profile = Arc::new(atm.create_profile(&agent_did, &mediator).await?);

        Ok(Self {
            our_did: agent_did,
            our_secrets: agent_secrets,
            remote_did: tool_server_did,
            atm,
            profile,
        })
    }
}

#[async_trait]
impl Transport for DIDCommTransport {
    /// Send MCP message wrapped in DIDComm
    async fn send(&mut self, message: TransportMessage) -> Result<()> {
        // Serialize MCP message (JSON-RPC)
        let mcp_json = serde_json::to_value(&message)?;

        // Create DIDComm message with MCP payload
        let didcomm_msg = MessageBuilder::new(
            Uuid::new_v4().to_string(),
            "https://mcp.didcomm.org/jsonrpc/1.0".to_string(), // MCP message type
            mcp_json,
        )
        .from(self.our_did.clone())
        .to(self.remote_did.clone())
        .created_time(current_timestamp())
        .expires_time(current_timestamp() + 300)
        .finalize();

        // Pack and encrypt
        let (packed_msg, _) = self.atm
            .pack_encrypted(
                &didcomm_msg,
                &self.remote_did,
                Some(&self.our_did),
                Some(&self.our_did),
                Some(&PackEncryptedOptions {
                    forward: false,
                    ..Default::default()
                }),
            )
            .await?;

        // Send via DIDComm
        self.atm
            .forward_and_send_message(&self.profile, false, &packed_msg)
            .await?;

        Ok(())
    }

    /// Receive MCP message from DIDComm
    async fn receive(&mut self) -> Result<TransportMessage> {
        // Receive DIDComm message
        let response = self.atm
            .receive_message(&self.profile, Some(Duration::from_secs(30)))
            .await?;

        // Unpack DIDComm message
        let unpacked = self.atm.unpack(&response).await?;

        // Extract MCP payload
        let mcp_message: TransportMessage = serde_json::from_value(unpacked.message.body)?;

        Ok(mcp_message)
    }
}
```

**2. MCP Server with DIDComm Transport**

```rust
// src/bin/mcp_server/didcomm_transport.rs

use anyhow::Result;
use rmcp::{ServerHandler, ServiceExt};
use enterprise_pw_vault::mcp::transport::DIDCommTransport;

struct PasswordVaultMCPServer {
    // No HTTP client needed!
    // Direct DIDComm to password service
    our_did: String,
    our_secrets: Vec<Secret>,
    service_did: String,
    atm: Arc<RwLock<Option<Arc<ATM>>>>,
}

impl ServerHandler for PasswordVaultMCPServer {
    // Same implementation as before
    // get_info, list_tools, call_tool
}

#[tokio::main]
async fn main() -> Result<()> {
    let args = Args::parse();

    // Load agent DID from keychain
    let (agent_did, agent_secrets) = load_from_keychain()?;

    // Create MCP server
    let server = PasswordVaultMCPServer::new(
        agent_did.clone(),
        agent_secrets.clone(),
        args.tool_server_did,
    );

    // Use DIDComm transport instead of stdio
    let transport = DIDCommTransport::new(
        agent_did,
        agent_secrets,
        args.tool_server_did,
        args.mediator,
    ).await?;

    // Serve MCP over DIDComm
    let service = server.serve(transport).await?;
    service.waiting().await?;

    Ok(())
}
```

**3. MCP Tool Server (Password Vault)**

```rust
// src/bin/mcp_tool_server/main.rs

/// MCP Tool Server that receives MCP requests via DIDComm
/// and forwards to actual password service

use rmcp::{ServerHandler, ServiceExt};
use enterprise_pw_vault::mcp::transport::DIDCommTransport;

struct PasswordVaultToolServer {
    our_did: String,
    our_secrets: Vec<Secret>,
    password_service_did: String,
    atm: Arc<ATM>,
}

impl ServerHandler for PasswordVaultToolServer {
    async fn call_tool(
        &self,
        request: CallToolRequestParam,
        _context: RequestContext<RoleServer>,
    ) -> Result<CallToolResult, McpError> {
        match request.name.as_ref() {
            "get_password" => {
                let key = request.arguments
                    .as_ref()
                    .and_then(|args| args.get("key"))
                    .and_then(|v| v.as_str())
                    .ok_or_else(|| McpError::invalid_params("Missing 'key'", None))?;

                // Forward to password service via DIDComm
                let password = self.get_password_from_service(key).await?;

                Ok(CallToolResult {
                    content: Some(vec![Content::text(format!(
                        "Password for '{}': {}",
                        key, password
                    ))]),
                    is_error: None,
                    structured_content: None,
                })
            }
            _ => Err(McpError::invalid_request("Unknown tool", None)),
        }
    }
}

#[tokio::main]
async fn main() -> Result<()> {
    let args = Args::parse();

    // Tool server has its own DID
    let (tool_did, tool_secrets) = load_from_config()?;

    let server = PasswordVaultToolServer::new(
        tool_did.clone(),
        tool_secrets.clone(),
        args.password_service_did,
    );

    // Listen for MCP requests over DIDComm
    let transport = DIDCommTransport::new(
        tool_did,
        tool_secrets,
        "any", // Accept from any agent
        args.mediator,
    ).await?;

    println!("MCP Tool Server listening via DIDComm");
    println!("Tool Server DID: {}", args.tool_did);

    let service = server.serve(transport).await?;
    service.waiting().await?;

    Ok(())
}
```

#### Message Flow Example

**1. Agent → Tool Server (MCP Initialize)**
```json
// DIDComm envelope (encrypted)
{
  "type": "https://mcp.didcomm.org/jsonrpc/1.0",
  "from": "did:peer:2.Ez6LS...(agent)",
  "to": "did:peer:2.Ez6LS...(tool-server)",
  "body": {
    // MCP JSON-RPC inside
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "protocolVersion": "2024-11-05",
      "capabilities": {},
      "clientInfo": {
        "name": "claude-agent",
        "version": "1.0.0"
      }
    }
  }
}
```

**2. Tool Server → Agent (MCP Initialize Response)**
```json
// DIDComm envelope (encrypted)
{
  "type": "https://mcp.didcomm.org/jsonrpc/1.0",
  "from": "did:peer:2.Ez6LS...(tool-server)",
  "to": "did:peer:2.Ez6LS...(agent)",
  "body": {
    // MCP JSON-RPC response
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
      "protocolVersion": "2024-11-05",
      "capabilities": {
        "tools": {}
      },
      "serverInfo": {
        "name": "password-vault-mcp",
        "version": "0.1.0"
      }
    }
  }
}
```

**3. Agent → Tool Server (Get Password)**
```json
// DIDComm envelope (encrypted)
{
  "type": "https://mcp.didcomm.org/jsonrpc/1.0",
  "from": "did:peer:2.Ez6LS...(agent)",
  "to": "did:peer:2.Ez6LS...(tool-server)",
  "body": {
    "jsonrpc": "2.0",
    "id": 2,
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

**4. Tool Server → Password Service (DIDComm)**
```json
// Standard DIDComm message
{
  "type": "https://demo.example/password/request",
  "from": "did:peer:2.Ez6LS...(tool-server)",
  "to": "did:peer:2.Ez6LS...(password-service)",
  "body": {
    "key": "myapp"
  }
}
```

#### Configuration

**Agent Config:**
```json
{
  "mcpServers": {
    "password-vault": {
      "transport": "didcomm",
      "toolServerDid": "did:peer:2.Ez6LS...(tool-server)",
      "mediator": "did:web:mediator-nlb.storm.ws:mediator:v1:.well-known"
    }
  }
}
```

**Tool Server Config:**
```json
{
  "our_did": "did:peer:2.Ez6LS...(tool-server)",
  "did_secrets": [...],
  "password_service_did": "did:peer:2.Ez6LS...(password-service)",
  "mediator_did": "did:web:mediator-nlb.storm.ws:mediator:v1:.well-known"
}
```

#### Advantages over HTTP

| Aspect | HTTP/HTTPS | DIDComm Transport |
|--------|------------|-------------------|
| **Encryption** | TLS only | End-to-end DIDComm |
| **Authentication** | mTLS/certificates | DID-based (cryptographic) |
| **Identity** | IP/DNS based | Decentralized DID |
| **Verification** | CA trust chain | Direct cryptographic verification |
| **Network** | Requires open ports | Works through mediator |
| **Firewall** | May be blocked | Mediator handles routing |
| **Privacy** | IP addresses exposed | DIDs don't reveal location |
| **Auditability** | Server logs | Cryptographically signed messages |

---

### Option 3: Secure Bridge with TLS (Fallback)

If keeping bridge as separate service is required:

#### Architecture
```
┌──────────────┐   stdio      ┌──────────────┐   HTTPS+mTLS ┌──────────────┐   DIDComm    ┌──────────────┐
│  AI Agent    │  JSON-RPC    │  MCP Server  │  encrypted   │   DIDComm    │  encrypted   │  Password    │
│  (Claude)    │ ──────────── │ (has agent   │ ──────────── │   Bridge     │ ──────────── │  Service     │
│              │              │  DID)        │   network    │ (no DID)     │              │              │
└──────────────┘              └──────────────┘              └──────────────┘              └──────────────┘
```

#### Changes Required

**1. Add HTTPS + Mutual TLS**
```rust
// Bridge configuration
struct BridgeConfig {
    // TLS certificates
    tls_cert: PathBuf,
    tls_key: PathBuf,
    client_ca: PathBuf,  // For mTLS validation

    // No DID! Bridge is just a relay
    service_did: String,
}

// Bridge server with TLS
let tls_config = RustlsConfig::from_pem_file(
    config.tls_cert,
    config.tls_key
).await?;

axum_server::bind_rustls(addr, tls_config)
    .serve(app.into_make_service())
    .await?;
```

**2. Move DID to MCP Server**
```rust
// MCP Server now owns agent DID
struct MCPServerConfig {
    agent_did: String,           // Agent's persistent DID
    agent_secrets: Vec<Secret>,  // Agent controls keys
    bridge_url: String,          // Now HTTPS with mTLS
    service_did: String,
    client_cert: PathBuf,        // For mTLS auth
    client_key: PathBuf,
}

// MCP server makes DIDComm calls
async fn get_password(&self, key: String) -> Result<Option<String>> {
    // Create DIDComm message
    let message = MessageBuilder::new(...)
        .from(self.agent_did.clone())  // Agent is sender
        .to(self.service_did.clone())
        .finalize();

    // Send via HTTPS to bridge (encrypted twice: TLS + DIDComm)
    let response = self.client
        .post(format!("{}/relay", self.bridge_url))
        .json(&message)
        .send()
        .await?;
}
```

**3. Bridge as Dumb Relay**
```rust
// Bridge just forwards DIDComm messages (doesn't decrypt)
async fn relay_didcomm(
    State(state): State<Arc<BridgeState>>,
    Json(encrypted_message): Json<String>,
) -> Result<Json<String>, AppError> {
    // Bridge doesn't decrypt - just relays
    let response = state.atm
        .forward_and_send_message(
            &state.profile,
            false,
            &encrypted_message,
        )
        .await?;

    Ok(Json(response))
}
```

#### Benefits
- ✅ HTTPS encrypts transport layer
- ✅ mTLS authenticates both sides
- ✅ Agent owns DID and keys
- ✅ Bridge can't see plaintext passwords
- ✅ Can audit/monitor bridge separately

#### Drawbacks
- ⚠️ Still has network exposure
- ⚠️ More complex certificate management
- ⚠️ Performance overhead (TLS handshake)
- ⚠️ Bridge is still a separate attack surface

---

## 🔑 Key Management Architecture

### Proposed: Agent-Managed DIDs

```
┌─────────────────────────────────────────────────────┐
│  AI Agent (Claude Desktop, etc.)                    │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │  Agent Key Manager                             │ │
│  │  ┌──────────────────────────────────────────┐ │ │
│  │  │  DID Generation (First Run)              │ │ │
│  │  │  - generate_did(mediator)                │ │ │
│  │  │  - Returns (did, secrets)                │ │ │
│  │  │  - Save to OS keychain/secure storage    │ │ │
│  │  └──────────────────────────────────────────┘ │ │
│  │                                                │ │
│  │  ┌──────────────────────────────────────────┐ │ │
│  │  │  Secure Storage Integration              │ │ │
│  │  │  - macOS: Keychain                       │ │ │
│  │  │  - Windows: DPAPI/Credential Manager     │ │ │
│  │  │  - Linux: Secret Service API             │ │ │
│  │  └──────────────────────────────────────────┘ │ │
│  │                                                │ │
│  │  ┌──────────────────────────────────────────┐ │ │
│  │  │  Key Rotation                            │ │ │
│  │  │  - Periodic key updates                  │ │ │
│  │  │  - Revocation support                    │ │ │
│  │  └──────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │  MCP Server (In-Process)                       │ │
│  │  - Reads DID from key manager                  │ │
│  │  - No key storage                              │ │
│  │  - Just uses keys for DIDComm                  │ │
│  └────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

### Implementation

**1. Agent Initialization**
```rust
// On first run, agent creates its DID
pub struct AgentIdentity {
    pub did: String,
    pub secrets: Vec<Secret>,
    pub created_at: SystemTime,
}

impl AgentIdentity {
    /// Initialize agent identity (first run)
    pub fn initialize(mediator: &str) -> Result<Self> {
        let (did, secrets) = generate_did(mediator)?;

        let identity = Self {
            did,
            secrets,
            created_at: SystemTime::now(),
        };

        // Save to OS-specific secure storage
        identity.save_to_keychain()?;

        Ok(identity)
    }

    /// Load existing identity
    pub fn load_from_keychain() -> Result<Self> {
        // Platform-specific keychain access
        #[cfg(target_os = "macos")]
        {
            Self::load_from_macos_keychain()
        }

        #[cfg(target_os = "windows")]
        {
            Self::load_from_windows_credential_manager()
        }

        #[cfg(target_os = "linux")]
        {
            Self::load_from_secret_service()
        }
    }

    #[cfg(target_os = "macos")]
    fn save_to_keychain(&self) -> Result<()> {
        use security_framework::passwords::*;

        // Save DID
        set_generic_password(
            "ai.agent.identity",
            "did",
            self.did.as_bytes(),
        )?;

        // Save secrets (encrypted)
        let secrets_json = serde_json::to_string(&self.secrets)?;
        set_generic_password(
            "ai.agent.identity",
            "secrets",
            secrets_json.as_bytes(),
        )?;

        Ok(())
    }
}
```

**2. Claude Desktop Integration**
```javascript
// Claude Desktop config (conceptual)
{
  "identity": {
    "useSystemKeychain": true,
    "serviceName": "ai.agent.identity"
  },
  "mcpServers": {
    "password-vault": {
      // MCP server runs in-process
      "type": "embedded",
      "module": "enterprise_pw_vault",
      "config": {
        "serviceDid": "did:peer:2.Ez6LS...",
        "mediator": "did:web:mediator-nlb.storm.ws:mediator:v1:.well-known"
      }
    }
  }
}
```

---

## 🏗️ Migration Path

### Phase 1: Immediate (Critical Security Fix)
**Goal:** Stop plaintext password transmission

**Option A: Implement MCP/DIDComm Transport** (Recommended)

**Changes:**
1. Create DIDComm transport implementation for rmcp
2. Update MCP server to use DIDComm transport
3. Create MCP Tool Server (receives MCP via DIDComm)
4. Agent manages its own DID

**Files to create:**
- `src/mcp/transport/didcomm.rs` - DIDComm transport for MCP
- `src/bin/mcp_tool_server/main.rs` - MCP tool server
- `src/agent/key_manager.rs` - Agent DID management

**Files to modify:**
- `src/bin/mcp_server/http_client.rs` → `src/bin/mcp_server/didcomm_client.rs`
- `Cargo.toml` - Add rmcp transport feature

**Benefits:**
- ✅ Native encryption (no TLS needed)
- ✅ DID-based authentication
- ✅ Standard MCP protocol
- ✅ Decentralized architecture

**Timeline:** 3-5 days

---

**Option B: Add HTTPS + mTLS** (Fallback)

**Changes:**
1. Add HTTPS + mTLS to bridge
2. Update MCP server to use HTTPS
3. Add certificate management

**Files to modify:**
- `src/bin/mcp_server/bridge_service.rs` - Add TLS
- `src/bin/mcp_server/http_client.rs` - Use HTTPS client
- Add `Cargo.toml` dependencies: `rustls`, `tokio-rustls`

**Timeline:** 1-2 days

---

### Phase 2: Near-term (If using Option B)
**Goal:** Move DID ownership to agent

**Changes:**
1. Move DID generation to MCP server
2. Add key storage configuration
3. Bridge becomes stateless relay
4. Update configuration files

**Files to modify:**
- `src/bin/mcp_server/http_client.rs` - Add DID management
- `src/bin/mcp_server/bridge_service.rs` - Remove DID, become relay
- Update `README.md` and `ARCHITECTURE_FLOWS.md`

**Timeline:** 3-5 days

**Note:** Not needed if using MCP/DIDComm transport (Option A) - agent already owns DID

---

### Phase 3: Long-term (Optional Enhancement)
**Goal:** In-process MCP server for maximum security

**Changes:**
1. Convert MCP server to library crate
2. Remove separate process entirely
3. OS keychain integration
4. Agent SDK for embedding

**New files:**
- `src/mcp_server/lib.rs` - Library interface
- `src/agent/keychain.rs` - OS keychain integration
- `examples/embedded_agent.rs` - Example integration

**Timeline:** 1-2 weeks

**Note:** This can work with either DIDComm transport or direct function calls

---

## 📊 Security Comparison

| Aspect | Current | With TLS | MCP/DIDComm | In-Process |
|--------|---------|----------|-------------|------------|
| **Transport Security** | ❌ Plaintext HTTP | ✅ HTTPS + mTLS | ✅ E2E DIDComm | ✅ N/A (no network) |
| **Password Exposure** | ❌ Network plaintext | ✅ TLS encrypted | ✅ DIDComm encrypted | ✅ In-memory only |
| **DID Ownership** | ❌ Bridge owns | ✅ Agent owns | ✅ Agent owns | ✅ Agent owns |
| **Attack Surface** | ❌ Large (HTTP + Bridge) | ⚠️ Medium (HTTPS + Bridge) | ✅ Small (DIDComm only) | ✅ Smallest (process only) |
| **Network Exposure** | ❌ Yes | ⚠️ Yes (encrypted) | ⚠️ Yes (E2E encrypted) | ✅ No |
| **Process Boundary** | ❌ Crosses | ⚠️ Crosses | ⚠️ Crosses | ✅ Same process |
| **Key Management** | ❌ Bridge controls | ✅ Agent controls | ✅ Agent controls | ✅ Agent controls |
| **Authentication** | ❌ None | ✅ mTLS certs | ✅ DID-based crypto | ✅ N/A |
| **Decentralized** | ❌ No | ❌ No | ✅ Yes | ✅ Yes (if desired) |
| **Standard MCP** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |
| **Performance** | ⚠️ Network latency | ⚠️ TLS overhead | ⚠️ DIDComm overhead | ✅ Function call |
| **Deployment** | ⚠️ Two services | ⚠️ Two services + certs | ⚠️ Two services | ✅ One process |
| **Firewall Friendly** | ⚠️ Requires ports | ⚠️ Requires ports | ✅ Via mediator | ✅ N/A |
| **Verifiable** | ❌ No | ⚠️ Cert-based | ✅ Crypto signatures | ✅ N/A |

---

## 🎯 Recommendations

### Recommended Approach Comparison

**Best for Maximum Security:** In-Process MCP Server (Option 1)
- Zero network exposure
- OS-level memory protection
- Simplest deployment

**Best for Standard MCP Compatibility:** MCP over DIDComm Transport (Option 2)
- Standard MCP protocol
- Native DIDComm encryption and authentication
- Decentralized identity
- Works through firewalls via mediator
- **Recommended for production enterprise deployments**

**Best for Legacy Systems:** HTTPS + mTLS (Option 3)
- Traditional security model
- Certificate-based authentication
- Familiar to ops teams

---

### Immediate Action (This Week)
**Priority: CRITICAL**

**Option A: Implement MCP/DIDComm Transport** (Recommended)
- Add DIDComm transport layer to rmcp
- Update MCP server to use DIDComm instead of HTTP
- Create MCP Tool Server that listens via DIDComm
- Agent manages its own DID

**Option B: Add TLS** (Fallback)
- Add TLS to bridge to stop plaintext password transmission
- Implement mTLS for authentication

```bash
# Generate certificates
openssl req -x509 -newkey rsa:4096 -keyout bridge-key.pem -out bridge-cert.pem -days 365 -nodes
```

---

### Short-term (Next Sprint)
**Priority: HIGH**

Move DID ownership to MCP server:
1. Agent creates and stores DID on first run
2. MCP server reads DID from configuration
3. Bridge becomes stateless relay
4. Agent controls its own identity

---

### Long-term (Strategic)
**Priority: RECOMMENDED**

Convert to in-process MCP server:
1. Create library crate for MCP server
2. Integrate with OS keychain for DID storage
3. Eliminate network calls entirely
4. Provide SDK for agent developers

**Benefits:**
- Maximum security (no network exposure)
- Best performance (no HTTP overhead)
- Simplified deployment (one process)
- Agent has full control

---

## 📚 Implementation Examples

### Example 1: macOS Keychain Integration

```rust
// src/agent/keychain_macos.rs
use security_framework::passwords::*;
use anyhow::Result;

pub struct MacOSKeychain;

impl MacOSKeychain {
    pub fn save_did(did: &str, secrets: &[Secret]) -> Result<()> {
        set_generic_password(
            "ai.agent.mcp",
            "did",
            did.as_bytes(),
        )?;

        let secrets_json = serde_json::to_string(secrets)?;
        set_generic_password(
            "ai.agent.mcp",
            "secrets",
            secrets_json.as_bytes(),
        )?;

        Ok(())
    }

    pub fn load_did() -> Result<(String, Vec<Secret>)> {
        let did_bytes = get_generic_password("ai.agent.mcp", "did")?;
        let did = String::from_utf8(did_bytes)?;

        let secrets_bytes = get_generic_password("ai.agent.mcp", "secrets")?;
        let secrets_json = String::from_utf8(secrets_bytes)?;
        let secrets: Vec<Secret> = serde_json::from_str(&secrets_json)?;

        Ok((did, secrets))
    }
}
```

### Example 2: In-Process MCP Server

```rust
// examples/embedded_agent.rs
use enterprise_pw_vault::{PasswordVaultMCPServer, MCPServerConfig};
use anyhow::Result;

#[tokio::main]
async fn main() -> Result<()> {
    // Agent manages its own DID
    let (agent_did, agent_secrets) = if let Ok((did, secrets)) = load_from_keychain() {
        (did, secrets)
    } else {
        // First run - create DID
        let (did, secrets) = generate_did(DEFAULT_MEDIATOR)?;
        save_to_keychain(&did, &secrets)?;
        (did, secrets)
    };

    // Create in-process MCP server (no network!)
    let mcp_server = PasswordVaultMCPServer::new(MCPServerConfig {
        agent_did,
        agent_secrets,
        service_did: "did:peer:2.Ez6LS...".to_string(),
        mediator_did: DEFAULT_MEDIATOR.to_string(),
    })?;

    // Use MCP server directly (function calls, not HTTP)
    let password = mcp_server.get_password("myapp").await?;
    println!("Retrieved password: {:?}", password);

    Ok(())
}
```

---

## � MCP over DIDComm Protocol Specification

### Message Type Registry

To standardize MCP over DIDComm, we define a new message type namespace:

```
https://mcp.didcomm.org/jsonrpc/1.0
```

### Message Structure

All MCP messages are wrapped in DIDComm envelopes:

**Generic MCP/DIDComm Message:**
```json
{
  "type": "https://mcp.didcomm.org/jsonrpc/1.0",
  "id": "<unique-message-id>",
  "from": "did:peer:2.Ez6LS...(sender)",
  "to": "did:peer:2.Ez6LS...(receiver)",
  "created_time": 1698765432,
  "expires_time": 1698765732,
  "body": {
    // Standard MCP JSON-RPC message
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": { ... }
  }
}
```

### Request/Response Pattern

**Pattern 1: Request → Response**
```
Agent → Tool Server: MCP Request (via DIDComm)
Tool Server → Agent: MCP Response (via DIDComm)
```

**Pattern 2: Notifications**
```
Agent → Tool Server: MCP Notification (no response expected)
```

### Implementation Requirements

**For rmcp crate:**
1. Add `Transport` trait implementation for DIDComm
2. Support async send/receive
3. Handle message correlation (match responses to requests)
4. Support timeouts

**For tool servers:**
1. Listen for DIDComm messages of type `https://mcp.didcomm.org/jsonrpc/1.0`
2. Extract MCP JSON-RPC from body
3. Process using standard `ServerHandler`
4. Wrap response in DIDComm envelope
5. Send back to requesting DID

### Security Properties

1. **Confidentiality**: All MCP messages encrypted via DIDComm
2. **Integrity**: Messages cryptographically signed
3. **Authentication**: Sender verified via DID
4. **Non-repudiation**: All messages are signed (audit trail)
5. **Forward secrecy**: DIDComm supports ephemeral keys

### Example: Complete Flow

```
1. Agent initializes DID:
   did:peer:2.Ez6LS...(agent)

2. Agent discovers tool server DID:
   did:peer:2.Ez6LS...(tool-server)

3. Agent sends initialize via DIDComm:
   {
     "type": "https://mcp.didcomm.org/jsonrpc/1.0",
     "from": "did:peer:2.Ez6LS...(agent)",
     "to": "did:peer:2.Ez6LS...(tool-server)",
     "body": {
       "jsonrpc": "2.0",
       "id": 1,
       "method": "initialize",
       "params": {...}
     }
   }

4. Tool server responds via DIDComm:
   {
     "type": "https://mcp.didcomm.org/jsonrpc/1.0",
     "from": "did:peer:2.Ez6LS...(tool-server)",
     "to": "did:peer:2.Ez6LS...(agent)",
     "body": {
       "jsonrpc": "2.0",
       "id": 1,
       "result": {...}
     }
   }

5. Agent sends initialized notification:
   {
     "type": "https://mcp.didcomm.org/jsonrpc/1.0",
     "from": "did:peer:2.Ez6LS...(agent)",
     "to": "did:peer:2.Ez6LS...(tool-server)",
     "body": {
       "jsonrpc": "2.0",
       "method": "notifications/initialized"
     }
   }

6. Agent can now call tools via DIDComm...
```

---

## �🔍 Threat Model Analysis

### Current Threats

| Threat | Likelihood | Impact | Current | With TLS | In-Process |
|--------|------------|--------|---------|----------|------------|
| Local network sniffing | High | Critical | ❌ Exposed | ✅ Encrypted | ✅ N/A |
| MITM attack | Medium | Critical | ❌ Possible | ✅ Prevented | ✅ N/A |
| Process debugging | Low | High | ⚠️ Possible | ⚠️ Possible | ✅ OS-protected |
| Key extraction | Medium | Critical | ❌ Bridge has keys | ✅ Agent has keys | ✅ Agent has keys |
| Service impersonation | Medium | High | ❌ No auth | ✅ mTLS prevents | ✅ N/A |
| Memory dumps | Low | Medium | ⚠️ Two processes | ⚠️ Two processes | ✅ One process |

---

## 📖 Related Documentation

- [ARCHITECTURE_FLOWS.md](ARCHITECTURE_FLOWS.md) - Current architecture
- [BIFURCATED_ARCHITECTURE.md](BIFURCATED_ARCHITECTURE.md) - Design rationale
- [README.md](README.md) - Quick start guide

---

## ✅ Next Steps

1. **Review this proposal** with security team
2. **Choose migration path** (recommend in-process for maximum security)
3. **Create implementation tickets**:
   - [ ] Phase 1: Add TLS to bridge (immediate)
   - [ ] Phase 2: Move DID to agent (short-term)
   - [ ] Phase 3: In-process MCP server (strategic)
4. **Update architecture docs** after each phase
5. **Security audit** after Phase 1 completion

---

## 💡 Key Insights

1. **Current vulnerability**: HTTP transmits passwords in plaintext on localhost
2. **Agent should own DID**: Better security model, proper key custody
3. **In-process is optimal**: Eliminates network entirely, OS-level protection
4. **Migration is feasible**: Can be done incrementally with backward compatibility
5. **DIDComm remains secure**: End-to-end encryption maintained throughout

**The in-process model gives us the best of both worlds:**
- MCP protocol for AI agent integration
- DIDComm security for enterprise backend
- No network exposure between them
- Agent controls its own identity
