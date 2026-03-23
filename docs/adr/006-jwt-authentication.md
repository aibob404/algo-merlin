# ADR-006: JWT Authentication for REST API + Shared Secret for Tool Execution

**Date:** 2025-03
**Status:** Accepted

## Context

Two authentication gaps identified:
1. The Platform REST API (`/api/*`) has no authentication — any actor can create bots, place orders, view trade history
2. The `/api/tools/execute` endpoint (called by AI Service) has no authentication — any actor on the network can place orders

## Decision

### REST API: JWT Authentication
- `POST /auth/login` accepts username + password, returns signed JWT (24h expiry)
- All `/api/*` endpoints require `Authorization: Bearer {JWT}`
- JWT validated on every request (signature + expiry check)
- MVP: single admin user stored in env/secret (no user management DB)

### Tool Execution: Shared Secret Token
- Both Platform and AI Service receive `TOOL_EXECUTION_TOKEN` from K8s Secret
- AI Service includes `Authorization: Bearer {TOOL_EXECUTION_TOKEN}` in every `/api/tools/execute` call
- Platform validates the token on every request
- Simpler than mTLS for MVP, sufficient for internal-only traffic

## Consequences

**Positive:**
- REST API protected from unauthorized access
- Tool execution endpoint not callable by external actors
- Simple implementation — no external auth provider needed for MVP

**Negative:**
- Shared secret is symmetric — if compromised, must rotate both services simultaneously
- No per-user audit trail (MVP uses single admin user)
- mTLS would be stronger — considered for Phase 2 via service mesh (Istio)
