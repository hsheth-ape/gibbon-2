# Gibbon - Lessons Learned

## Format
```markdown
### [DATE] Lesson Title
**Context:** What were we doing?
**Lesson:** What did we learn?
**Action:** What should we do differently?
```

---

## Lessons

### [2025-12-29] Artifacts are Memory, Chat is Ephemeral
**Context:** Planning initial requirements; discussing how Gibbon differs from Claude.ai chats.
**Lesson:** The key insight is that artifacts (versioned docs) should be the source of truth, not chat history. Chat produces artifacts, but artifacts persist. This solves context bleed and makes knowledge explicit and controllable.
**Action:** Design the system so chats are always scoped (to task, project, or workspace) and their value is captured in artifact changes, not in the chat transcript itself.

### [2025-12-29] Three Layers of Work
**Context:** Defining agent types and their roles.
**Lesson:** There are three distinct "hats" that map to three chat contexts: (1) Specialist Agent = task execution, (2) Lead Agent = project planning/coordination, (3) System Agent = evolving the system itself. Each has different context needs and outputs.
**Action:** Build all three chat types, but start simple: System Agent in MVP is just "edit agent definitions + ways of working + lessons via chat."

### [2025-12-29] Capture Data for Future Training
**Context:** Discussing how agents can learn and specialize over time.
**Lesson:** Even without ML, structured logging of every execution (context, output, feedback, outcome) creates a dataset for future training or fine-tuning. The learning loop in MVP is human-curated (update prompts based on lessons), but the data capture enables ML later.
**Action:** Design execution_logs table from day one with all fields needed for future training: context_snapshot, messages, proposed_changes, accepted_changes, feedback, outcome.

### [2025-12-29] Model Flexibility from Day One
**Context:** Discussing cost and open-source LLM options.
**Lesson:** Building with Anthropic API first makes sense (best quality), but the LLM layer should be abstracted so you can mix models: Claude for complex tasks, Llama for routine work, Haiku for simple formatting.
**Action:** Agent definitions should include model_config (provider + model). Build provider interface that can support Anthropic, Ollama, OpenAI, etc.

### [2025-12-29] Recipes Define What Gibbon Can Build
**Context:** Exploring what the first product built with Gibbon should be (Data Analytics app).
**Lesson:** Different products need different stacks, artifacts, and agents. A CRUD app (Vercel + Supabase) is fundamentally different from a Data Analytics app (Modal + Supabase + Metabase). Recipes make these patterns reusable and selectable.
**Action:** Design Recipe as a first-class concept: defines infrastructure, artifact types, suggested agents, deployment flow. Ship MVP with one recipe, but architecture supports many. First additional recipe: Data Analytics.

### [2025-12-29] Chat is the Control Plane, Not a Feature
**Context:** Reviewing Sprint 1 output; realizing an empty task board is intimidating to start with.
**Lesson:** We initially designed artifacts and tasks as the "center" with chat bolted on. But the real flow is: chat produces artifacts and tasks. Chat is the entry point, the thinking space, the control plane. Everything else is output.
**Action:** Sprint 2 redesigned as chat-first: user lands in chat, talks with Lead Agent, agent proposes changes, user accepts via chat. Right panel is view-only — all actions through conversation.

### [2025-12-29] Sessions Need Purpose and Memory Needs to Be Explicit
**Context:** Discussing how to prevent context creep and drift that happens in Claude.ai chats.
**Lesson:** Two key mechanisms: (1) Sessions have explicit goals — drift detection nudges user to refocus or start new session. (2) Memory is an artifact, not magic — users can see exactly what's in context, remove items, add items, promote from session to project. Control solves trust.
**Action:** Build Session Memory and Project Memory as visible, editable artifacts. Memory Agent extracts items with confidence thresholds. User always has final say on what's included.

### [2025-12-29] Multi-Party Chat Creates Real Collaboration
**Context:** Discussing how to bring teams into the human-AI workflow.
**Lesson:** The power isn't just "me + AI" — it's "team + multiple agents" in the same conversation. Humans can invite other humans, Lead Agent can bring in specialists (with consent). Every message is attributed. This makes the collaboration visible and shared.
**Action:** Sessions support multiple participants (humans + agents). Agent invocation requires consent (for now). All messages attributed. System events shown inline ("Backend Agent joined").

### [2025-12-29] GitHub is Source of Truth, Gibbon is the Interface
**Context:** Clarifying where artifacts live — in Supabase (Gibbon's database) or GitHub (the repo)?
**Lesson:** Artifacts should live in GitHub, not Supabase. This gives us: single source of truth, git history as version history, portability (repo works without Gibbon), familiar patterns. Supabase stores collaboration state (sessions, messages, memory, tasks) but not the artifacts themselves.
**Action:** Sprint 3 redesigned: artifacts read from / written to GitHub. "Accept change" = git commit. Agent definitions live in `.gibbon/config.yml` in each product repo.

### [2025-12-29] One Repo Per Product, Self-Contained
**Context:** Figuring out how many repos are needed when Gibbon builds products.
**Lesson:** Each product (including Gibbon itself) is one self-contained repo with: `.gibbon/` (agent config), `docs/` (artifacts), `src/` (code). No separate "meta" repo needed. When Gibbon builds Gibbon, it works from the same repo.
**Action:** The `.gibbon/` directory structure becomes part of the Recipe. Each project repo carries its own agent definitions, process artifacts (WAYS_OF_WORKING, LESSONS), and product artifacts (SCHEMA, REQUIREMENTS, etc.).

### [2025-12-29] Anthropic API is Brain-Only — You Provide the Body
**Context:** Understanding what happens when using API vs Claude.ai.
**Lesson:** Claude.ai provides a full computer (bash, Python, filesystem). Anthropic API provides only the brain — text in, text out. Tools must be built and executed by Gibbon. When Claude wants to read a file, Gibbon calls GitHub API. When Claude wants to run code, Gibbon calls Modal.
**Action:** Build tool executor in Gibbon that handles GitHub tools (read/write/branch/PR) and Modal tools (execute commands, run tests). All tool calls logged for audit.

### [2025-12-29] Humans and Agents Are Architecturally Identical
**Context:** Designing permissions and realizing the model was getting complex.
**Lesson:** The cleanest architecture treats humans and agents as the same entity type: "participants." Both have profiles, memories, lessons learned, and the ability to act. The only difference is agents call Anthropic API while humans type.
**Action:** Single `participants` table with `type` (human/agent). Both get `participant_profiles` for memories and lessons. Agent-specific config (prompts, models) in separate `agent_configs` table.

### [2025-12-29] CRITICAL: Never Commit Without Build Verification
**Context:** Sprint 2 failed catastrophically. Added ~15 files at once without running `npm run build`. Cascading TypeScript errors broke the deployment. Spent hours on failed fix attempts.
**Lesson:** This is the most important lesson. No matter how confident you are, ALWAYS run `npm run build` before committing. One small type error becomes ten when you add more files on top of it. The execution agent must treat build verification as mandatory, not optional.
**Action:** 
1. EVERY code change must pass `npm run build` before commit
2. Break work into atomic steps (one file or one feature at a time)
3. Verify Vercel deployment succeeds before proceeding
4. If build fails, fix it immediately — never commit on top of broken code
5. Add this rule to WAYS_OF_WORKING.md for all future work

### [2025-12-29] GitHub API Works When Git Push Is Blocked
**Context:** In some environments, `git clone` and `git push` fail due to proxy/network restrictions, but the GitHub API is accessible.
**Lesson:** Use the GitHub Contents API to update individual files when standard git operations are blocked. Download repo via tarball, work locally, then push files individually via PUT requests to `/repos/{owner}/{repo}/contents/{path}`.
**Action:**
1. Clone via API: `curl -L "https://api.github.com/repos/OWNER/REPO/tarball/main" -H "Authorization: token TOKEN" | tar -xz`
2. Update file via API: `curl -X PUT "https://api.github.com/repos/OWNER/REPO/contents/PATH" -H "Authorization: token TOKEN" -d '{"message":"...", "content":"BASE64", "sha":"CURRENT_SHA"}'`
3. For new files, omit the `sha` field

---


### [2025-12-30] Session Lifecycle Needs Clear UX
**Context:** Sprint 2 retrospective; discussing session management after implementing chat + memory.
**Lesson:** Closing a session through session management doesn't provide clear feedback in the chat UI. Users can still type/send messages even after "closing" a session. Need to decide: is this a feature (archive mode) or a bug (should block interaction)?
**Action:** Planning decision needed: (1) Feature approach: closed sessions remain viewable but not editable, session memory gets reviewed for promotion to project memory. (2) Bug approach: closed sessions should visually lock the chat, prevent new messages, show "Session Ended" state.

### [2025-12-30] Memory Context Inclusion Needs Verification
**Context:** Sprint 2 retrospective; reviewing memory system implementation.
**Lesson:** Manual memory items can be added via UI, but it's unclear if they're actually being included in LLM context. The `included_in_context` flag exists but the full flow needs verification.
**Action:** Test and verify: add manual memory item → check if it appears in Claude's context during chat completion. Document the expected behavior.

### [2025-12-30] GitHub API Should Be Primary Connection Method
**Context:** Sprint 2 execution repeatedly hit `git clone`/`git push` failures due to proxy/auth issues. GitHub API via curl always worked.
**Lesson:** The execution environment has inconsistent git CLI support, but GitHub's REST API is reliable. We documented the API fallback in LESSONS.md but it should be the PRIMARY method, not the fallback.
**Action:** Update all project instruction docs (PROJECT.md templates) to specify GitHub API via curl as the first connection method. Git CLI should be the fallback, not the other way around.

### [2025-12-30] Streaming Chat is a UX Win
**Context:** Sprint 2 completion; comparing before/after chat experience.
**Lesson:** Implementing SSE streaming for AI responses dramatically improves perceived responsiveness. Users see text appearing in real-time rather than waiting for complete responses. The placeholder → streaming → final message pattern works well.
**Action:** Keep this pattern for all future AI interactions. Consider streaming for other operations (tool execution status, file processing).

### [2025-12-30] Dogfooding Surfaces Real Problems
**Context:** Using the Planning/Execution/Review Claude workflow to build Gibbon itself.
**Lesson:** Dogfooding exposes gaps that design documents miss. The GitHub connection failure wasn't visible in docs — it only surfaced when Claude actually tried to follow the instructions. Real usage reveals real friction.
**Action:** Continue building Gibbon with Gibbon. Every friction point we hit is a friction point users will hit. Fix them as we find them.


### [2025-12-30] Modal Bootstrap Pattern: One Deploy, Unlimited Capability
**Context:** Discovery session exploring how Modal could increase Claude's throughput for building Gibbon.
**Lesson:** A single "bootstrap" function deployed to Modal gives Claude (and future Gibbon agents) full Linux execution capability. The function accepts shell or Python code via HTTP POST, runs it in a container, and returns results. Packages can be installed at runtime (e.g., `apt-get install nodejs`), eliminating the need to redeploy for new capabilities. This is the "keys to all keys" pattern - one endpoint unlocks everything.
**Action:**
1. Bootstrap endpoint deployed: `https://hsheth-ape--gibbon-bootstrap-run.modal.run`
2. All Claude roles (Planning, Execution, Review) can now verify builds, run tests, execute scripts
3. Gibbon's agents will use this same endpoint for code execution
4. Modal is serverless - costs only when running (~$0.0005 per build), nothing to manage

### [2025-12-30] Claude Already Has Local Execution
**Context:** Assumed Modal was needed for Claude to run `npm run build`. Tested and discovered Claude's environment already has bash_tool.
**Lesson:** Claude (in claude.ai) already has execution capability via bash_tool. Can download repos via tarball, run npm install, npm run build, etc. Modal is still valuable for: (1) Gibbon's agents (API-only, no bash), (2) Long-running tasks, (3) Browser automation (Playwright). But for immediate development work, Claude can self-verify builds locally.
**Action:** Use local execution for speed during development. Modal for Gibbon's agent integration.

### [2025-12-30] gRPC vs REST: Why Claude Can't Deploy Modal Functions
**Context:** Tried to deploy Modal functions from Claude's environment, failed with "Could not connect to Modal server."
**Lesson:** Modal CLI uses gRPC protocol, which is blocked in Claude's network environment. Regular HTTPS (REST APIs) works fine - GitHub, Figma, Supabase, and calling deployed Modal endpoints all work. This is a network proxy limitation, not a credentials issue.
**Action:** User deploys Modal functions once from their local machine. Claude calls the deployed endpoints via HTTP forever after.


### [2025-12-30] Design-First Approach: UX Before Implementation
**Context:** Replanning sprints after realizing Execution Claude was building UI without design reference.
**Lesson:** Building UI without designs leads to inconsistency and rework. The UX Agent should create designs first, then Execution Claude implements to match. This gives a clear visual target and enables design system consistency.
**Action:** 
1. Reprioritized sprints: UX Agent (Sprint 3) before GitHub+Modal tools (Sprint 4)
2. Created GIBBON_UI_SPEC.md as first design task
3. Execution Claude must reference designs before building any UI

### [2025-12-30] Role Guardrails Prevent Scope Creep
**Context:** Planning sessions sometimes drifted into "let me just code this quick fix" territory.
**Lesson:** Without explicit boundaries, it's easy to violate role separation. Adding documented trigger phrases ("code this...", "build me...") with proper redirects helps catch scope creep early.
**Action:**
1. Added guardrails section to PLANNING.md template
2. Documented trigger phrases → redirect responses
3. Self-check questions before substantive responses

### [2025-12-30] Merged Planning + Review Improves Continuity
**Context:** Separate Planning and Review Claude roles meant context was lost between sessions.
**Lesson:** Strategic thinking (planning) and outcome analysis (review) are two sides of the same coin. Merging them into one role maintains the strategic thread and reduces handoff overhead.
**Action:**
1. Merged Planning + Review into single role
2. Updated project-templates/PLANNING.md
3. Review mode activates based on context (after execution, "what happened")




### [2025-12-30] Design-First Means Specs, Not Always UI
**Context:** Discussing whether workflow/headless design needs a separate agent like UX Agent.
**Lesson:** Planning outputs two spec types: visual (→ UX Agent via Penpot) and workflow (→ Execution directly via Mermaid). "Design-first" doesn't mean everything goes through a design tool. Mermaid diagrams are Claude-native, version-controlled, and sufficient for workflows. Penpot is for UI mockups and persistent architectural artifacts only.
**Action:** Use Mermaid for execution plans. Reserve Penpot/UX Agent for actual UI work. No separate "Workflow Agent" needed — that's Planning Claude's job.

### [2025-12-30] Branch Protection Prevents Sprint Failures
**Context:** Sprint 2 failed because broken code went straight to main. Analyzing how to prevent recurrence.
**Lesson:** Without branches, every commit risks breaking main. With branches + PR review, Execution Claude can experiment safely. Planning Claude reviews diffs before merge. Human has final approval. Main stays clean, rollback is easy.
**Action:** All code changes go to branches. PR required. Planning reviews, human approves. Docs can go to main directly (low risk).

## Categories for Reference
- **Process:** How we work
- **Technical:** Code and architecture
- **Tools:** Services and integrations
- **Communication:** Working together

### [2025-12-30] Planning Hierarchy Emerged from Dogfooding
**Context:** Attempting to replan Sprint 3 after execution failure. Realized we were jumping straight to tasks without clear goals or understanding of what's possible.
**Lesson:** Dogfooding surfaces process gaps that design docs miss. We discovered we needed a planning hierarchy (Discovery → Inventory → Strategy → Sprint Goals → Tasks) by hitting the wall ourselves. Discovery isn't just "what to build" — it's "what's possible, what knowledge exists, what could unlock our approach."
**Action:** 
1. Added Planning Hierarchy to WAYS_OF_WORKING.md
2. Created toolbox/PLANNING.md for detailed guidance
3. Updated project-templates repo so future projects inherit the framework
4. Continue dogfooding — every friction point we hit is a friction point users will hit



### [2025-12-30] First Successful Tool Execution in Chat
**Context:** Testing Sprint 3.5 tool loop after deployment. Lead Agent was creating task descriptions instead of calling tools.
**Lesson:** Two bugs blocked tool execution: (1) Frontend wasn't sending `use_tools: true` to the API, (2) ChatPanel expected SSE streaming but tool loop returns JSON. The database config was correct all along - the bugs were in the frontend code.
**Action:** 
1. Always verify the full request/response flow when tools don't work
2. Check what flags are being passed to the API
3. Test with different response formats (SSE vs JSON)

**Milestone:** First successful tool execution - Lead Agent read docs/gibbon/CURRENT.md via github_read_file tool! 🎉

