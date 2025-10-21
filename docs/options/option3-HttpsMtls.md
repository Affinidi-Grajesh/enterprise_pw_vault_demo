# Option 3: HTTPS + mTLS

**Best for: Traditional enterprise security with certificate-based authentication**

---

## 🎯 Overview

Secure the existing bridge architecture using HTTPS with mutual TLS (mTLS). This approach uses traditional certificate-based authentication while moving DID ownership to the agent.

---

## 🏗️ Architecture

```
┌──────────────┐   stdio      ┌──────────────┐   HTTPS+mTLS ┌──────────────┐   DIDComm    ┌──────────────┐
│  AI Agent    │  JSON-RPC    │  MCP Server  │  encrypted   │   DIDComm    │  encrypted   │  Password    │
│  (Claude)    │ ──────────── │ (has agent   │ ──────────── │   Bridge     │ ──────────── │  Service     │
│              │              │  DID)        │   network    │ (relay only) │              │              │
└──────────────┘              └──────────────┘              └──────────────┘              └──────────────┘
                                      │                              │
                                      │     TLS Certificate          │
                                      │     Mutual Authentication    │
                                      └──────────────────────────────┘
```

---

## ✅ Benefits

| Benefit | Description |
|---------|-------------|
| **TLS encryption** | Standard HTTPS protects transport layer |
| **mTLS authentication** | Both client and server verify identities |
| **Agent owns DID** | MCP server manages agent's identity |
| **Bridge as relay** | Bridge doesn't decrypt messages |
| **Familiar ops model** | Traditional certificate management |
| **Auditable** | Standard HTTPS logging |

---

## 🔒 Security Properties

### Encryption Layers
```
Application Layer:  MCP JSON-RPC
                    ↓
DIDComm Layer:      End-to-end encryption
                    ↓
TLS Layer:          Transport encryption + authentication
                    ↓
TCP Layer:          Network transport
```

### Key Features
- **TLS 1.3**: Modern encryption standards
- **Certificate validation**: Both directions
- **Forward secrecy**: Ephemeral keys per session
- **Bridge isolation**: Cannot see plaintext
- **Standard monitoring**: HTTPS tooling

---

## 📊 Comparison

| Aspect | HTTP (Current) | HTTPS + mTLS |
|--------|----------------|--------------|
| Transport encryption | ❌ Plaintext | ✅ TLS 1.3 |
| Authentication | ❌ None | ✅ Certificate-based |
| DID ownership | ❌ Bridge | ✅ Agent |
| Bridge role | ⚠️ Decrypts | ✅ Relay only |
| Ops complexity | ✅ Simple | ⚠️ Certificate mgmt |
| Performance | ✅ Fast | ⚠️ TLS overhead |

---

## ⚠️ Considerations

### Pros
- ✅ Familiar security model
- ✅ Standard ops tooling
- ✅ TLS encryption
- ✅ Certificate-based auth
- ✅ Agent owns DID

### Cons
- ⚠️ Certificate management overhead
- ⚠️ Still has network exposure
- ⚠️ TLS performance overhead
- ⚠️ Two processes to manage
- ⚠️ Certificate rotation complexity

---

## 🎯 Best For

- **Traditional enterprises** with PKI infrastructure
- **Regulated industries** requiring standard protocols
- **Operations teams** familiar with TLS
- **Scenarios** where certificate management exists
- **Migration path** from current HTTP setup

---

## 🔗 Related Options

- [Option 1: In-Process MCP Server](option1-InProcess.md) - Maximum security
- [Option 2: MCP over DIDComm Transport](option2-McpDidcomm.md) - Native encryption

---

## 📚 Next Steps

1. Review certificate management processes
2. Generate test certificates
3. Update bridge with TLS
4. Test with mTLS client
5. Document certificate rotation
