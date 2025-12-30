# Sprint 4: Orchestration Foundation

> **Goal:** Intelligent routing, focused context, learning infrastructure
> **Branch:** `sprint-4/orchestration`
> **Estimate:** ~28.5 hours (9 phases)
> **Status:** 🟡 Ready for Execution

---

## Overview

This sprint builds the orchestration layer that makes Gibbon intelligent:

```
User Message → Orchestrator → Curated Context → Right Agent → Response
                    ↓
              Training Log (for future local LLM)
```

**Key Outcomes:**
1. Messages routed to correct agent automatically
2. Agents receive only relevant context (not everything)
3. Users can send multiple messages without blocking
4. Every interaction logged for future model training
5. Lead Agent can create/manage other agents

---

## Prerequisites

Before starting:
- [ ] Sprint 3.5 merged (v0.5.0) ✅
- [ ] GitHub tools working ✅
- [ ] Modal endpoint deployed ✅

---

## Phase 1: Fix Existing Bugs (2h)

### 1.1 Add Close Session Button

**File:** `src/components/ChatPanel.tsx`

Find the Session Header section and add a close button:

```typescript
{/* Session Header */}
{activeSession && (
  <div className="px-4 py-3 border-b border-zinc-800 flex justify-between items-start">
    <div>
      <h3 className="text-white font-medium">{activeSession.goal}</h3>
      <div className="text-xs text-zinc-500 mt-1">
        Started {new Date(activeSession.created_at).toLocaleDateString()}
      </div>
    </div>
    {activeSession.status === 'active' && (
      <button
        onClick={handleCloseSession}
        className="text-xs px-2 py-1 text-zinc-400 hover:text-white hover:bg-zinc-800 rounded"
      >
        Close Session
      </button>
    )}
    {activeSession.status === 'closed' && (
      <span className="text-xs px-2 py-1 text-zinc-500 bg-zinc-800 rounded">
        Closed
      </span>
    )}
  </div>
)}
```

### 1.2 Add Session Outcome Prompt

Update `handleCloseSession` in `ChatPanel.tsx`:

```typescript
const handleCloseSession = async () => {
  if (!activeSession || activeSession.status === 'closed') return

  // Ask for outcome
  const goalAchieved = confirm('Did you accomplish your session goal?')
  const feedback = prompt('Any feedback on this session? (optional)')
  
  const res = await fetch(`/api/sessions/${activeSession.id}`, {
    method: 'PATCH',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ 
      status: 'closed',
    }),
  })

  if (res.ok) {
    // Save session outcome
    await fetch(`/api/sessions/${activeSession.id}/outcome`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        goal_achieved: goalAchieved ? 'yes' : 'no',
        user_feedback: feedback || null,
      }),
    })
    
    // Reload to reflect changes
    window.location.reload()
  }
}
```

### 1.3 Create Session Outcomes Table

**File:** `supabase/migrations/20251230_session_outcomes.sql`

```sql
-- Session outcomes for training data
CREATE TABLE session_outcomes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id UUID NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
  
  -- User feedback
  goal_achieved TEXT NOT NULL,  -- 'yes', 'partial', 'no'
  user_feedback TEXT,
  
  -- Computed metrics (updated async)
  total_messages INTEGER,
  total_corrections INTEGER,
  avg_response_score FLOAT,
  duration_minutes INTEGER,
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  
  UNIQUE(session_id)
);

-- Index for analysis
CREATE INDEX idx_session_outcomes_achieved ON session_outcomes(goal_achieved);
```

### 1.4 Create Session Outcome API

**File:** `src/app/api/sessions/[id]/outcome/route.ts`

```typescript
import { createClient } from '@/lib/supabase/server'
import { NextResponse } from 'next/server'

export async function POST(
  request: Request,
  { params }: { params: { id: string } }
) {
  const supabase = await createClient()
  const { data: { user } } = await supabase.auth.getUser()

  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const sessionId = params.id
  const body = await request.json()
  const { goal_achieved, user_feedback } = body

  // Verify session exists and user has access
  const { data: session } = await supabase
    .from('sessions')
    .select('id, project_id')
    .eq('id', sessionId)
    .single()

  if (!session) {
    return NextResponse.json({ error: 'Session not found' }, { status: 404 })
  }

  // Count messages for metrics
  const { count: messageCount } = await supabase
    .from('messages')
    .select('id', { count: 'exact' })
    .eq('session_id', sessionId)

  // Insert outcome
  const { data: outcome, error } = await supabase
    .from('session_outcomes')
    .upsert({
      session_id: sessionId,
      goal_achieved,
      user_feedback,
      total_messages: messageCount || 0,
    })
    .select()
    .single()

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 })
  }

  return NextResponse.json({ outcome })
}
```

**Commit:** `fix: add session close button and outcome tracking`

**Build checkpoint:** `npm run build` → Session close works

---

## Phase 2: Project File Index (4h)

### 2.1 Create Project Index Table

**File:** `supabase/migrations/20251230_project_index.sql`

```sql
-- Project file index for orchestrator context
CREATE TABLE project_index (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  
  -- File info
  file_path TEXT NOT NULL,
  summary TEXT NOT NULL,           -- One-line LLM summary
  file_type TEXT,                  -- 'component', 'hook', 'api', 'config', 'util', 'test'
  size_bytes INTEGER,
  
  -- Timestamps
  last_indexed TIMESTAMPTZ DEFAULT NOW(),
  file_modified TIMESTAMPTZ,
  
  UNIQUE(project_id, file_path)
);

-- Index for lookups
CREATE INDEX idx_project_index_project ON project_index(project_id);
CREATE INDEX idx_project_index_type ON project_index(project_id, file_type);
```

### 2.2 File Indexer Service

**File:** `src/lib/indexer/service.ts`

```typescript
import Anthropic from '@anthropic-ai/sdk'
import { getGitHubConfig } from '@/lib/github/config'
import { createGitHubClient } from '@/lib/github/client'

const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
})

interface FileIndex {
  path: string
  summary: string
  fileType: string
  sizeBytes: number
}

const FILE_TYPE_MAP: Record<string, string> = {
  'components/': 'component',
  'hooks/': 'hook',
  'app/api/': 'api',
  'lib/': 'util',
  'test': 'test',
  '.config': 'config',
  'config': 'config',
}

function inferFileType(path: string): string {
  for (const [pattern, type] of Object.entries(FILE_TYPE_MAP)) {
    if (path.includes(pattern)) return type
  }
  if (path.endsWith('.tsx') || path.endsWith('.jsx')) return 'component'
  if (path.endsWith('.ts') || path.endsWith('.js')) return 'util'
  if (path.endsWith('.md')) return 'doc'
  if (path.endsWith('.json')) return 'config'
  return 'other'
}

async function summarizeFile(path: string, content: string): Promise<string> {
  // Skip very large files
  if (content.length > 50000) {
    return `Large file (${Math.round(content.length / 1000)}KB) - ${path.split('/').pop()}`
  }

  // Use Haiku for cheap summarization
  const response = await anthropic.messages.create({
    model: 'claude-3-haiku-20240307',
    max_tokens: 100,
    messages: [{
      role: 'user',
      content: `Summarize this file in ONE short sentence (max 15 words). Focus on what it does, not implementation details.

File: ${path}

\`\`\`
${content.slice(0, 4000)}
\`\`\`

One-sentence summary:`
    }]
  })

  const text = response.content[0]
  if (text.type === 'text') {
    return text.text.trim().replace(/^["']|["']$/g, '')
  }
  return `File: ${path.split('/').pop()}`
}

export async function indexProject(projectId: string): Promise<FileIndex[]> {
  const config = await getGitHubConfig(projectId)
  if (!config) {
    throw new Error('GitHub not configured for project')
  }

  const client = createGitHubClient(config)
  const files: FileIndex[] = []

  // Get all files recursively
  async function indexDirectory(path: string = '') {
    const items = await client.listFiles(path)
    
    for (const item of items) {
      // Skip non-code files
      if (item.type === 'dir') {
        // Skip node_modules, .git, etc.
        if (['node_modules', '.git', '.next', 'dist', 'build'].includes(item.name)) {
          continue
        }
        await indexDirectory(item.path)
      } else if (item.type === 'file') {
        // Only index code files
        const ext = item.name.split('.').pop()?.toLowerCase()
        if (!['ts', 'tsx', 'js', 'jsx', 'md', 'json', 'sql', 'css'].includes(ext || '')) {
          continue
        }

        try {
          const content = await client.readFile(item.path)
          const summary = await summarizeFile(item.path, content)
          
          files.push({
            path: item.path,
            summary,
            fileType: inferFileType(item.path),
            sizeBytes: content.length,
          })
        } catch (error) {
          console.error(`Failed to index ${item.path}:`, error)
        }
      }
    }
  }

  await indexDirectory()
  return files
}

export async function saveIndex(
  supabase: any,
  projectId: string,
  files: FileIndex[]
): Promise<void> {
  // Delete old index
  await supabase
    .from('project_index')
    .delete()
    .eq('project_id', projectId)

  // Insert new index
  const rows = files.map(f => ({
    project_id: projectId,
    file_path: f.path,
    summary: f.summary,
    file_type: f.fileType,
    size_bytes: f.sizeBytes,
    last_indexed: new Date().toISOString(),
  }))

  // Batch insert
  const batchSize = 50
  for (let i = 0; i < rows.length; i += batchSize) {
    const batch = rows.slice(i, i + batchSize)
    await supabase.from('project_index').insert(batch)
  }
}
```

### 2.3 Index API Endpoint

**File:** `src/app/api/projects/[id]/index/route.ts`

```typescript
import { createClient } from '@/lib/supabase/server'
import { NextResponse } from 'next/server'
import { indexProject, saveIndex } from '@/lib/indexer/service'

// GET - retrieve current index
export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  const supabase = await createClient()
  const projectId = params.id

  const { data: index, error } = await supabase
    .from('project_index')
    .select('file_path, summary, file_type, size_bytes, last_indexed')
    .eq('project_id', projectId)
    .order('file_path')

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 })
  }

  return NextResponse.json({ index, count: index?.length || 0 })
}

// POST - trigger re-index
export async function POST(
  request: Request,
  { params }: { params: { id: string } }
) {
  const supabase = await createClient()
  const { data: { user } } = await supabase.auth.getUser()

  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const projectId = params.id

  try {
    // Index the project
    const files = await indexProject(projectId)
    
    // Save to database
    await saveIndex(supabase, projectId, files)

    return NextResponse.json({ 
      success: true, 
      indexed: files.length,
      files: files.map(f => ({ path: f.path, summary: f.summary }))
    })
  } catch (error) {
    console.error('Index error:', error)
    return NextResponse.json({ 
      error: String(error) 
    }, { status: 500 })
  }
}
```

### 2.4 Add Index Types

**File:** `src/lib/types.ts` (add to existing)

```typescript
// Project index types
export interface ProjectFileIndex {
  id: string
  project_id: string
  file_path: string
  summary: string
  file_type: string
  size_bytes: number
  last_indexed: string
}
```

**Commit:** `feat(index): project file indexer with LLM summaries`

**Build checkpoint:** `npm run build` → Index API works

---

## Phase 3: Orchestrator Core (5h)

### 3.1 Orchestrator Types

**File:** `src/lib/orchestrator/types.ts`

```typescript
export interface OrchestrationInput {
  message: string
  sessionId: string
  projectId: string
  sessionGoal: string
  fileIndex: { path: string; summary: string }[]
  memory: { content: string; type: string }[]
  agents: { slug: string; name: string; description: string }[]
  recentMessages: { role: string; content: string }[]
}

export interface OrchestrationOutput {
  targetAgent: string
  refinedPrompt: string
  selectedFiles: string[]      // Paths to include
  selectedMemory: string[]     // Memory item IDs
  conversationWindow: number   // How many recent messages
  reasoning: string            // Why this decision (for logging)
}

export interface OrchestrationContext {
  files: Record<string, string>  // path -> content
  memory: { content: string; type: string }[]
  messages: { role: string; content: string }[]
}
```

### 3.2 Orchestrator Prompt

**File:** `src/lib/orchestrator/prompt.ts`

```typescript
export const ORCHESTRATOR_SYSTEM_PROMPT = `You are an orchestration agent for Gibbon, a Product Factory OS. Your job is to route user messages to the right agent with the right context.

## Your Task

Given a user message and project context, determine:
1. Which agent should handle this request
2. What files are relevant (select from the index)
3. What memory items are relevant
4. How to refine the prompt for clarity
5. How much conversation history is needed

## Available Agents

- **lead**: General assistant, planning, discussion, coordination. Use for questions, planning, unclear requests.
- **execution**: Code changes, file modifications, running builds. Use for "fix", "build", "implement", "create", "update code".
- **review**: Analyzing work, checking quality, retrospectives. Use for "review", "check", "analyze", "what happened".

## Rules

1. Default to "lead" if unclear
2. Select only files that are directly relevant (usually 1-5 files)
3. Include memory items that provide essential context
4. Refine vague prompts to be specific and actionable
5. For code tasks, always include related files (if user mentions "login", include auth-related files)

## Output Format

Respond with JSON only:
{
  "targetAgent": "lead" | "execution" | "review",
  "refinedPrompt": "Clear, specific version of the user's request",
  "selectedFiles": ["path/to/file1.ts", "path/to/file2.ts"],
  "selectedMemory": ["memory item content to include"],
  "conversationWindow": 5,
  "reasoning": "Brief explanation of your routing decision"
}
`

export function buildOrchestratorUserPrompt(input: {
  message: string
  sessionGoal: string
  fileIndex: { path: string; summary: string }[]
  memory: { content: string; type: string }[]
  agents: { slug: string; name: string }[]
  recentSummary: string
}): string {
  return `## User Message
"${input.message}"

## Session Goal
${input.sessionGoal}

## Available Files
${input.fileIndex.map(f => `- ${f.path}: ${f.summary}`).join('\n')}

## Memory Items
${input.memory.map(m => `- [${m.type}] ${m.content}`).join('\n') || 'No memory items'}

## Recent Conversation Summary
${input.recentSummary || 'No prior messages in this session'}

## Your Decision (JSON only):`
}
```

### 3.3 Orchestrator Tools

**File:** `src/lib/orchestrator/tools.ts`

```typescript
import { createClient } from '@/lib/supabase/server'
import { getGitHubConfig } from '@/lib/github/config'
import { createGitHubClient } from '@/lib/github/client'

export interface OrchestratorToolContext {
  projectId: string
  sessionId: string
}

// Tool to read actual file content (not just summary)
export async function readFileContent(
  path: string, 
  context: OrchestratorToolContext
): Promise<string> {
  const config = await getGitHubConfig(context.projectId)
  if (!config) {
    return 'Error: GitHub not configured'
  }
  
  const client = createGitHubClient(config)
  try {
    return await client.readFile(path)
  } catch (error) {
    return `Error reading file: ${error}`
  }
}

// Tool to get memory items
export async function getMemoryItems(
  scope: 'session' | 'project',
  context: OrchestratorToolContext
): Promise<{ content: string; type: string }[]> {
  const supabase = await createClient()
  
  const scopeId = scope === 'session' ? context.sessionId : context.projectId
  
  const { data } = await supabase
    .from('memory_items')
    .select('content, item_type')
    .eq('scope', scope)
    .eq('scope_id', scopeId)
    .eq('active', true)

  return data?.map(m => ({ content: m.content, type: m.item_type })) || []
}

// Tool to get recent messages
export async function getRecentMessages(
  count: number,
  context: OrchestratorToolContext
): Promise<{ role: string; content: string }[]> {
  const supabase = await createClient()
  
  const { data } = await supabase
    .from('messages')
    .select('sender_type, content')
    .eq('session_id', context.sessionId)
    .order('created_at', { ascending: false })
    .limit(count)

  return data?.reverse().map(m => ({
    role: m.sender_type === 'user' ? 'user' : 'assistant',
    content: m.content
  })) || []
}

// Tool to list available agents
export async function listAgents(): Promise<{ slug: string; name: string; description: string }[]> {
  const supabase = await createClient()
  
  const { data } = await supabase
    .from('agents')
    .select('slug, name, role')
    .eq('is_system', true)

  return data?.map(a => ({
    slug: a.slug,
    name: a.name,
    description: a.role || ''
  })) || []
}
```

### 3.4 Orchestrator Service

**File:** `src/lib/orchestrator/service.ts`

```typescript
import Anthropic from '@anthropic-ai/sdk'
import { createClient } from '@/lib/supabase/server'
import { 
  ORCHESTRATOR_SYSTEM_PROMPT, 
  buildOrchestratorUserPrompt 
} from './prompt'
import { 
  readFileContent, 
  getMemoryItems, 
  getRecentMessages,
  listAgents,
  OrchestratorToolContext 
} from './tools'
import type { 
  OrchestrationInput, 
  OrchestrationOutput, 
  OrchestrationContext 
} from './types'

const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
})

export async function orchestrate(
  message: string,
  sessionId: string,
  projectId: string
): Promise<{ decision: OrchestrationOutput; context: OrchestrationContext }> {
  const supabase = await createClient()
  const toolContext: OrchestratorToolContext = { projectId, sessionId }

  // Gather initial context
  const [
    sessionResult,
    indexResult,
    projectMemory,
    sessionMemory,
    recentMessages,
    agents
  ] = await Promise.all([
    supabase.from('sessions').select('goal').eq('id', sessionId).single(),
    supabase.from('project_index').select('file_path, summary').eq('project_id', projectId),
    getMemoryItems('project', toolContext),
    getMemoryItems('session', toolContext),
    getRecentMessages(10, toolContext),
    listAgents()
  ])

  const sessionGoal = sessionResult.data?.goal || 'General assistance'
  const fileIndex = indexResult.data?.map(f => ({ 
    path: f.file_path, 
    summary: f.summary 
  })) || []
  const allMemory = [...projectMemory, ...sessionMemory]

  // Summarize recent conversation
  const recentSummary = recentMessages.length > 0
    ? recentMessages.slice(-5).map(m => `${m.role}: ${m.content.slice(0, 100)}...`).join('\n')
    : ''

  // Build prompt for orchestrator
  const userPrompt = buildOrchestratorUserPrompt({
    message,
    sessionGoal,
    fileIndex,
    memory: allMemory,
    agents,
    recentSummary
  })

  // Call Haiku for orchestration (cheap & fast)
  const response = await anthropic.messages.create({
    model: 'claude-3-haiku-20240307',
    max_tokens: 1000,
    system: ORCHESTRATOR_SYSTEM_PROMPT,
    messages: [{ role: 'user', content: userPrompt }]
  })

  // Parse response
  const responseText = response.content[0]
  if (responseText.type !== 'text') {
    throw new Error('Unexpected response type')
  }

  let decision: OrchestrationOutput
  try {
    // Extract JSON from response (handle markdown code blocks)
    let jsonStr = responseText.text
    const jsonMatch = jsonStr.match(/```(?:json)?\s*([\s\S]*?)```/)
    if (jsonMatch) {
      jsonStr = jsonMatch[1]
    }
    decision = JSON.parse(jsonStr.trim())
  } catch (error) {
    // Default to lead agent if parsing fails
    console.error('Failed to parse orchestrator response:', responseText.text)
    decision = {
      targetAgent: 'lead',
      refinedPrompt: message,
      selectedFiles: [],
      selectedMemory: [],
      conversationWindow: 10,
      reasoning: 'Fallback due to parse error'
    }
  }

  // Fetch actual content for selected files
  const fileContents: Record<string, string> = {}
  for (const path of decision.selectedFiles.slice(0, 5)) { // Limit to 5 files
    const content = await readFileContent(path, toolContext)
    fileContents[path] = content
  }

  // Filter memory to selected items
  const selectedMemoryItems = allMemory.filter(m => 
    decision.selectedMemory.some(selected => 
      m.content.toLowerCase().includes(selected.toLowerCase()) ||
      selected.toLowerCase().includes(m.content.toLowerCase())
    )
  )

  // Get conversation window
  const conversationSlice = recentMessages.slice(-decision.conversationWindow)

  return {
    decision,
    context: {
      files: fileContents,
      memory: selectedMemoryItems.length > 0 ? selectedMemoryItems : allMemory.slice(0, 3),
      messages: conversationSlice
    }
  }
}
```

**Commit:** `feat(orchestrator): intelligent message routing with Haiku`

**Build checkpoint:** `npm run build` → Orchestrator compiles

---

## Phase 4: Training Infrastructure (3h)

### 4.1 Create Orchestration Logs Table

**File:** `supabase/migrations/20251230_orchestration_logs.sql`

```sql
-- Orchestration decisions for training data
CREATE TABLE orchestration_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  
  -- References
  message_id UUID REFERENCES messages(id) ON DELETE SET NULL,
  session_id UUID NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  
  -- Input (what orchestrator saw)
  input_message TEXT NOT NULL,
  input_file_index JSONB NOT NULL,
  input_memory JSONB NOT NULL,
  input_context JSONB,
  
  -- Decision (what orchestrator decided)
  selected_agent TEXT NOT NULL,
  refined_prompt TEXT,
  selected_files JSONB NOT NULL,
  selected_memory JSONB NOT NULL,
  conversation_window INTEGER,
  reasoning TEXT,
  
  -- Outcome (updated after agent responds)
  agent_completed BOOLEAN,
  agent_tools_used JSONB,
  agent_error TEXT,
  
  -- User feedback (explicit)
  user_thumbs TEXT,  -- 'up', 'down', null
  user_comment TEXT,
  
  -- Implicit feedback (computed)
  follow_up_type TEXT,  -- 'new_topic', 'correction', 'continuation'
  
  -- Quality score (computed by evaluator)
  quality_score FLOAT,
  quality_reasoning TEXT,
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  scored_at TIMESTAMPTZ
);

-- Indexes for analysis
CREATE INDEX idx_orch_logs_session ON orchestration_logs(session_id);
CREATE INDEX idx_orch_logs_agent ON orchestration_logs(selected_agent);
CREATE INDEX idx_orch_logs_score ON orchestration_logs(quality_score);
CREATE INDEX idx_orch_logs_thumbs ON orchestration_logs(user_thumbs);
```

### 4.2 Logging Service

**File:** `src/lib/orchestrator/logging.ts`

```typescript
import { createClient } from '@/lib/supabase/server'
import type { OrchestrationOutput } from './types'

export interface LogInput {
  sessionId: string
  projectId: string
  messageId?: string
  inputMessage: string
  fileIndex: { path: string; summary: string }[]
  memory: { content: string; type: string }[]
  context?: Record<string, unknown>
}

export async function logOrchestration(
  input: LogInput,
  decision: OrchestrationOutput
): Promise<string> {
  const supabase = await createClient()

  const { data, error } = await supabase
    .from('orchestration_logs')
    .insert({
      session_id: input.sessionId,
      project_id: input.projectId,
      message_id: input.messageId,
      input_message: input.inputMessage,
      input_file_index: input.fileIndex,
      input_memory: input.memory,
      input_context: input.context,
      selected_agent: decision.targetAgent,
      refined_prompt: decision.refinedPrompt,
      selected_files: decision.selectedFiles,
      selected_memory: decision.selectedMemory,
      conversation_window: decision.conversationWindow,
      reasoning: decision.reasoning,
    })
    .select('id')
    .single()

  if (error) {
    console.error('Failed to log orchestration:', error)
    return ''
  }

  return data.id
}

export async function updateOrchestrationOutcome(
  logId: string,
  outcome: {
    completed: boolean
    toolsUsed?: string[]
    error?: string
  }
): Promise<void> {
  const supabase = await createClient()

  await supabase
    .from('orchestration_logs')
    .update({
      agent_completed: outcome.completed,
      agent_tools_used: outcome.toolsUsed,
      agent_error: outcome.error,
    })
    .eq('id', logId)
}

export async function updateOrchestrationFeedback(
  logId: string,
  feedback: {
    thumbs?: 'up' | 'down'
    comment?: string
    followUpType?: string
  }
): Promise<void> {
  const supabase = await createClient()

  await supabase
    .from('orchestration_logs')
    .update({
      user_thumbs: feedback.thumbs,
      user_comment: feedback.comment,
      follow_up_type: feedback.followUpType,
    })
    .eq('id', logId)
}
```

**Commit:** `feat(training): orchestration logging infrastructure`

**Build checkpoint:** `npm run build` → Logging compiles

---

## Phase 5: Feedback UI (2.5h)

### 5.1 Feedback Component

**File:** `src/components/chat/Feedback.tsx`

```typescript
'use client'

import { useState } from 'react'

interface FeedbackProps {
  messageId: string
  onFeedback?: (thumbs: 'up' | 'down') => void
}

export function Feedback({ messageId, onFeedback }: FeedbackProps) {
  const [submitted, setSubmitted] = useState<'up' | 'down' | null>(null)
  const [loading, setLoading] = useState(false)

  const handleFeedback = async (thumbs: 'up' | 'down') => {
    if (loading || submitted) return
    
    setLoading(true)
    try {
      await fetch(`/api/messages/${messageId}/feedback`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ thumbs }),
      })
      setSubmitted(thumbs)
      onFeedback?.(thumbs)
    } catch (error) {
      console.error('Feedback error:', error)
    } finally {
      setLoading(false)
    }
  }

  return (
    <div className="flex gap-1 mt-1">
      <button
        onClick={() => handleFeedback('up')}
        disabled={loading || submitted !== null}
        className={`p-1 rounded text-sm transition-colors ${
          submitted === 'up'
            ? 'text-green-400 bg-green-400/10'
            : 'text-zinc-500 hover:text-zinc-300 hover:bg-zinc-800'
        } disabled:opacity-50`}
        title="Good response"
      >
        👍
      </button>
      <button
        onClick={() => handleFeedback('down')}
        disabled={loading || submitted !== null}
        className={`p-1 rounded text-sm transition-colors ${
          submitted === 'down'
            ? 'text-red-400 bg-red-400/10'
            : 'text-zinc-500 hover:text-zinc-300 hover:bg-zinc-800'
        } disabled:opacity-50`}
        title="Poor response"
      >
        👎
      </button>
    </div>
  )
}
```

### 5.2 Update MessageList

**File:** `src/components/MessageList.tsx`

Add import and render Feedback for agent messages:

```typescript
import { Feedback } from './chat/Feedback'

// In the message render, after the message content for agent messages:
{msg.sender_type === 'agent' && (
  <Feedback messageId={msg.id} />
)}
```

### 5.3 Feedback API

**File:** `src/app/api/messages/[id]/feedback/route.ts`

```typescript
import { createClient } from '@/lib/supabase/server'
import { NextResponse } from 'next/server'
import { updateOrchestrationFeedback } from '@/lib/orchestrator/logging'

export async function POST(
  request: Request,
  { params }: { params: { id: string } }
) {
  const supabase = await createClient()
  const { data: { user } } = await supabase.auth.getUser()

  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const messageId = params.id
  const body = await request.json()
  const { thumbs, comment } = body

  // Find the orchestration log for this message
  const { data: log } = await supabase
    .from('orchestration_logs')
    .select('id')
    .eq('message_id', messageId)
    .single()

  if (log) {
    await updateOrchestrationFeedback(log.id, { thumbs, comment })
  }

  // Also store on message metadata for quick access
  await supabase
    .from('messages')
    .update({
      metadata: supabase.sql`COALESCE(metadata, '{}'::jsonb) || '{"feedback": "${thumbs}"}'::jsonb`
    })
    .eq('id', messageId)

  return NextResponse.json({ success: true })
}
```

**Commit:** `feat(feedback): thumbs up/down on agent responses`

**Build checkpoint:** `npm run build` → Feedback UI works

---

## Phase 6: Wire Orchestrator to Chat (3h)

### 6.1 Update Chat Completion Route

**File:** `src/app/api/chat/complete/route.ts`

Replace the existing route with orchestrator integration:

```typescript
import { createClient } from '@/lib/supabase/server'
import Anthropic from '@anthropic-ai/sdk'
import { runToolLoop } from '@/lib/chat/tool-loop'
import { getToolDefinitions } from '@/lib/tools/executor'
import { orchestrate } from '@/lib/orchestrator/service'
import { logOrchestration, updateOrchestrationOutcome } from '@/lib/orchestrator/logging'

const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
})

export async function POST(request: Request) {
  const supabase = await createClient()
  const { data: { user } } = await supabase.auth.getUser()

  if (!user) {
    return new Response(JSON.stringify({ error: 'Unauthorized' }), { 
      status: 401,
      headers: { 'Content-Type': 'application/json' }
    })
  }

  const body = await request.json()
  const { session_id, use_tools = true } = body

  if (!session_id) {
    return new Response(JSON.stringify({ error: 'session_id is required' }), { 
      status: 400,
      headers: { 'Content-Type': 'application/json' }
    })
  }

  // Get session and project
  const { data: session } = await supabase
    .from('sessions')
    .select('id, goal, status, project_id')
    .eq('id', session_id)
    .single()

  if (!session || session.status === 'closed') {
    return new Response(JSON.stringify({ error: 'Session not found or closed' }), { 
      status: 404,
      headers: { 'Content-Type': 'application/json' }
    })
  }

  // Get the latest user message
  const { data: messages } = await supabase
    .from('messages')
    .select('id, sender_type, content')
    .eq('session_id', session_id)
    .order('created_at', { ascending: false })
    .limit(1)

  const latestMessage = messages?.[0]
  if (!latestMessage || latestMessage.sender_type !== 'user') {
    return new Response(JSON.stringify({ error: 'No user message to respond to' }), { 
      status: 400,
      headers: { 'Content-Type': 'application/json' }
    })
  }

  // === ORCHESTRATE ===
  const { decision, context } = await orchestrate(
    latestMessage.content,
    session_id,
    session.project_id
  )

  // Log orchestration decision
  const { data: fileIndex } = await supabase
    .from('project_index')
    .select('file_path, summary')
    .eq('project_id', session.project_id)

  const logId = await logOrchestration(
    {
      sessionId: session_id,
      projectId: session.project_id,
      messageId: latestMessage.id,
      inputMessage: latestMessage.content,
      fileIndex: fileIndex?.map(f => ({ path: f.file_path, summary: f.summary })) || [],
      memory: context.memory,
    },
    decision
  )

  // Get target agent
  const { data: agent } = await supabase
    .from('agents')
    .select('id, name, system_prompt, model_config')
    .eq('slug', decision.targetAgent)
    .eq('is_system', true)
    .single()

  // Fallback to lead if agent not found
  const targetAgent = agent || await supabase
    .from('agents')
    .select('id, name, system_prompt, model_config')
    .eq('slug', 'lead')
    .eq('is_system', true)
    .single()
    .then(r => r.data)

  if (!targetAgent) {
    return new Response(JSON.stringify({ error: 'No agent available' }), { 
      status: 500,
      headers: { 'Content-Type': 'application/json' }
    })
  }

  // Build system prompt with curated context
  let systemPrompt = targetAgent.system_prompt || ''
  
  systemPrompt += `\n\n--- Current Context ---`
  systemPrompt += `\nSession Goal: ${session.goal}`
  
  if (context.memory.length > 0) {
    systemPrompt += `\n\n--- Relevant Memory ---`
    for (const item of context.memory) {
      systemPrompt += `\n- [${item.type}] ${item.content}`
    }
  }

  if (Object.keys(context.files).length > 0) {
    systemPrompt += `\n\n--- Relevant Files ---`
    for (const [path, content] of Object.entries(context.files)) {
      systemPrompt += `\n\n### ${path}\n\`\`\`\n${content.slice(0, 3000)}\n\`\`\``
    }
  }

  if (use_tools) {
    const tools = getToolDefinitions()
    systemPrompt += `\n\n--- Available Tools ---`
    for (const tool of tools) {
      systemPrompt += `\n- ${tool.name}: ${tool.description}`
    }
  }

  // Build messages array
  const claudeMessages: Anthropic.MessageParam[] = context.messages.map(m => ({
    role: m.role as 'user' | 'assistant',
    content: m.content
  }))

  // Use refined prompt if different from original
  if (decision.refinedPrompt && decision.refinedPrompt !== latestMessage.content) {
    // Replace last user message with refined version
    claudeMessages[claudeMessages.length - 1] = {
      role: 'user',
      content: decision.refinedPrompt
    }
  }

  // Execute with tool loop
  const toolContext = {
    projectId: session.project_id,
    sessionId: session_id,
    userId: user.id,
  }

  try {
    const result = await runToolLoop(anthropic, claudeMessages, systemPrompt, {
      context: toolContext,
      onToolCall: (name, input) => {
        console.log(`[${decision.targetAgent}] Tool: ${name}`)
      },
    })

    // Update orchestration outcome
    await updateOrchestrationOutcome(logId, {
      completed: true,
      toolsUsed: result.toolCalls.map(t => t.name),
    })

    // Save response
    const { data: savedMessage } = await supabase
      .from('messages')
      .insert({
        session_id,
        sender_type: 'agent',
        sender_id: targetAgent.id,
        content: result.finalText,
        message_type: 'text',
        metadata: {
          tool_calls: result.toolCalls,
          orchestration_log_id: logId,
          agent_slug: decision.targetAgent,
        },
      })
      .select()
      .single()

    // Update log with message ID
    await supabase
      .from('orchestration_logs')
      .update({ message_id: savedMessage?.id })
      .eq('id', logId)

    return new Response(JSON.stringify({
      message: {
        ...savedMessage,
        agent: {
          id: targetAgent.id,
          name: targetAgent.name,
          agent_type: decision.targetAgent,
        },
      },
      orchestration: {
        agent: decision.targetAgent,
        reasoning: decision.reasoning,
        files_used: decision.selectedFiles,
      },
    }), {
      headers: { 'Content-Type': 'application/json' }
    })

  } catch (error) {
    await updateOrchestrationOutcome(logId, {
      completed: false,
      error: String(error),
    })
    
    return new Response(JSON.stringify({ error: 'Agent failed' }), { 
      status: 500,
      headers: { 'Content-Type': 'application/json' }
    })
  }
}
```

**Commit:** `feat(chat): integrate orchestrator into chat completion`

**Build checkpoint:** `npm run build` → Full flow works

---

## Phase 7: Message Queue (4h)

### 7.1 Add Queue Fields to Messages

**File:** `supabase/migrations/20251230_message_queue.sql`

```sql
-- Add queue fields to messages
ALTER TABLE messages 
ADD COLUMN queue_status TEXT DEFAULT 'done',  -- pending, processing, done, failed
ADD COLUMN queued_at TIMESTAMPTZ DEFAULT NOW(),
ADD COLUMN started_at TIMESTAMPTZ,
ADD COLUMN completed_at TIMESTAMPTZ,
ADD COLUMN depends_on UUID[];

-- Index for queue processing
CREATE INDEX idx_messages_queue ON messages(session_id, queue_status, queued_at);

-- For new messages, default to pending
ALTER TABLE messages ALTER COLUMN queue_status SET DEFAULT 'pending';
```

### 7.2 Queue Manager Service

**File:** `src/lib/queue/manager.ts`

```typescript
import { createClient } from '@/lib/supabase/server'

export interface QueuedMessage {
  id: string
  sessionId: string
  content: string
  status: 'pending' | 'processing' | 'done' | 'failed'
  dependsOn: string[]
}

export async function getNextMessage(sessionId: string): Promise<QueuedMessage | null> {
  const supabase = await createClient()

  // Get pending messages for this session
  const { data: pending } = await supabase
    .from('messages')
    .select('id, session_id, content, queue_status, depends_on')
    .eq('session_id', sessionId)
    .eq('sender_type', 'user')
    .eq('queue_status', 'pending')
    .order('queued_at', { ascending: true })
    .limit(1)

  if (!pending || pending.length === 0) return null

  const msg = pending[0]

  // Check dependencies
  if (msg.depends_on && msg.depends_on.length > 0) {
    const { data: deps } = await supabase
      .from('messages')
      .select('id, queue_status')
      .in('id', msg.depends_on)

    const pendingDeps = deps?.filter(d => d.queue_status !== 'done')
    if (pendingDeps && pendingDeps.length > 0) {
      return null // Dependencies not done
    }
  }

  // Mark as processing
  await supabase
    .from('messages')
    .update({ 
      queue_status: 'processing',
      started_at: new Date().toISOString()
    })
    .eq('id', msg.id)

  return {
    id: msg.id,
    sessionId: msg.session_id,
    content: msg.content,
    status: 'processing',
    dependsOn: msg.depends_on || []
  }
}

export async function markMessageDone(messageId: string): Promise<void> {
  const supabase = await createClient()
  
  await supabase
    .from('messages')
    .update({ 
      queue_status: 'done',
      completed_at: new Date().toISOString()
    })
    .eq('id', messageId)
}

export async function markMessageFailed(messageId: string, error: string): Promise<void> {
  const supabase = await createClient()
  
  await supabase
    .from('messages')
    .update({ 
      queue_status: 'failed',
      completed_at: new Date().toISOString(),
      metadata: supabase.sql`COALESCE(metadata, '{}'::jsonb) || '{"error": "${error}"}'::jsonb`
    })
    .eq('id', messageId)
}

export async function getQueueStatus(sessionId: string): Promise<{
  pending: number
  processing: number
  done: number
  failed: number
}> {
  const supabase = await createClient()
  
  const { data } = await supabase
    .from('messages')
    .select('queue_status')
    .eq('session_id', sessionId)
    .eq('sender_type', 'user')

  const counts = { pending: 0, processing: 0, done: 0, failed: 0 }
  data?.forEach(m => {
    const status = m.queue_status as keyof typeof counts
    if (status in counts) counts[status]++
  })
  
  return counts
}
```

### 7.3 Update ChatPanel for Non-Blocking

**File:** `src/components/ChatPanel.tsx`

Update `handleSendMessage` to not wait for response:

```typescript
const handleSendMessage = async (content: string) => {
  if (!activeSessionId) return

  // Send user message (this queues it)
  const res = await fetch(`/api/sessions/${activeSessionId}/messages`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ content }),
  })

  if (res.ok) {
    const data = await res.json()
    setMessages(prev => [...prev, data.message])

    // Trigger processing (fire and forget)
    processQueue(activeSessionId)
  }
}

// Process queue in background
const processQueue = async (sessionId: string) => {
  setAiLoading(true)
  
  try {
    const res = await fetch('/api/chat/complete', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ session_id: sessionId, use_tools: true }),
    })

    if (res.ok) {
      const contentType = res.headers.get('content-type') || ''
      if (contentType.includes('application/json')) {
        const data = await res.json()
        setMessages(prev => [...prev, data.message])
      }
    }
  } finally {
    setAiLoading(false)
  }
}
```

### 7.4 Queue Status Component

**File:** `src/components/chat/QueueStatus.tsx`

```typescript
'use client'

interface QueueStatusProps {
  pending: number
  processing: number
}

export function QueueStatus({ pending, processing }: QueueStatusProps) {
  if (pending === 0 && processing === 0) return null

  return (
    <div className="flex items-center gap-2 text-xs text-zinc-500 px-4 py-2 border-t border-zinc-800">
      {processing > 0 && (
        <span className="flex items-center gap-1">
          <span className="w-2 h-2 bg-yellow-500 rounded-full animate-pulse" />
          Processing {processing}...
        </span>
      )}
      {pending > 0 && (
        <span className="flex items-center gap-1">
          <span className="w-2 h-2 bg-zinc-500 rounded-full" />
          {pending} queued
        </span>
      )}
    </div>
  )
}
```

**Commit:** `feat(queue): non-blocking message queue`

**Build checkpoint:** `npm run build` → Queue works

---

## Phase 8: Agent Management (3h)

### 8.1 Agent Management Tool

**File:** `src/lib/tools/agent.ts`

```typescript
import { createClient } from '@/lib/supabase/server'
import type { ToolDefinition, ToolHandler, ToolResult } from './types'

export const agentToolDefinitions: ToolDefinition[] = [
  {
    name: 'manage_agent',
    description: 'Create, update, or list agents. Use to add new specialist agents or modify existing ones.',
    input_schema: {
      type: 'object',
      properties: {
        action: { 
          type: 'string', 
          enum: ['create', 'update', 'list', 'get'],
          description: 'Action to perform'
        },
        slug: { 
          type: 'string', 
          description: 'Agent slug (required for create/update/get)' 
        },
        name: { 
          type: 'string', 
          description: 'Agent display name' 
        },
        agent_type: { 
          type: 'string', 
          enum: ['lead', 'specialist', 'memory'],
          description: 'Type of agent'
        },
        role: {
          type: 'string',
          description: 'Short description of agent role'
        },
        system_prompt: { 
          type: 'string', 
          description: 'System prompt for the agent' 
        },
      },
      required: ['action'],
    },
  },
]

export const agentToolHandlers: Record<string, ToolHandler> = {
  manage_agent: async (input): Promise<ToolResult> => {
    const supabase = await createClient()
    const { action, slug, name, agent_type, role, system_prompt } = input as {
      action: string
      slug?: string
      name?: string
      agent_type?: string
      role?: string
      system_prompt?: string
    }

    switch (action) {
      case 'list': {
        const { data, error } = await supabase
          .from('agents')
          .select('slug, name, agent_type, role')
          .eq('is_system', true)
        
        if (error) return { success: false, error: error.message }
        return { 
          success: true, 
          data: { agents: data }
        }
      }

      case 'get': {
        if (!slug) return { success: false, error: 'slug required' }
        
        const { data, error } = await supabase
          .from('agents')
          .select('*')
          .eq('slug', slug)
          .eq('is_system', true)
          .single()
        
        if (error) return { success: false, error: error.message }
        return { success: true, data: { agent: data } }
      }

      case 'create': {
        if (!slug || !name) {
          return { success: false, error: 'slug and name required' }
        }

        const { data, error } = await supabase
          .from('agents')
          .insert({
            slug,
            name,
            agent_type: agent_type || 'specialist',
            role: role || '',
            system_prompt: system_prompt || '',
            is_system: true,
          })
          .select()
          .single()

        if (error) return { success: false, error: error.message }
        return { 
          success: true, 
          data: { agent: data, message: `Agent "${name}" created` }
        }
      }

      case 'update': {
        if (!slug) return { success: false, error: 'slug required' }

        const updates: Record<string, unknown> = {}
        if (name) updates.name = name
        if (agent_type) updates.agent_type = agent_type
        if (role) updates.role = role
        if (system_prompt) updates.system_prompt = system_prompt

        const { data, error } = await supabase
          .from('agents')
          .update(updates)
          .eq('slug', slug)
          .eq('is_system', true)
          .select()
          .single()

        if (error) return { success: false, error: error.message }
        return { 
          success: true, 
          data: { agent: data, message: `Agent "${slug}" updated` }
        }
      }

      default:
        return { success: false, error: `Unknown action: ${action}` }
    }
  },
}
```

### 8.2 Register Agent Tools

**File:** `src/lib/tools/executor.ts`

Add import and registration:

```typescript
import { agentToolDefinitions, agentToolHandlers } from './agent'

// Add to toolDefinitions
export const toolDefinitions: ToolDefinition[] = [
  ...githubToolDefinitions,
  ...modalToolDefinitions,
  ...agentToolDefinitions,
]

// Add to staticToolHandlers
const staticToolHandlers: Record<string, ToolHandler> = {
  ...modalToolHandlers,
  ...agentToolHandlers,
}
```

### 8.3 Execution Agent Definition

Create via the tool or directly in database:

```sql
INSERT INTO agents (slug, name, agent_type, role, system_prompt, is_system)
VALUES (
  'execution',
  'Execution',
  'specialist',
  'Code implementation and file modifications',
  'You are Execution, a specialist agent for implementing code changes in Gibbon.

Your role:
- Implement features and fix bugs
- Write clean, typed TypeScript code
- Follow existing patterns in the codebase
- Run builds to verify changes
- Create branches and PRs for changes

Rules:
- Always read relevant files before modifying
- Run npm run build after changes
- Create descriptive commit messages
- Ask for clarification if requirements are unclear

Available tools: github_read_file, github_write_file, github_create_branch, github_create_pr, execute_script',
  true
);
```

**Commit:** `feat(agents): agent management tool`

**Build checkpoint:** `npm run build` → Agent tool works

---

## Phase 9: GitHub Webhook (2h)

### 9.1 Webhook Endpoint

**File:** `src/app/api/webhooks/github/route.ts`

```typescript
import { NextResponse } from 'next/server'
import { createClient } from '@/lib/supabase/server'
import { indexProject, saveIndex } from '@/lib/indexer/service'
import crypto from 'crypto'

const WEBHOOK_SECRET = process.env.GITHUB_WEBHOOK_SECRET

function verifySignature(payload: string, signature: string): boolean {
  if (!WEBHOOK_SECRET) return true // Skip verification if no secret
  
  const hmac = crypto.createHmac('sha256', WEBHOOK_SECRET)
  const digest = 'sha256=' + hmac.update(payload).digest('hex')
  return crypto.timingSafeEqual(Buffer.from(digest), Buffer.from(signature))
}

export async function POST(request: Request) {
  const payload = await request.text()
  const signature = request.headers.get('x-hub-signature-256') || ''
  
  if (!verifySignature(payload, signature)) {
    return NextResponse.json({ error: 'Invalid signature' }, { status: 401 })
  }

  const event = request.headers.get('x-github-event')
  const body = JSON.parse(payload)

  if (event === 'push') {
    const repo = body.repository?.full_name
    if (!repo) {
      return NextResponse.json({ error: 'No repo' }, { status: 400 })
    }

    // Find project with this repo
    const supabase = await createClient()
    const { data: project } = await supabase
      .from('projects')
      .select('id')
      .eq('github_repo', repo)
      .single()

    if (project) {
      // Re-index in background
      indexProject(project.id)
        .then(files => saveIndex(supabase, project.id, files))
        .catch(err => console.error('Re-index failed:', err))
    }

    return NextResponse.json({ status: 'indexing' })
  }

  return NextResponse.json({ status: 'ignored' })
}
```

### 9.2 Register Webhook (Manual)

```bash
# Via GitHub API or UI
curl -X POST https://api.github.com/repos/hsheth-ape/gibbon/hooks \
  -H "Authorization: token YOUR_TOKEN" \
  -d '{
    "config": {
      "url": "https://gibbon.vercel.app/api/webhooks/github",
      "content_type": "json",
      "secret": "your-webhook-secret"
    },
    "events": ["push"]
  }'
```

**Commit:** `feat(webhook): auto-reindex on GitHub push`

**Build checkpoint:** `npm run build` → Webhook works

---

## Verification Checklist

After all phases:

- [ ] Session close button works
- [ ] Session outcome captured on close
- [ ] Project files indexed
- [ ] Orchestrator routes correctly
- [ ] Training logs captured
- [ ] Feedback UI shows on messages
- [ ] Messages process without blocking
- [ ] Lead can create agents via tool
- [ ] Index updates on repo push

---

## Test Commands

```
# Test orchestration
"Read the CURRENT.md and summarize what sprint we're on"
→ Should route to lead, include CURRENT.md in context

# Test agent creation  
"Create an execution agent that specializes in TypeScript code"
→ Should use manage_agent tool

# Test file index
"What files do we have related to chat?"
→ Should use file index to find relevant files

# Test queue
Send 3 messages quickly
→ Should queue and process in order
```

---

## After Sprint 4

Ready for:
- Sprint 5: Local LLM experimentation
- Sprint 5: Rich file index
- Sprint 5: Sonnet evaluation job
- Sprint 6: UX Agent

---

*Created: 2025-12-30 | Planning Session 009*
