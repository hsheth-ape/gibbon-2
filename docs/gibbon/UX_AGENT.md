# UX/UI Agent - Design Specification

> **Status:** Planned
> **Priority:** P1 - Sprint 4
> **Created:** 2025-12-30

## Overview

The UX Agent is a specialized AI agent that creates, modifies, and manages UI/UX designs programmatically through Penpot. It works alongside the Backend Agent, Lead Agent, and other specialists in Gibbon's agent ecosystem.

---

## Agent Identity

| Field | Value |
|-------|-------|
| **Name** | UX Agent |
| **Slug** | `ux-agent` |
| **Type** | specialist |
| **Model** | claude-sonnet-4-20250514 |
| **Is System** | true |

---

## Capabilities

### Core Functions

| Function | Description | Penpot API |
|----------|-------------|------------|
| `create_wireframe` | Generate wireframe from text description | create-file, update-file |
| `create_component` | Create reusable UI component | update-file |
| `modify_design` | Update existing design based on feedback | update-file |
| `list_designs` | List all designs in project | get-all-projects |
| `get_design` | Retrieve design details | get-file |
| `export_design` | Export design for handoff | export endpoints |

### Shape Generation

The agent can create these shape types:

| Shape | Use Cases |
|-------|-----------|
| `rect` | Buttons, inputs, cards, containers, backgrounds |
| `frame` | Artboards, component containers, sections |
| `text` | Labels, headings, body text (requires font config) |
| `circle` | Icons, avatars, indicators |
| `path` | Custom icons, decorations |

### Component Library Management

The agent manages a component library with:

```
.gibbon/
  components/
    buttons/
      primary.json      # Button definition + Penpot shape data
      secondary.json
      ghost.json
    inputs/
      text.json
      textarea.json
      select.json
    cards/
      basic.json
      media.json
    navigation/
      navbar.json
      sidebar.json
      tabs.json
    layout/
      container.json
      grid.json
      stack.json
```

Each component JSON contains:
- `name`: Human-readable name
- `description`: What it's for
- `variants`: Different states (default, hover, disabled)
- `penpot_shapes`: Shape definitions for Penpot API
- `props`: Configurable properties (size, color, text)
- `version`: Semantic version

---

## System Prompt

```markdown
You are the UX Agent, a specialist in creating and managing UI/UX designs for the Gibbon Product Factory.

## Your Role
You work with the Lead Agent to translate requirements into visual designs. When invoked, you:
1. Understand the design requirements from context
2. Create designs programmatically using Penpot API
3. Build from the component library when possible
4. Document design decisions
5. Provide links to view designs

## Available Tools
- `penpot_create_file`: Create new design file
- `penpot_add_shapes`: Add shapes to a file
- `penpot_list_components`: List available components
- `penpot_use_component`: Instantiate a component
- `penpot_export`: Export design for viewing

## Component Library
Always check the component library first. Reuse existing components when possible.
If a new component pattern emerges, propose adding it to the library.

## Design Principles
1. **Consistency**: Use design tokens (colors, spacing, typography)
2. **Simplicity**: Start with wireframes, add detail iteratively
3. **Accessibility**: Consider contrast, sizing, interaction targets
4. **Responsiveness**: Design for multiple screen sizes

## Output Format
After creating a design:
1. Provide Penpot file URL
2. Summarize what was created
3. Note any new components that should be added to library
4. Suggest next steps (feedback, refinement, handoff)
```

---

## Tools (Gibbon Integration)

### Tool Definitions

```typescript
const uxTools = [
  {
    name: "penpot_create_file",
    description: "Create a new design file in Penpot",
    parameters: {
      type: "object",
      properties: {
        name: { type: "string", description: "File name" },
        project_id: { type: "string", description: "Project ID (defaults to Drafts)" }
      },
      required: ["name"]
    }
  },
  {
    name: "penpot_add_shapes",
    description: "Add shapes to a Penpot file",
    parameters: {
      type: "object",
      properties: {
        file_id: { type: "string" },
        page_id: { type: "string" },
        shapes: {
          type: "array",
          items: {
            type: "object",
            properties: {
              type: { type: "string", enum: ["rect", "frame", "text", "circle"] },
              name: { type: "string" },
              x: { type: "number" },
              y: { type: "number" },
              width: { type: "number" },
              height: { type: "number" },
              fill: { type: "string", description: "Hex color" },
              stroke: { type: "string", description: "Hex color" }
            }
          }
        }
      },
      required: ["file_id", "page_id", "shapes"]
    }
  },
  {
    name: "penpot_use_component",
    description: "Instantiate a component from the library",
    parameters: {
      type: "object",
      properties: {
        component_path: { type: "string", description: "Path like 'buttons/primary'" },
        file_id: { type: "string" },
        page_id: { type: "string" },
        x: { type: "number" },
        y: { type: "number" },
        props: { type: "object", description: "Component props to override" }
      },
      required: ["component_path", "file_id", "page_id", "x", "y"]
    }
  }
];
```

---

## Version Control for Designs

### Design Versioning Strategy

Since Penpot designs live outside Git, we track them via metadata:

```
docs/designs/
  inventory.json        # Master list of all designs
  versions/
    2025-01-01_login-form_v1.json    # Snapshot metadata
    2025-01-02_login-form_v2.json    # Updated version
```

**inventory.json** structure:
```json
{
  "designs": [
    {
      "name": "Login Form",
      "penpot_file_id": "abc-123",
      "penpot_url": "https://penpot.../view/abc-123",
      "current_version": "1.2.0",
      "created_at": "2025-01-01T10:00:00Z",
      "updated_at": "2025-01-02T15:30:00Z",
      "status": "approved",
      "linked_artifacts": ["REQUIREMENTS.md#login"]
    }
  ]
}
```

**Version snapshot** structure:
```json
{
  "design_name": "Login Form",
  "version": "1.2.0",
  "penpot_file_id": "abc-123",
  "penpot_revn": 5,
  "snapshot_at": "2025-01-02T15:30:00Z",
  "changes": "Added forgot password link",
  "approved_by": "user-id",
  "shapes_count": 12,
  "thumbnail_url": "exported-thumbnail.png"
}
```

### Workflow

1. **Create**: UX Agent creates design → adds to inventory.json
2. **Update**: UX Agent modifies design → creates version snapshot
3. **Review**: Human reviews in Penpot → approves/requests changes
4. **Approve**: Human approves → status updated, version tagged
5. **Handoff**: Export specs → developers implement

---

## Integration with Gibbon UI

### Penpot Viewer Component

Add to artifacts panel to view Penpot designs inline:

```typescript
// src/components/PenpotViewer.tsx
interface PenpotViewerProps {
  fileId: string;
  pageId?: string;
}

export function PenpotViewer({ fileId, pageId }: PenpotViewerProps) {
  // Option 1: Embed via iframe (if Penpot supports)
  // Option 2: Render exported SVG/PNG
  // Option 3: Custom shape renderer using Penpot data
}
```

**Implementation Options:**

| Option | Pros | Cons |
|--------|------|------|
| iFrame embed | Native Penpot experience | May have CORS issues |
| SVG export | Fast, lightweight | Static, no interaction |
| Custom renderer | Full control | Complex to build |

**Recommended:** Start with SVG export (simplest), add iframe if Penpot supports embedding.

---

## Ways of Working: UI Changes

### UI Change Process

1. **Request**: Lead Agent receives UI requirement
2. **Invoke**: Lead Agent invokes UX Agent with context
3. **Design**: UX Agent creates/modifies design in Penpot
4. **Review**: UX Agent shares link, human reviews
5. **Iterate**: Feedback → UX Agent updates → repeat
6. **Approve**: Human approves design
7. **Handoff**: UX Agent exports specs, Backend Agent implements
8. **Verify**: QA Agent verifies implementation matches design

### Design Tokens

Centralized design system values:

```json
// .gibbon/design-tokens.json
{
  "colors": {
    "primary": "#4A90D9",
    "secondary": "#6C757D",
    "success": "#28A745",
    "warning": "#FFC107",
    "error": "#DC3545",
    "background": "#FFFFFF",
    "surface": "#F8F9FA",
    "text": "#212529",
    "text-muted": "#6C757D"
  },
  "spacing": {
    "xs": 4,
    "sm": 8,
    "md": 16,
    "lg": 24,
    "xl": 32
  },
  "typography": {
    "font-family": "Inter, sans-serif",
    "sizes": {
      "xs": 12,
      "sm": 14,
      "base": 16,
      "lg": 18,
      "xl": 20,
      "2xl": 24
    }
  },
  "radii": {
    "sm": 4,
    "md": 8,
    "lg": 12,
    "full": 9999
  }
}
```

---

## Next Steps

1. **Sprint 4 Execution:**
   - [ ] Create `lib/penpot/client.ts` - Penpot API wrapper
   - [ ] Create `lib/penpot/shapes.ts` - Shape generator helpers
   - [ ] Create `lib/tools/ux-tools.ts` - Tool definitions
   - [ ] Add UX Agent to agents table
   - [ ] Create initial component library

2. **First Design Task:**
   - [ ] Design Gibbon's main layout (chat + context panels)
   - [ ] Create component library foundation

3. **Integration:**
   - [ ] Penpot viewer in artifacts panel
   - [ ] Design inventory tracking

---

*Created: 2025-12-30 | Planning Session 007*
