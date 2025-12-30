# Gibbon - Backlog

> **Last Updated:** 2025-12-29
> **Managed By:** Planning Sessions

## Priority Legend
- 🔴 **P0:** Critical / Blocking
- 🟠 **P1:** High / This Sprint
- 🟡 **P2:** Medium / Next Sprint
- 🟢 **P3:** Low / Future
- ⚪ **P4:** Nice to Have

---

## Immediate Tasks

| Task | Priority | Due | Notes |
|------|----------|-----|-------|
| Review Modal billing | 🟡 P2 | 2025-01-02 | Check Modal dashboard for usage costs after a few days of use |
| Review Figma integration needs | 🟢 P3 | 2025-01-05 | Evaluate what Figma API access enables for Gibbon |

---

## Discovery Queue

| Topic | Question | Added |
|-------|----------|-------|
| **Meta-orchestration: LLM prompt engineering** | Should a smarter LLM curate context/prompts for a cheaper executor, or should a cheaper LLM curate for a smarter reasoner? Where does leverage live — in framing or in reasoning? | 2025-12-30 |
| **Local LLM Training Prep** | What data format works best for fine-tuning? (JSONL, conversations, instruction pairs?) | 2025-12-30 |
| **Local LLM Training Prep** | How much training data is needed before fine-tuning is worthwhile? (1K, 10K, 100K examples?) | 2025-12-30 |
| **Local LLM Training Prep** | Which base model to fine-tune? (Llama 3, Mistral, Phi-3, Qwen?) | 2025-12-30 |
| **Local LLM Training Prep** | What's the minimum viable fine-tune? (LoRA vs full, cloud vs local?) | 2025-12-30 |
| **Local LLM Training Prep** | How to evaluate fine-tuned model quality? (benchmarks, A/B testing?) | 2025-12-30 |
| **Artifact Gap** | Why don't GitHub file operations appear in the Artifacts panel? Need GitHub→Artifacts sync | 2025-12-30 |


---


## Epic Overview

| Epic | Name | Description | Sprint |
|------|------|-------------|--------|
| E1 | Foundation | Auth, workspaces, users, roles | 1 ✅ |
| E2 | Projects & Artifacts | Projects, versioned artifacts | 1 ✅ |
| E3 | Tasks | Task board, states, assignments | 1 ✅ |
| E4 | Chat & Sessions | Chat UI, purpose-driven sessions, multi-party | 2 |
| E5 | Memory System | Session memory, project memory, Memory Agent | 2 |
| E6 | Agent Chat | LLM integration, Lead Agent, proposals, drift detection | 2 |
| E7 | GitHub Tools | Read/write files, branches, PRs via agent tools | 4 |
| E8 | Execution Tools | Run commands, tests, lint via Modal | 4 |
| E9 | QA Tools | Browser automation, screenshots, assertions via Modal | 4 |
| E10 | Unified Participants | Humans + agents as same entity type | 4 |
| E11 | Config from Repo | .gibbon/ directory, agent definitions in repo | 4 |
| E12 | Artifacts from GitHub | GitHub as source of truth, not Supabase | 4 |
| E13 | Recipes | Product templates, infrastructure configs | 4+ |
| E14 | Vercel Integration | Deploy status (read-only) | 4+ |
| E15 | UX Agent System | UX Agent, component library, Penpot integration | 3 |

---

## Completed: Sprint 1 ✅

### E1: Foundation
| ID | Story | Status |
|----|-------|--------|
| E1.1 | Sign up / log in (Supabase Auth) | ✅ |
| E1.2 | Create workspace | ✅ |
| E1.3 | Invite users to workspace | ✅ |
| E1.4 | Assign roles (Owner, Member, Viewer) | ✅ |

### E2: Projects & Artifacts
| ID | Story | Status |
|----|-------|--------|
| E2.1 | Create project in workspace | ✅ |
| E2.2 | View project artifacts | ✅ |
| E2.3 | View artifact version history | ✅ |
| E2.4 | Archive project | ✅ |

### E3: Tasks
| ID | Story | Status |
|----|-------|--------|
| E3.1 | Create task in project | ✅ |
| E3.2 | Task board view (Proposed → Done) | ✅ |
| E3.3 | Approve/reject tasks | ✅ |
| E3.4 | Assign task to user | ✅ |

---

## Sprint 2: Chat & Sessions 🟠

**Goal:** Chat-first UI with purpose-driven sessions, multi-party collaboration, and memory system.

**Critical Rule:** Every code change must pass `npm run build` before commit.

### E4: Chat & Sessions

#### 4A: Chat Foundation
| ID | Story | Acceptance Criteria | Est |
|----|-------|---------------------|-----|
| E4.1 | Fixed layout (chat left 40%, context right 60%) | Layout never changes; right panel view-only | 2h |
| E4.2 | Session creation with goal prompt | "What would you like to achieve?" dialog | 2h |
| E4.3 | Session start/end (explicit) | `gibbon start/end session` commands + button | 2h |
| E4.4 | Tabbed sessions | Multiple active sessions as tabs; unread indicator | 2h |
| E4.5 | Session history | Closed sessions readable, not editable | 2h |
| E4.6 | Participant list in session header | Shows humans + agents with icons | 1h |

#### 4B: Multi-Party
| ID | Story | Acceptance Criteria | Est |
|----|-------|---------------------|-----|
| E4.7 | Multiple participants in session | Humans + agents in same session | 2h |
| E4.8 | Agent invocation with consent | Lead requests → user approves → agent joins | 2h |
| E4.9 | Human invocation | Any participant can invite another human | 2h |
| E4.10 | Message attribution | Every message shows sender + timestamp | 1h |
| E4.11 | System events | "Agent joined", "User left" shown inline | 1h |

### E5: Memory System
| ID | Story | Acceptance Criteria | Est |
|----|-------|---------------------|-----|
| E5.1 | Session Memory artifact | Visible in right panel Memory tab | 2h |
| E5.2 | Project Memory artifact | Persists across sessions | 2h |
| E5.3 | Include/exclude memory items | Checkbox toggles context inclusion | 1h |
| E5.4 | Remove memory items | Delete items entirely | 1h |
| E5.5 | Promote session → project memory | Copy item to project level | 1h |
| E5.6 | Manual add memory items | User can add items directly | 1h |
| E5.7 | Memory Agent (background) | Observes sessions, extracts items | 3h |
| E5.8 | Confidence-based suggestions | ≥80% auto; 60-80% suggest; <60% silent | 2h |
| E5.9 | Session end summary | Lead summarizes; Memory Agent suggests promotions | 2h |

### E6: Agent Chat
| ID | Story | Acceptance Criteria | Est |
|----|-------|---------------------|-----|
| E6.1 | Lead Agent as default | Every session starts with Lead Agent | 1h |
| E6.2 | LLM integration (Anthropic API) | Send message → get response | 3h |
| E6.3 | Agent proposes artifact changes | Shows diff link in chat | 2h |
| E6.4 | Diff view in context panel | Right panel shows proposed changes | 2h |
| E6.5 | Accept/reject via chat | `accept` / `reject` commands work | 2h |
| E6.6 | Accepted changes create artifact version | Version linked to session | 2h |
| E6.7 | Agent creates tasks | Proposes task → appears on board when accepted | 2h |
| E6.8 | Drift detection | Lead notices topic drift; offers options | 3h |

**Sprint 2 Total: ~52 hours**

---

## Sprint 4: Agent Tools & Infrastructure 🟢

**Goal:** Agents become fully capable. Can read/write code, execute commands, test UI, and act autonomously.

**Prerequisite:** Modal account + API keys

### E7: GitHub Tools
| ID | Story | Acceptance Criteria | Est |
|----|-------|---------------------|-----|
| E7.1 | Project links to GitHub repo | `github_repo` field on project | 1h |
| E7.2 | `github_read_file` tool | Agent can read any file from repo | 2h |
| E7.3 | `github_write_file` tool | Agent can commit changes to repo | 2h |
| E7.4 | `github_list_files` tool | Agent can list directory contents | 1h |
| E7.5 | `github_create_branch` tool | Agent can create feature branches | 2h |
| E7.6 | `github_create_pr` tool | Agent can open pull requests | 2h |
| E7.7 | `github_get_pr_status` tool | Agent can check PR status | 1h |

### E8: Execution Tools (Modal)
| ID | Story | Acceptance Criteria | Est |
|----|-------|---------------------|-----|
| E8.1 | Modal account + function deployment | Functions deployed to Modal | 2h |
| E8.2 | `execute_command` tool | Agent can run shell commands | 3h |
| E8.3 | `run_tests` tool | Agent can run test suite | 2h |
| E8.4 | `lint_code` tool | Agent can run linter | 1h |
| E8.5 | Tool execution logging | All tool calls logged with input/output/status | 2h |
| E8.6 | Tool execution UI | See tool calls in chat as expandable blocks | 2h |

### E9: QA Tools (Modal + Playwright)
| ID | Story | Acceptance Criteria | Est |
|----|-------|---------------------|-----|
| E9.1 | Deploy Playwright Modal functions | Chromium available in Modal | 2h |
| E9.2 | `browser_open` tool | Agent opens URL, gets screenshot + text | 3h |
| E9.3 | `browser_interact` tool | Agent can click, type, wait on page | 3h |
| E9.4 | `browser_check` tool | Agent runs assertions on page | 2h |
| E9.5 | Screenshots visible in chat | Base64 images rendered inline | 2h |
| E9.6 | QA verification after deploy | Agent verifies feature works after commit | 2h |

### E10: Unified Participants
| ID | Story | Acceptance Criteria | Est |
|----|-------|---------------------|-----|
| E10.1 | `participants` table (unified) | Humans + agents in same table with `type` | 3h |
| E10.2 | `participant_profiles` table | Memories, lessons for both humans and agents | 2h |
| E10.3 | `agent_configs` table | System prompt, model, tools per agent | 2h |
| E10.4 | Migrate existing users | Sprint 1-2 users become participants | 2h |
| E10.5 | Profile view in UI | See any participant's profile, memories | 3h |

### E11: Config from Repo
| ID | Story | Acceptance Criteria | Est |
|----|-------|---------------------|-----|
| E11.1 | `.gibbon/config.yml` schema | Defines agents, artifacts, default context | 2h |
| E11.2 | Load agent config from repo | Agents defined in repo, not just Supabase | 3h |
| E11.3 | Prompt files in repo | System prompts in `.gibbon/prompts/*.md` | 2h |
| E11.4 | Context builder uses config | Default artifacts loaded per config | 2h |

### E12: Artifacts from GitHub
| ID | Story | Acceptance Criteria | Est |
|----|-------|---------------------|-----|
| E12.1 | Read artifacts from GitHub | Context panel reads files from repo | 3h |
| E12.2 | Accept = git commit | "accept" command commits to branch | 2h |
| E12.3 | Remove Supabase artifacts tables | Artifacts/versions tables deleted | 2h |
| E12.4 | Version history from git log | Show commit history for artifact | 2h |

### Tool Orchestration
| ID | Story | Acceptance Criteria | Est |
|----|-------|---------------------|-----|
| E.Orch.1 | Tool executor service | Routes tool requests to GitHub/Modal | 3h |
| E.Orch.2 | Tool loop in chat completion | Agent can call multiple tools per response | 3h |
| E.Orch.3 | Error handling | Tool failures shown to agent, can retry | 2h |
| E.Orch.4 | Timeout handling | Long-running tools handled gracefully | 2h |

**Sprint 4 Total: ~74 hours**

---

## Sprint 5+: Recipes & Polish 🟢

### E13: Recipe System
| ID | Story | Acceptance Criteria |
|----|-------|---------------------|
| E13.1 | See available recipes when creating project | Recipe list with descriptions |
| E13.2 | Select recipe for new project | Project seeded with recipe config |
| E13.3 | Recipe seeds `.gibbon/` directory | Agent definitions, artifact types pre-populated |
| E13.4 | Recipe includes deployment flow docs | How-to guide in project |
| E13.5 | Web App (CRUD) recipe | Next.js + Supabase + Vercel |
| E13.6 | Data Analytics recipe | Supabase + Modal + Metabase |

### E14: Vercel Integration
| ID | Story | Acceptance Criteria |
|----|-------|---------------------|
| E14.1 | Connect Vercel project | Token + project ID stored |
| E14.2 | See deploy status | Badge: building, ready, error |
| E14.3 | `vercel_get_status` tool | Agent can check deployment status |
| E14.4 | Auto-wait for deploy | Agent waits for deploy before QA |

---


---

## Sprint 3: UX Agent System 🟠 CURRENT

**Goal:** UX Agent can create designs programmatically, manage component library, and integrate with Gibbon's artifact viewer.

**Prerequisites:** Sprint 3 complete (GitHub + Modal tools working)

**Reference:** See `docs/gibbon/UX_AGENT.md` for full specification.

### E15: UX Agent Foundation

#### 15A: Penpot Integration
| ID | Story | Acceptance Criteria | Est |
|----|-------|---------------------|-----|
| E15.1 | Penpot API client | `lib/penpot/client.ts` with auth, error handling | 3h |
| E15.2 | Shape generator helpers | `lib/penpot/shapes.ts` - rect, frame, text creators | 4h |
| E15.3 | UX Agent tools | `penpot_create_file`, `penpot_add_shapes`, `penpot_use_component` | 4h |
| E15.4 | Add UX Agent to system | Agent record in database with system prompt | 2h |

#### 15B: Component Library
| ID | Story | Acceptance Criteria | Est |
|----|-------|---------------------|-----|
| E15.5 | Component schema | JSON schema for component definitions | 2h |
| E15.6 | Initial components | buttons, inputs, cards, layout primitives | 6h |
| E15.7 | Component loader | Load components from `.gibbon/components/` | 3h |
| E15.8 | Use component tool | Instantiate component with props at position | 3h |

#### 15C: Design Versioning
| ID | Story | Acceptance Criteria | Est |
|----|-------|---------------------|-----|
| E15.9 | Design inventory | `docs/designs/inventory.json` tracking all designs | 2h |
| E15.10 | Version snapshots | Create snapshots on design changes | 2h |
| E15.11 | Design tokens | Centralized colors, spacing, typography in `.gibbon/design-tokens.json` | 2h |

#### 15D: Gibbon Integration
| ID | Story | Acceptance Criteria | Est |
|----|-------|---------------------|-----|
| E15.12 | Penpot viewer component | Display designs in artifacts panel | 4h |
| E15.13 | Design artifact type | Add "DESIGN" to artifact types | 2h |
| E15.14 | Lead Agent → UX Agent handoff | Lead can invoke UX Agent for design tasks | 3h |

**Sprint 3 Total: ~42 hours**

### First UX Agent Task: Design Gibbon UI

After Sprint 4 infrastructure is complete, first task:

**Goal:** Create comprehensive UI design for Gibbon based on current architecture

**Scope:**
1. **Main Layout** - Chat panel (40%) + Context panel (60%)
2. **Chat Components** - Message bubbles, input, tool call blocks, session tabs
3. **Context Panel** - Artifacts, memory, tasks tabs
4. **Navigation** - Workspace selector, project list, settings
5. **Component Library** - All reusable elements as Penpot components

**Deliverables:**
- Penpot design files with all screens
- Component library in `.gibbon/components/`
- Design tokens in `.gibbon/design-tokens.json`
- Version inventory in `docs/designs/`


## Tool Summary

After Sprint 4, agents have these tools:

### GitHub Tools
| Tool | Description |
|------|-------------|
| `github_read_file` | Read a file from repo |
| `github_write_file` | Write/commit a file |
| `github_list_files` | List directory contents |
| `github_create_branch` | Create a new branch |
| `github_create_pr` | Open a pull request |
| `github_get_pr_status` | Check PR status |

### Execution Tools (Modal)
| Tool | Description |
|------|-------------|
| `execute_command` | Run shell command |
| `run_tests` | Run test suite |
| `lint_code` | Run linter |

### QA Tools (Modal + Playwright)
| Tool | Description |
|------|-------------|
| `browser_open` | Open URL, get screenshot + text |
| `browser_interact` | Click, type, wait on page |
| `browser_check` | Run assertions on page |

---

## The Full Agent Loop

After Sprint 4, an agent can:

```
1. Read context (GitHub tools)
   - Read SCHEMA.md, CURRENT.md, existing code
   
2. Write code (GitHub tools)
   - Create/modify files
   - Commit to branch
   
3. Run tests (Execution tools)
   - npm test, npm run lint
   - Fix issues, re-commit
   
4. Open PR (GitHub tools)
   - Create PR with description
   
5. Verify deployment (QA tools)
   - Open deployed URL
   - Check UI renders correctly
   - Run assertions
   - Take screenshots as proof
   
6. Report back
   - "Feature deployed and verified. Here's the screenshot."
```

---

## Database Schema (Sprint 2)

### New Tables

```sql
-- Sessions
CREATE TABLE sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id),
  goal TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'active',
  created_by UUID NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  closed_at TIMESTAMPTZ
);

-- Session Participants
CREATE TABLE session_participants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id UUID NOT NULL REFERENCES sessions(id),
  participant_type TEXT NOT NULL, -- 'user' or 'agent'
  participant_id UUID NOT NULL,
  joined_at TIMESTAMPTZ DEFAULT NOW()
);

-- Messages
CREATE TABLE messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id UUID NOT NULL REFERENCES sessions(id),
  sender_type TEXT NOT NULL,
  sender_id UUID,
  content TEXT NOT NULL,
  message_type TEXT DEFAULT 'text',
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Memory Items
CREATE TABLE memory_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  scope TEXT NOT NULL, -- 'session' or 'project'
  scope_id UUID NOT NULL,
  content TEXT NOT NULL,
  item_type TEXT NOT NULL,
  source_type TEXT NOT NULL,
  source_id UUID NOT NULL,
  active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Agents (system-defined for now)
CREATE TABLE agents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id UUID,
  name TEXT NOT NULL,
  slug TEXT NOT NULL,
  role TEXT,
  agent_type TEXT NOT NULL,
  system_prompt TEXT,
  is_system BOOLEAN DEFAULT false,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Sprint 3 Additions

```sql
-- Unified participants (replaces users + agents)
CREATE TABLE participants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  type TEXT NOT NULL, -- 'human' or 'agent'
  workspace_id UUID,
  email TEXT UNIQUE,
  name TEXT NOT NULL,
  slug TEXT NOT NULL,
  role_description TEXT,
  is_system BOOLEAN DEFAULT false,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Participant profiles (memories, lessons)
CREATE TABLE participant_profiles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  participant_id UUID NOT NULL REFERENCES participants(id),
  profile_type TEXT NOT NULL, -- 'context', 'memory', 'lesson', 'preference'
  content TEXT NOT NULL,
  active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Agent configs
CREATE TABLE agent_configs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  participant_id UUID NOT NULL REFERENCES participants(id),
  system_prompt TEXT,
  model_provider TEXT DEFAULT 'anthropic',
  model_name TEXT DEFAULT 'claude-sonnet-4-20250514',
  tools_enabled JSONB DEFAULT '[]',
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Tool executions (audit log)
CREATE TABLE tool_executions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id UUID REFERENCES sessions(id),
  participant_id UUID NOT NULL,
  tool_name TEXT NOT NULL,
  tool_input JSONB NOT NULL,
  tool_output JSONB,
  status TEXT DEFAULT 'pending',
  error_message TEXT,
  duration_ms INTEGER,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## Icebox (Parked Ideas)

- Multi-agent orchestration (agents calling agents autonomously)
- Cross-project learning via embeddings/RAG
- Voice/video chat with agents
- Auto-merge capabilities
- GitHub comment integration
- Mobile support
- Agent performance metrics and A/B testing

## 🎨 UX/UI Agent Development

### Task: Implement UX Agent System
**Priority:** High (P1) | **Status:** Sprint 4 | **Added:** 2025-12-30

**Full Specification:** See `docs/gibbon/UX_AGENT.md`

#### Overview
The UX Agent creates and manages UI designs through Penpot, with component library support and version tracking.

#### Prerequisites
- [x] Self-hosted Penpot instance deployed (Elestio)
- [x] API access verified and documented
- [x] Shape creation API working
- [x] Full specification created (UX_AGENT.md)
- [ ] Sprint 3 complete (GitHub + Modal tools)

#### Sprint 4 Work
See E15 epic above for detailed breakdown:
- E15.1-E15.4: Penpot integration (~13h)
- E15.5-E15.8: Component library (~14h)
- E15.9-E15.11: Design versioning (~6h)
- E15.12-E15.14: Gibbon integration (~9h)

#### First Design Task
After infrastructure: Design Gibbon UI
- Main layout (chat + context panels)
- All chat components
- Context panel tabs
- Navigation elements
- Full component library

#### References
- Penpot API: `docs/gibbon/PENPOT.md`
- UX Agent Spec: `docs/gibbon/UX_AGENT.md`
- Session: `docs/gibbon/sessions/2025-12-30-planning-007.md`
