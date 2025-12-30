# Gibbon - Architecture Decisions

> **Purpose:** Record key architecture decisions and their rationale
> **Format:** ADR (Architecture Decision Record)

---

## ADR-001: Tech Stack

**Date:** 2025-12-27  
**Status:** Accepted

### Decision
- **Frontend:** Next.js 14 (App Router)
- **Backend:** Next.js API routes + Supabase
- **Database:** Supabase (PostgreSQL)
- **Auth:** Supabase Auth
- **Hosting:** Vercel
- **AI:** Anthropic Claude API
- **Compute:** Modal (Python/bash execution)

### Rationale
- Next.js + Vercel = zero-config deployment, excellent DX
- Supabase = Postgres + Auth + Realtime in one, generous free tier
- Modal = serverless Python without managing infrastructure
- All services have good free tiers for MVP

### Consequences
- Vendor lock-in to Vercel/Supabase (acceptable for speed)
- Limited to Node.js runtime for API routes
- Modal adds latency for script execution (~500ms cold start)

---

## ADR-002: No Kafka / No Message Queue

**Date:** 2025-12-30  
**Status:** Accepted

### Context
Considered whether Gibbon needs Kafka or a message queue for:
- Long-running tool execution
- Agent-to-agent communication
- Event sourcing
- Async job processing

### Decision
**Do not use Kafka or any external message queue.**

Use instead:
- **Supabase Realtime** for live updates
- **Modal** for async compute
- **Database tables** for job state (`tool_executions`)
- **Polling or Realtime subscriptions** for agent coordination

### Rationale

| Factor | Kafka | Our Approach |
|--------|-------|--------------|
| Scale needed | Millions/sec | 1-10 users |
| Operational cost | High (manage clusters) | Zero (managed services) |
| Monthly cost | $100-500+ | $0 (free tiers) |
| Complexity | High | Low |
| Latency | Adds hops | Direct |

Kafka solves problems we don't have:
- We're not processing millions of events
- We don't need guaranteed delivery across microservices
- We don't need event replay/sourcing
- We don't have partition/consumer scaling concerns

### Alternatives Considered

| Option | Verdict |
|--------|---------|
| Kafka | Overkill, expensive, complex |
| RabbitMQ | Still overkill for our scale |
| AWS SQS | Adds AWS dependency |
| Redis pub/sub | Possible, but Supabase Realtime suffices |
| Bull/BullMQ | Good if we need job queues later |
| pg-boss | Good if we need job queues later |
| Inngest | Good for complex workflows later |
| Temporal | Enterprise-grade, overkill now |

### When to Revisit
- 1000+ concurrent users
- Need for complex multi-step workflows with retries
- Distributed microservices architecture
- Event sourcing requirements

### Consequences
- Simpler architecture, faster development
- May need to add job queue later (pg-boss or Bull)
- Agent coordination via database polling (slightly less elegant)

---

## ADR-003: GitHub as Source of Truth

**Date:** 2025-12-27  
**Status:** Accepted

### Decision
All code artifacts live in GitHub. Gibbon agents read/write via GitHub API.

### Rationale
- Version control built-in
- PRs enable review workflows
- Actions enable CI/CD
- Familiar to developers
- No need to build our own file storage

### Consequences
- Dependent on GitHub API rate limits
- Need PAT with appropriate permissions
- Per-project GitHub config required

---

## ADR-004: Per-Project GitHub Config

**Date:** 2025-12-30  
**Status:** Accepted

### Decision
- GitHub token stored at **workspace** level (one token per workspace)
- GitHub repo/branch stored at **project** level (each project can target different repo)

### Rationale
- Most users have one GitHub account (workspace token)
- Different projects = different repos (natural mapping)
- Allows dogfooding: Gibbon project targets Gibbon repo

### Schema
```sql
workspaces.github_token  -- encrypted, shared by all projects
projects.github_repo     -- e.g., "owner/repo"
projects.github_branch   -- default branch, e.g., "main"
```

### Consequences
- All projects share same GitHub permissions
- Can't have projects with different GitHub accounts (acceptable)
- Token stored in database (should encrypt at rest later)

---

## ADR-005: Tool Loop Architecture

**Date:** 2025-12-30  
**Status:** Accepted

### Decision
Implement multi-turn tool execution loop:
1. Send message + tools to Claude
2. If Claude requests tool_use, execute tool
3. Send tool_result back to Claude
4. Repeat until text response or max iterations (10)

### Rationale
- Matches Claude's native tool use protocol
- Enables multi-step agent workflows
- Max iterations prevents infinite loops
- Audit logging at each step

### Consequences
- Longer latency for multi-tool responses
- Token usage increases with each loop
- Need to handle partial failures gracefully

---

## ADR-006: Participants = Humans + Agents

**Date:** 2025-12-27  
**Status:** Accepted

### Decision
Both humans and AI agents are "participants" with:
- Same profile structure
- Same memory system
- Same message format

### Rationale
- Simplifies architecture (one participant model)
- Enables agent-to-agent communication
- Memory works the same for all
- Future: agents can have their own workspaces

### Consequences
- Need `participant_type` field to distinguish
- Agent "users" don't have auth credentials
- Some UI paths only make sense for humans

---

## Future Decisions (Not Yet Made)

| Topic | Options | When to Decide |
|-------|---------|----------------|
| Token encryption | Supabase Vault, AWS KMS, app-level | Before production |
| Multi-tenant isolation | RLS only, schema per tenant | If SaaS model |
| Agent orchestration | Simple loop, LangGraph, CrewAI | When workflows get complex |
| File storage | GitHub only, Supabase Storage, S3 | When non-code artifacts needed |
| Search | Postgres full-text, pgvector, Algolia | When search matters |

---

*Last updated: 2025-12-30*
