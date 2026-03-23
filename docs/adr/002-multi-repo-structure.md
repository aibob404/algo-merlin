# ADR-002: Multi-Repo Structure

**Date:** 2025-03
**Status:** Accepted

## Context

The system has 4 distinct components with different tech stacks (Kotlin, Python, React, Docs).
We need to decide between a monorepo and multi-repo approach.

## Decision

Use **separate repositories** per service:
- `algo-merlin` — docs, system design, docker-compose
- `algo-merlin-platform` — Kotlin backend
- `algo-merlin-ai` — Python AI service
- `algo-merlin-ui` — React frontend

## Consequences

**Positive:**
- Each service can be deployed, versioned, and scaled independently
- Cleaner separation of concerns
- Different teams/contributors can own different repos
- CI/CD pipelines are simpler per repo

**Negative:**
- Cross-service changes require multiple PRs
- Docker Compose in `algo-merlin` must be kept in sync manually
- More repos to manage

## Mitigation

- `algo-merlin` is the source of truth for architecture and docker-compose
- API contracts documented in `algo-merlin/docs/api-contracts.md`
- Changes to API contracts require updating docs first
