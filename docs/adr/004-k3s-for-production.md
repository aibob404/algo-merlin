# ADR-004: K3s for Production Deployment

**Date:** 2025-03
**Status:** Accepted

## Context

The system needs a production deployment strategy for a single VPS. Options considered:
- Docker Compose — simple but no autoscaling, secrets management, or self-healing
- Full Kubernetes (kubeadm) — powerful but heavy for a single VPS
- K3s — lightweight Kubernetes distribution
- Docker Swarm — middle ground, but limited ecosystem

## Decision

Use **K3s** on the VPS for production with **two environments** (prod, test),
managed via **Kustomize** overlays. Docker Compose is kept for local development only.

## Rationale

K3s is the right fit because:
- Single binary, installs in under 60 seconds
- Runs comfortably on 2GB+ RAM (vs 4GB+ for full K8s)
- Full Kubernetes API — all standard `kubectl` commands work
- Built-in Traefik ingress with automatic SSL via Let's Encrypt
- Native K8s Secrets — no plaintext `.env` files in production
- HPA (Horizontal Pod Autoscaler) for AI service under load
- Supports future growth — can add nodes to the cluster

## Two Environments

| Setting | `test` | `prod` |
|---------|--------|--------|
| Namespace | `algo-merlin-test` | `algo-merlin-prod` |
| Image tag | `latest` | `stable` |
| ai-service replicas | 1 | 2 (+ HPA up to 5) |
| Resource limits | reduced | full |
| Ingress | `test.yourdomain.com` | `yourdomain.com` |
| BingX | demo | demo (live later) |

## K8s Structure

```
k8s/
├── base/               # Shared manifests
│   ├── postgres/
│   ├── platform/
│   ├── ai-service/
│   └── ui/
└── overlays/
    ├── prod/           # kubectl apply -k k8s/overlays/prod
    └── test/           # kubectl apply -k k8s/overlays/test
```

## Secrets Management

Secrets are created manually via `kubectl create secret` — never stored in Git.
See `k8s/secrets/README.md` for exact commands.

## Consequences

**Positive:**
- Production-grade from day one
- Secrets never in `.env` files on server
- AI service autoscales under load (multiple bots running simultaneously)
- Rolling deployments — zero downtime updates
- Self-healing — pods restart automatically on crash
- Test env allows safe staging before prod deploy

**Negative:**
- More setup than Docker Compose
- K3s must be installed and configured on VPS before first deploy

## Local Development

Docker Compose is used for local development only — same service topology, faster iteration.
