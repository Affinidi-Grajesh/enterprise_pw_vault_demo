# Enterprise Password Vault Demo

A password vault with **MCP architecture** - separating MCP protocol from DIDComm security layer for scalability, security, and AI agent integration.

## 🎯 What Can I Do With This?

| I Want To... | Use This | Command |
|--------------|----------|---------|
| **Retrieve passwords** | Client | `cargo run --bin client -- -s <DID> --password-key myapp` |
| **Integrate with AI agents** | MCP Server + Bridge | See [Quick Start](#quick-start) |
| **Test the architecture** | Test Client | `cargo run --bin bridge-test-client -- list-tools` |

## �️ Architecture

**Design**: Clean separation between MCP protocol and DIDComm security

```
┌─────────────┐   stdio    ┌─────────────┐   HTTP     ┌──────────────┐   DIDComm  ┌──────────────┐
│  AI Agent   │ ─────────> │ MCP Server  │ ─────────> │   DIDComm    │ ─────────> │  Password    │
│  (Claude)   │  JSON-RPC  │ (stateless) │   REST     │   Bridge     │  Encrypted │  Service     │
│             │ <───────── │             │ <─────────  │ (persistent) │ <─────────  │              │
└─────────────┘            └─────────────┘            └──────────────┘            └──────────────┘
```

**Benefits**:
- ✅ **Persistent Bridge DID** - Trusted, verifiable identity
- ✅ **Lightweight MCP Servers** - No DIDComm overhead
- ✅ **Scalable** - Multiple agents → one bridge
- ✅ **Secure** - Centralized credential management

📖 **Full Details**: [architecture-flows.md](docs/architecture/architecture-flows.md)

## Components

### Core Services

1. **🔐 Password Service** (`service`)
   - Backend vault storing passwords
   - DIDComm-only access
   - [Source: `src/bin/service/`](src/bin/service/)

2. **💻 CLI Client** (`client`)
   - Direct password management
   - DIDComm client
   - [Source: `src/bin/client/`](src/bin/client/)

### MCP Architecture

3. **🌉 DIDComm Bridge** (`didcomm-bridge`)
   - Persistent service with fixed DID
   - HTTP API for MCP servers
   - DIDComm backend to password service
   - [Source: `src/bin/mcp_server/bridge_service.rs`](src/bin/mcp_server/bridge_service.rs)

4. **🤖 MCP Server** (`mcp-server`)
   - Standard MCP protocol (stdio/JSON-RPC)
   - HTTP client to bridge
   - Spawned by AI agents
   - [Source: `src/bin/mcp_server/http_client.rs`](src/bin/mcp_server/http_client.rs)

5. **🧪 Test Client** (`bridge-test-client`)
   - Validates complete flow
   - Tests MCP → Bridge → Service
   - [Source: `src/bin/mcp_server/test_client.rs`](src/bin/mcp_server/test_client.rs)

## Quick Start

### 3-Step Setup

#### Step 1: Start Password Service
```bash
# Terminal 1
cargo run --bin service
```
**→ Copy the Service DID** (e.g., `did:peer:2.Ez6LSghw...`)

**Note:** On first run, the setup wizard will help you configure passwords in `config.json`. These passwords are pre-configured and cannot be changed via the API.

---

#### Step 2: Start DIDComm Bridge
```bash
# Terminal 2
cargo run --bin didcomm-bridge -- --service-did "did:peer:2.Ez6LSghw..."
```
**→ Bridge DID is saved** and reused on restart

---

#### Step 3: Test It!
```bash
# Terminal 3 - Test via MCP (passwords are pre-configured in config.json)
cargo run --bin bridge-test-client -- \
  --bridge-url http://127.0.0.1:8080 \
  get-password --key myapp
```

**Expected Output**: `Password for 'myapp': secret123`

---

### Integration with AI Agents

#### Claude Desktop
Edit `claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "password-vault": {
      "command": "cargo",
      "args": ["run", "--bin", "mcp-server", "--", "--bridge-url", "http://127.0.0.1:8080"],
      "cwd": "/path/to/enterprise_pw_vault_demo"
    }
  }
}
```

#### MCP Inspector
```bash
npx @modelcontextprotocol/inspector \
  cargo run --bin mcp-server -- --bridge-url http://127.0.0.1:8080
```

---

## Available Commands

### Core Operations
```bash
# Start services
cargo run --bin service                              # Password vault
cargo run --bin didcomm-bridge -- --service-did <DID> # Bridge

# Password retrieval (passwords must be in config.json)
cargo run --bin client -- -s <DID> --password-key <KEY>

# Testing
cargo run --bin bridge-test-client -- list-tools
cargo run --bin bridge-test-client -- get-password --key <KEY>
```

### Bridge Management
```bash
# Custom port
cargo run --bin didcomm-bridge -- --service-did <DID> --port 9090

# Check health
curl http://127.0.0.1:8080/health

# Test bridge directly
curl -X POST http://127.0.0.1:8080/bridge \
  -H "Content-Type: application/json" \
  -d '{"type":"GetPassword","key":"myapp"}'
```

---

## Key Features

- ✅ **Architecture**: Clean MCP/DIDComm separation
- ✅ **Persistent Bridge DID**: Trusted, verifiable identity
- ✅ **End-to-End Encryption**: All passwords via DIDComm
- ✅ **Decentralized**: No central authority
- ✅ **AI-Agent Ready**: Claude, MCP Inspector, custom agents
- ✅ **Scalable**: Multiple agents → one bridge
- ✅ **Production Ready**: Centralized security management

---

## Documentation

### Architecture
- 📖 [Architecture Flows](docs/architecture/architecture-flows.md) - Component interactions and data flows
- 📖 [MCP/DIDComm Architecture](docs/architecture/mcp-didcomm-architecture.md) - Visual architecture guide

### Security
- 🔒 [Security Hardening](docs/security/security-hardening.md) - Complete security analysis
- 🔒 [Option 1: In-Process MCP Server](docs/options/option1-InProcess.md) - Maximum security
- 🔒 [Option 2: MCP over DIDComm Transport](docs/options/option2-McpDidcomm.md) - Recommended approach
- 🔒 [Option 3: HTTPS + mTLS](docs/options/option3-HttpsMtls.md) - Traditional security
- 🔒 [Options Comparison](docs/options/optionsComparison.md) - Decision guide

---

## What's New

### Architecture (Current)
- ✅ **Separated concerns**: MCP protocol vs DIDComm security
- ✅ **Persistent bridge DID**: Saved and reused across restarts
- ✅ **Lightweight MCP servers**: No DIDComm overhead
- ✅ **Better scalability**: One bridge, many MCP servers
- ✅ **Centralized security**: All credentials in bridge

### Why This Matters
Traditional approaches bundled everything together. The bifurcated architecture:
1. **MCP Server** handles AI agent communication (stdio/JSON-RPC)
2. **Bridge** handles security and DIDComm (persistent, trusted)
3. **Service** stores passwords (backend)

This separation enables better security, monitoring, and scalability.

---

## License

MIT License - See LICENSE file for details

---

## Support

- 📖 [Architecture Flows](docs/architecture/architecture-flows.md)
- 📖 [MCP/DIDComm Architecture](docs/architecture/mcp-didcomm-architecture.md)
- 🔒 [Security Options](docs/options/optionsComparison.md)
- 💬 Open an issue for questions

**Built with**:
- [Affinidi TDK](https://github.com/affinidi/affinidi-tdk) - DIDComm implementation
- [rmcp](https://github.com/modelcontextprotocol/rust-sdk) - Official MCP SDK
