# Dogfooding Tracker

> **Purpose:** Track parity between Claude's current capabilities and Gibbon's features
> **Goal:** Gibbon should eventually provide the same (or better) tooling that Claude uses today
> **Updated:** 2025-12-30 (Sprint 3.5 planned)

---

## Legend

| Status | Meaning |
|--------|---------|
| ✅ | Gibbon can do this |
| 🟡 | Partially implemented |
| ❌ | Claude can, Gibbon cannot |
| 🔮 | Future capability (neither has) |

---

## 1. Code & File Operations

| Capability | Claude | Gibbon | Gap |
|------------|--------|--------|-----|
| Read GitHub files | ✅ API | 🟡 Sprint 3 | Needs tool loop wiring |
| Write GitHub files | ✅ API | 🟡 Sprint 3 | Needs tool loop wiring |
| List directories | ✅ API | 🟡 Sprint 3 | Needs tool loop wiring |
| Create branches | ✅ API | 🟡 Sprint 3 | Needs tool loop wiring |
| Create PRs | ✅ API | 🟡 Sprint 3 | Needs tool loop wiring |
| Merge PRs | ✅ API | ❌ | Add to GitHub tools |
| Run local builds | ✅ bash | ❌ | Needs compute (Modal?) |
| Edit files (str_replace) | ✅ native | ❌ | Implement in tool loop |
| View images | ✅ native | ❌ | Add to chat UI |

---

## 2. Execution Environment

| Capability | Claude | Gibbon | Gap |
|------------|--------|--------|-----|
| Run Python scripts | ✅ bash | 🟡 Modal | Modal tool created, not wired |
| Run bash commands | ✅ native | ❌ | Needs secure sandbox |
| Install packages | ✅ pip/npm | ❌ | Part of Modal? |
| Persistent filesystem | ✅ /home/claude | ❌ | Workspace concept |
| Download/upload files | ✅ curl/API | ❌ | Add fetch tool |

---

## 3. External APIs

| Capability | Claude | Gibbon | Gap |
|------------|--------|--------|-----|
| GitHub API | ✅ full | 🟡 Sprint 3 | 7 tools created |
| Supabase API | ✅ REST | ✅ native | Built-in |
| Supabase Management API | ✅ REST | ❌ | Add as tool |
| Vercel API | ✅ REST | ❌ | Add as tool |
| Modal API | ✅ REST | 🟡 Sprint 3 | execute_script created |
| Penpot API | ✅ REST | ❌ | Sprint 4 (UX Agent) |
| Web search | ✅ native | ❌ | Add search tool |
| Web fetch | ✅ native | ❌ | Add fetch tool |

---

## 4. GitHub Features

| Capability | Claude | Gibbon | Gap |
|------------|--------|--------|-----|
| Manage secrets | ✅ API | ❌ | Add secrets tool |
| Manage variables | ✅ API | ❌ | Add variables tool |
| Create releases | ✅ API | ❌ | Add release tool |
| Trigger workflows | ✅ API | ❌ | Add workflow tool |
| Create environments | ✅ API | ❌ | Add environment tool |
| Branch protection | ❌ (need Pro) | ❌ | Requires GitHub Pro for private repos |
| Issues/tickets | ✅ API | ❌ | Add issues tool |
| Deployments status | ✅ API | ❌ | Add deployments tool |

---

## 5. Chat & Memory

| Capability | Claude | Gibbon | Gap |
|------------|--------|--------|-----|
| Streaming responses | ✅ native | ✅ Sprint 2 | Done |
| Conversation history | ✅ native | ✅ Sprint 2 | Done |
| Session management | ✅ projects | ✅ Sprint 2 | Done |
| Memory extraction | ✅ native | ✅ Sprint 2 | Memory Agent |
| Search past chats | ✅ native | ❌ | Add search |
| Multi-turn tool use | ✅ native | 🟡 Sprint 3 | Tool loop created |

---

## 6. Design & UI

| Capability | Claude | Gibbon | Gap |
|------------|--------|--------|-----|
| Create artifacts (HTML/React) | ✅ native | ❌ | Add artifact system |
| Create documents (docx/pptx) | ✅ skills | ❌ | Add doc generation |
| Create diagrams (Mermaid) | ✅ native | ❌ | Add rendering |
| Design in Penpot | ❌ | 🔮 Sprint 4 | UX Agent |
| View/preview designs | ❌ | 🔮 Sprint 4 | Artifact viewer |

---

## 7. Orchestration

| Capability | Claude | Gibbon | Gap |
|------------|--------|--------|-----|
| Role separation (Planning/Execution) | ✅ Projects | 🟡 | Via specialist agents |
| Handoff between roles | ✅ manual | ❌ | Agent-to-agent messaging |
| Session goals | ❌ | ✅ Sprint 2 | Gibbon has this! |
| Approval workflows | ❌ | 🔮 | GitHub environments |
| Parallel execution | ❌ | 🔮 | Multiple agents |

---

## Sprint Status

### Sprint 3: Agent Autonomy ✅ Complete (v0.4.0)
- [x] GitHub tools (7 operations)
- [x] Modal execute tool
- [x] Tool loop orchestration
- [x] Tool call UI component

### Sprint 3.5: Tool Activation ✅ Complete (v0.5.0)
- [ ] Wire tool loop to chat completion
- [ ] Per-project GitHub config
- [ ] Render tool calls in UI
- [ ] Project/workspace settings

### What's Needed to Match Claude:
1. ~~Wire tool loop into chat route~~ → Sprint 3.5
2. Add more GitHub tools (issues, releases)
3. Add web fetch/search tools
4. Add Vercel/Supabase management tools
5. Artifact system for generated content

---

## Prioritized Gap List

| Priority | Gap | Effort | Status |
|----------|-----|--------|--------|
| P0 | Wire tool loop to chat | 2h | ✅ Done |
| P0 | Per-project GitHub config | 4h | ✅ Done |
| P0 | Tool call UI in chat | 2h | ✅ Done |
| P0 | Project settings page | 2h | ✅ Done |
| P1 | Add issues tool | 2h | Backlog |
| P1 | Add web fetch tool | 2h | Backlog |
| P2 | Add releases tool | 2h | Backlog |
| P2 | Add Vercel deploy tool | 4h | Backlog |
| P2 | Artifact system | 8h | Backlog |
| P3 | Penpot integration | 16h | Sprint 4 |

---

## Metrics

| Metric | Current | Target |
|--------|---------|--------|
| Tools Claude has | ~15 | - |
| Tools Gibbon has | 8 | 15+ |
| Tool categories covered | 2/7 | 7/7 |
| E2E workflows working | 1 | 3+ |

---

*This doc tracks progress toward Gibbon being self-sufficient for its own development.*
