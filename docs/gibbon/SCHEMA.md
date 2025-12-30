# Gibbon - Database Schema

> **Last Updated:** 2025-12-29
> **Database:** Supabase (PostgreSQL)
> **Status:** Sprint 1 Design

## Overview

Sprint 1 schema covers: users, workspaces, memberships, invites, projects, artifacts, artifact versions, and tasks.

## Entity Relationship

```
auth.users (Supabase managed)
    │
    └──< users (profile extension)
            │
            ├──< workspaces (created_by)
            │       │
            │       ├──< workspace_memberships
            │       │       └──> users
            │       │
            │       ├──< workspace_invites
            │       │
            │       └──< projects
            │               │
            │               ├──< artifacts
            │               │       │
            │               │       └──< artifact_versions
            │               │
            │               └──< tasks
            │
            └──< (various created_by, assigned_to references)
```

## Tables

### users

Extends Supabase `auth.users` with profile data.

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  email TEXT UNIQUE NOT NULL,
  full_name TEXT,
  avatar_url TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Trigger to auto-create user profile on signup
CREATE OR REPLACE FUNCTION handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.users (id, email, full_name, avatar_url)
  VALUES (
    NEW.id,
    NEW.email,
    NEW.raw_user_meta_data->>'full_name',
    NEW.raw_user_meta_data->>'avatar_url'
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION handle_new_user();
```

### workspaces

```sql
CREATE TABLE workspaces (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  created_by UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_workspaces_slug ON workspaces(slug);
CREATE INDEX idx_workspaces_created_by ON workspaces(created_by);
```

### workspace_memberships

```sql
CREATE TYPE workspace_role AS ENUM ('owner', 'member', 'viewer');

CREATE TABLE workspace_memberships (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role workspace_role NOT NULL DEFAULT 'member',
  invited_by UUID REFERENCES users(id),
  invited_at TIMESTAMPTZ DEFAULT NOW(),
  accepted_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  
  UNIQUE(workspace_id, user_id)
);

CREATE INDEX idx_memberships_workspace ON workspace_memberships(workspace_id);
CREATE INDEX idx_memberships_user ON workspace_memberships(user_id);
```

### workspace_invites

```sql
CREATE TABLE workspace_invites (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
  email TEXT NOT NULL,
  role workspace_role NOT NULL DEFAULT 'member',
  invited_by UUID NOT NULL REFERENCES users(id),
  token TEXT UNIQUE NOT NULL DEFAULT encode(gen_random_bytes(32), 'hex'),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  expires_at TIMESTAMPTZ DEFAULT (NOW() + INTERVAL '7 days'),
  accepted_at TIMESTAMPTZ,
  
  UNIQUE(workspace_id, email)
);

CREATE INDEX idx_invites_email ON workspace_invites(email);
CREATE INDEX idx_invites_token ON workspace_invites(token);
```

### projects

```sql
CREATE TABLE projects (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  description TEXT,
  archived_at TIMESTAMPTZ,
  created_by UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_projects_workspace ON projects(workspace_id);
CREATE INDEX idx_projects_archived ON projects(archived_at) WHERE archived_at IS NULL;
```

### artifacts

```sql
CREATE TYPE artifact_type AS ENUM (
  'CURRENT',
  'REQUIREMENTS', 
  'DECISIONS',
  'SCHEMA',
  'API_CONTRACT',
  'LESSONS'
);

CREATE TABLE artifacts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  type artifact_type NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  
  UNIQUE(project_id, type)
);

CREATE INDEX idx_artifacts_project ON artifacts(project_id);
```

### artifact_versions

```sql
CREATE TABLE artifact_versions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  artifact_id UUID NOT NULL REFERENCES artifacts(id) ON DELETE CASCADE,
  version_number INTEGER NOT NULL,
  content TEXT NOT NULL DEFAULT '',
  created_by UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  
  UNIQUE(artifact_id, version_number)
);

CREATE INDEX idx_versions_artifact ON artifact_versions(artifact_id);
CREATE INDEX idx_versions_latest ON artifact_versions(artifact_id, version_number DESC);
```

### tasks

```sql
CREATE TYPE task_type AS ENUM ('assist', 'draft', 'act');
CREATE TYPE task_status AS ENUM ('proposed', 'approved', 'in_progress', 'review', 'done');

CREATE TABLE tasks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  description TEXT,
  type task_type NOT NULL DEFAULT 'draft',
  status task_status NOT NULL DEFAULT 'proposed',
  assigned_to UUID REFERENCES users(id),
  created_by UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_tasks_project ON tasks(project_id);
CREATE INDEX idx_tasks_status ON tasks(project_id, status);
CREATE INDEX idx_tasks_assigned ON tasks(assigned_to) WHERE assigned_to IS NOT NULL;
```

---

## RLS Policies

### users

```sql
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- Users can read any user (for display purposes)
CREATE POLICY "Users are viewable by authenticated users"
  ON users FOR SELECT
  TO authenticated
  USING (true);

-- Users can update their own profile
CREATE POLICY "Users can update own profile"
  ON users FOR UPDATE
  TO authenticated
  USING (auth.uid() = id);
```

### workspaces

```sql
ALTER TABLE workspaces ENABLE ROW LEVEL SECURITY;

-- Users can view workspaces they're members of
CREATE POLICY "Users can view their workspaces"
  ON workspaces FOR SELECT
  TO authenticated
  USING (
    EXISTS (
      SELECT 1 FROM workspace_memberships
      WHERE workspace_id = workspaces.id
      AND user_id = auth.uid()
    )
  );

-- Any authenticated user can create a workspace
CREATE POLICY "Users can create workspaces"
  ON workspaces FOR INSERT
  TO authenticated
  WITH CHECK (auth.uid() = created_by);

-- Only owners can update workspace
CREATE POLICY "Owners can update workspace"
  ON workspaces FOR UPDATE
  TO authenticated
  USING (
    EXISTS (
      SELECT 1 FROM workspace_memberships
      WHERE workspace_id = workspaces.id
      AND user_id = auth.uid()
      AND role = 'owner'
    )
  );
```

### workspace_memberships

```sql
ALTER TABLE workspace_memberships ENABLE ROW LEVEL SECURITY;

-- Members can view other members in their workspaces
CREATE POLICY "Members can view workspace members"
  ON workspace_memberships FOR SELECT
  TO authenticated
  USING (
    EXISTS (
      SELECT 1 FROM workspace_memberships wm
      WHERE wm.workspace_id = workspace_memberships.workspace_id
      AND wm.user_id = auth.uid()
    )
  );

-- Owners can manage memberships
CREATE POLICY "Owners can manage memberships"
  ON workspace_memberships FOR ALL
  TO authenticated
  USING (
    EXISTS (
      SELECT 1 FROM workspace_memberships
      WHERE workspace_id = workspace_memberships.workspace_id
      AND user_id = auth.uid()
      AND role = 'owner'
    )
  );
```

### projects

```sql
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

-- Workspace members can view projects
CREATE POLICY "Members can view projects"
  ON projects FOR SELECT
  TO authenticated
  USING (
    EXISTS (
      SELECT 1 FROM workspace_memberships
      WHERE workspace_id = projects.workspace_id
      AND user_id = auth.uid()
    )
  );

-- Members and owners can create projects
CREATE POLICY "Members can create projects"
  ON projects FOR INSERT
  TO authenticated
  WITH CHECK (
    EXISTS (
      SELECT 1 FROM workspace_memberships
      WHERE workspace_id = projects.workspace_id
      AND user_id = auth.uid()
      AND role IN ('owner', 'member')
    )
  );

-- Members and owners can update projects
CREATE POLICY "Members can update projects"
  ON projects FOR UPDATE
  TO authenticated
  USING (
    EXISTS (
      SELECT 1 FROM workspace_memberships
      WHERE workspace_id = projects.workspace_id
      AND user_id = auth.uid()
      AND role IN ('owner', 'member')
    )
  );
```

### artifacts & artifact_versions

```sql
ALTER TABLE artifacts ENABLE ROW LEVEL SECURITY;
ALTER TABLE artifact_versions ENABLE ROW LEVEL SECURITY;

-- Inherit project access for artifacts
CREATE POLICY "Members can view artifacts"
  ON artifacts FOR SELECT
  TO authenticated
  USING (
    EXISTS (
      SELECT 1 FROM projects p
      JOIN workspace_memberships wm ON wm.workspace_id = p.workspace_id
      WHERE p.id = artifacts.project_id
      AND wm.user_id = auth.uid()
    )
  );

CREATE POLICY "Members can manage artifacts"
  ON artifacts FOR ALL
  TO authenticated
  USING (
    EXISTS (
      SELECT 1 FROM projects p
      JOIN workspace_memberships wm ON wm.workspace_id = p.workspace_id
      WHERE p.id = artifacts.project_id
      AND wm.user_id = auth.uid()
      AND wm.role IN ('owner', 'member')
    )
  );

-- Same pattern for versions
CREATE POLICY "Members can view versions"
  ON artifact_versions FOR SELECT
  TO authenticated
  USING (
    EXISTS (
      SELECT 1 FROM artifacts a
      JOIN projects p ON p.id = a.project_id
      JOIN workspace_memberships wm ON wm.workspace_id = p.workspace_id
      WHERE a.id = artifact_versions.artifact_id
      AND wm.user_id = auth.uid()
    )
  );

CREATE POLICY "Members can create versions"
  ON artifact_versions FOR INSERT
  TO authenticated
  WITH CHECK (
    EXISTS (
      SELECT 1 FROM artifacts a
      JOIN projects p ON p.id = a.project_id
      JOIN workspace_memberships wm ON wm.workspace_id = p.workspace_id
      WHERE a.id = artifact_versions.artifact_id
      AND wm.user_id = auth.uid()
      AND wm.role IN ('owner', 'member')
    )
  );
```

### tasks

```sql
ALTER TABLE tasks ENABLE ROW LEVEL SECURITY;

-- Same pattern as projects
CREATE POLICY "Members can view tasks"
  ON tasks FOR SELECT
  TO authenticated
  USING (
    EXISTS (
      SELECT 1 FROM projects p
      JOIN workspace_memberships wm ON wm.workspace_id = p.workspace_id
      WHERE p.id = tasks.project_id
      AND wm.user_id = auth.uid()
    )
  );

CREATE POLICY "Members can manage tasks"
  ON tasks FOR ALL
  TO authenticated
  USING (
    EXISTS (
      SELECT 1 FROM projects p
      JOIN workspace_memberships wm ON wm.workspace_id = p.workspace_id
      WHERE p.id = tasks.project_id
      AND wm.user_id = auth.uid()
      AND wm.role IN ('owner', 'member')
    )
  );
```

---

## Helper Functions

### Create workspace with owner membership

```sql
CREATE OR REPLACE FUNCTION create_workspace_with_owner(
  workspace_name TEXT,
  workspace_slug TEXT
)
RETURNS workspaces AS $$
DECLARE
  new_workspace workspaces;
BEGIN
  -- Create workspace
  INSERT INTO workspaces (name, slug, created_by)
  VALUES (workspace_name, workspace_slug, auth.uid())
  RETURNING * INTO new_workspace;
  
  -- Add creator as owner
  INSERT INTO workspace_memberships (workspace_id, user_id, role, accepted_at)
  VALUES (new_workspace.id, auth.uid(), 'owner', NOW());
  
  RETURN new_workspace;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

### Create project with seeded artifacts

```sql
CREATE OR REPLACE FUNCTION create_project_with_artifacts(
  p_workspace_id UUID,
  p_name TEXT,
  p_description TEXT DEFAULT NULL
)
RETURNS projects AS $$
DECLARE
  new_project projects;
  artifact_types artifact_type[] := ARRAY['CURRENT', 'REQUIREMENTS', 'DECISIONS', 'SCHEMA', 'API_CONTRACT', 'LESSONS'];
  atype artifact_type;
  new_artifact artifacts;
BEGIN
  -- Create project
  INSERT INTO projects (workspace_id, name, description, created_by)
  VALUES (p_workspace_id, p_name, p_description, auth.uid())
  RETURNING * INTO new_project;
  
  -- Seed artifacts
  FOREACH atype IN ARRAY artifact_types LOOP
    INSERT INTO artifacts (project_id, type)
    VALUES (new_project.id, atype)
    RETURNING * INTO new_artifact;
    
    -- Create initial version
    INSERT INTO artifact_versions (artifact_id, version_number, content, created_by)
    VALUES (new_artifact.id, 1, '', auth.uid());
  END LOOP;
  
  RETURN new_project;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

---

## Migrations

| Version | Date | Description |
|---------|------|-------------|
| 0.0.0 | 2025-12-28 | Initial setup (no tables yet) |
| 0.1.0 | TBD | Sprint 1: users, workspaces, projects, artifacts, tasks |

---

## Future Tables (Sprint 2+)

These will be added in future sprints:

- `agent_definitions` — Agent configurations
- `chats` — Chat sessions (task, project, system level)
- `chat_messages` — Individual messages
- `execution_logs` — Agent execution traces
- `files` — Uploaded files metadata
