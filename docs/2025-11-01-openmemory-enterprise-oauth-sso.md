---
title: OpenMemory Achieves Enterprise-Grade OAuth 2.0 and SSO Authentication
date: 2025-11-01 14:30 +1000
categories: [homelab, infrastructure, authentication]
tags: [openmemory, oauth, sso, authentik, security, mcp, api, automation]
---

## 🎯 Executive Summary

After an **epic multi-session implementation**, OpenMemory has been transformed from a basic memory service into a **production-ready, enterprise-grade, multi-user system** with full OAuth 2.0/OIDC authentication, Single Sign-On (SSO), and automated user provisioning.

> 💡 **Tip:** Want to learn more? Check out the [OpenMemory GitHub repository](https://github.com/mem0ai/mem0) for the open-source project and community resources.

## 🚀 What We Built

### Full OAuth 2.0/OIDC Infrastructure

We implemented a complete OAuth 2.0 authentication system integrated with Authentik, featuring:

- **OAuth Discovery Endpoints** - Automatic client configuration
- **Dynamic Client Registration (DCR)** - Zero manual configuration required
- **PKCE Support** - Enhanced security for authorization code flow
- **JWT Token Validation** - Full token verification with public key cryptography
- **User Access Control** - Email-based authorization with complete isolation

### The OAuth Flow Architecture

```
Claude Desktop (MCP Client)
  ↓ OAuth 2.0 Authorization Flow
Authentik (Identity Provider)  
  ↓ JWT Access Token
OpenMemory API (Resource Server)
  ↓ Validates Token → Serves User Memories
Database (Complete User Isolation)
```

## 🔧 Technical Achievements

### 1. SSO with Auto-Provisioning

Users are automatically created in the database on their first SSO login - **zero manual setup required**. Just log in and start using the system immediately.

### 2. Smart Authentik Configuration

Instead of managing dozens of static redirect URIs, we use **regex patterns** to elegantly handle dynamic ports for MCP clients without configuration sprawl.

### 3. User Isolation and Consistency

- Email is the unique identifier across UI, API, and database
- Complete user isolation - everyone sees only their own data  
- No hardcoded values - fully dynamic configuration

> ℹ️ **Key Insight:** User ID consistency across all layers (UI, API, database, vector storage) is absolutely critical for proper multi-user isolation.

### 4. Professional UI Enhancements

- **14 MCP Client Integrations** - Claude, Claude Code, Cursor, Cline, Roo Cline, Windsurf, VS Code, Codex CLI, Gemini CLI, Factory Droid, Witsy, Enconvo, Augment
- **Dynamic Install Commands** - Real user emails and API URLs (no placeholders!)
- **Professional Branding** - Custom logos for all clients (no emoji icons!)
- **Horizontal Scroll UI** - Smooth navigation across all 14 client tabs

### 5. Custom Install Tooling

We **forked and extended** the official install-mcp tool to add support for Factory AI Droid CLI, enabling OAuth-protected MCP server installation with a single command. Our fork adds seamless integration while maintaining compatibility with all upstream clients.

## 🐛 The Debugging Journey

We encountered and resolved **8+ critical issues** during implementation:

1. ✅ OAuth metadata endpoint failures
2. ✅ Authorization URL routing issues  
3. ✅ JWKS configuration problems
4. ✅ Token validation errors
5. ✅ User provisioning race conditions
6. ✅ Session synchronization bugs
7. ✅ Memory visibility issues
8. ✅ Browser caching surprises

> ⚠️ **Pro Tip:** Always test with `curl` commands first to verify server-side updates. Browser caching is the #1 reason for "why isn't this working?" moments!

## 🎯 Production-Ready Features

This implementation delivers:

- 🔐 **Enterprise-grade authentication** with OAuth 2.0/OIDC
- 👥 **SSO integration** via Authentik and NextAuth  
- ⚡ **Zero-touch provisioning** - users created automatically
- 🎯 **Complete user isolation** - data privacy by design
- 🔧 **Dynamic configuration** - no hardcoded credentials
- 📱 **14 MCP client integrations** - professional logos and commands
- 🚀 **Production deployment ready** - error handling, logging, monitoring

## 🌟 MCP Ecosystem Expansion

As part of this work, we also integrated **10+ working MCP servers** into our infrastructure:

- Proxmox management
- Kubernetes orchestration
- Nextcloud file operations
- Outline documentation (used to write this post!)
- Splunk analytics
- TriliumNext note-taking
- Grist spreadsheet databases
- AppFlowy project management

> ✅ **Meta Moment:** The Outline MCP integration was used to read our Jekyll deployment docs and programmatically generate this blog post!

## 📊 Impact and Results

### Before
- Basic memory service
- Manual user management  
- No authentication
- Single user focus
- Limited client support

### After
- Enterprise-grade multi-user platform
- Automatic user provisioning
- Full OAuth 2.0/OIDC authentication
- Complete user isolation
- 14 MCP client integrations
- Production-ready security
- Professional UI with custom branding

## 🔮 Future Enhancements

With this solid authentication foundation, we can now add:

- **Role-Based Access Control (RBAC)** - Admin, user, and read-only roles
- **Team Workspaces** - Shared memories for collaboration  
- **API Rate Limiting** - Per-user quotas
- **Audit Logging** - Track all authentication events
- **OAuth Scopes** - Fine-grained permission control

## 🎓 Key Takeaways

1. **Authentication is foundational** - Get it right early
2. **Test incrementally** - Catch issues before they compound
3. **Consistency matters** - User IDs must be consistent everywhere
4. **Regex is powerful** - Dynamic URIs beat static configuration
5. **Professional polish matters** - Real logos > emoji icons

## 🏆 Conclusion

This implementation represents a **major milestone** in OpenMemory's evolution. What started as a simple memory service is now a **production-ready, enterprise-grade platform** with authentication that rivals commercial products.

The combination of OAuth 2.0, SSO, auto-provisioning, complete user isolation, and support for 14+ MCP clients makes OpenMemory suitable for **real-world deployment** in professional environments.

---

**Technical Stack:**
- OpenMemory API (FastAPI)
- Authentik (SSO Provider)
- NextAuth (UI Authentication)  
- PyJWT (Token Validation)
- PostgreSQL (User Database)
- Qdrant (Vector Storage)

**MCP Integrations:**
- 14 Client types supported
- OAuth 2.0 for all MCP connections
- Custom install tooling with Droid support

**Status:** ✅ Production Ready

**Built with:** ❤️ and lots of debugging at Oztek Lab
