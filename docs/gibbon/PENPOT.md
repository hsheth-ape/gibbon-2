# Penpot Integration Guide

## Overview

Penpot is an open-source design platform (Figma alternative) that we've deployed self-hosted on Elestio. This enables programmatic design creation via API - a key capability for the future UX Agent.

## Instance Details

| Item | Value |
|------|-------|
| **URL** | https://penpot-sandbox-u7533.vm.elestio.app |
| **Admin Email** | hsheth@apeability.com |
| **Admin Password** | epSwwVr-4OyXXqVshdxY3 |
| **API Token** | See below |
| **pgAdmin** | https://penpot-sandbox-u7533.vm.elestio.app:56590 |

## Project Organization

**CRITICAL:** Each product gets its own Penpot project. Do NOT mix designs across products.

| Product | Penpot Project | File Naming |
|---------|----------------|-------------|
| **Gibbon** | `Gibbon` (create if missing) | `gibbon-{feature}-{version}` |
| **Psychit** | `Psychit` (create if missing) | `psychit-{feature}-{version}` |

### Creating a Project
```python
# First time setup - create project for your product
payload = f'["^ ","~:team-id","~u{TEAM_ID}","~:name","Gibbon"]'  # or "Psychit"
r = requests.post(f"{URL}/api/rpc/command/create-project", headers=HEADERS, data=payload)
# Save the returned project-id in your product's CURRENT.md
```

### Before Creating Any Design
1. Check you're using the correct project ID for your product
2. Use proper file naming: `{product}-{feature}-{version}`
3. Never create files in "Drafts" — always use product-specific project



### API Access Token
```
eyJhbGciOiJBMjU2S1ciLCJlbmMiOiJBMjU2R0NNIn0.9fJnoqETQD87dpTeooHTTIZHEDAueVR5GN0e9Y-I1TzBcWtMZ9keYQ.3cI2a2z9nJTbh0mb.Nl5XiwcTUiFH6cqz_wHAA-wSHajRDA2HNq6cZKQjQXhA_iCW-Rnj2Jgyv_zcnbaYIbHKeewz9GfoUNKRNvOLFbJcqD4FwCi8teJvxgG74RUdFwP-Yer7pCDOmZfuHHB4WPIqK-FuIKVatAqD4gu6zkXMiOeHKoF-8ShLPaOoNcVtn5lyEAPW1pwnsxRJbhDhUnjKX55lnnPU.EMf4rDxbMQBzloI1FzpqPg
```

## Why Self-Hosted?

Public Penpot (design.penpot.app) blocks API access via Cloudflare. Self-hosted instance required for programmatic access.

## API Basics

### Authentication
All API calls require the token in header:
```
Authorization: Token <ACCESS_TOKEN>
```

### Content Type
Penpot uses Transit+JSON format:
```
Content-Type: application/transit+json
```

### Base URL
```
https://penpot-sandbox-u7533.vm.elestio.app/api/rpc/command/
```

## Core API Operations

### 1. Get Profile
```python
import requests

TOKEN = "<access_token>"
URL = "https://penpot-sandbox-u7533.vm.elestio.app"
HEADERS = {"Authorization": f"Token {TOKEN}", "Content-Type": "application/transit+json"}

r = requests.post(f"{URL}/api/rpc/command/get-profile", headers=HEADERS, data='["^ "]')
# Returns: user profile with id, email, default-project-id, default-team-id
```

### 2. List Projects
```python
r = requests.post(f"{URL}/api/rpc/command/get-all-projects", headers=HEADERS, data='["^ "]')
```

### 3. Create File
```python
PROJECT_ID = "58935970-9f78-80fb-8007-56e9d0b538f5"  # Drafts project

payload = f'["^ ","~:project-id","~u{PROJECT_ID}","~:name","My Design"]'
r = requests.post(f"{URL}/api/rpc/command/create-file", headers=HEADERS, data=payload)
# Returns: file with id, pages, page-id
```

### 4. Add Shapes to File (The Complex Part)

Shapes require complete geometry data. Here's a working example:

```python
import uuid

FILE_ID = "<file-id>"
PAGE_ID = "<page-id>"
SESSION_ID = str(uuid.uuid4())
SHAPE_ID = str(uuid.uuid4())

def make_rect(name, x, y, w, h, fill_color, stroke_color="#666666", stroke_width=1):
    shape_id = str(uuid.uuid4())
    return f"""["^ ",
   "~:type", "~:add-obj",
   "~:id", "~u{shape_id}",
   "~:page-id", "~u{PAGE_ID}",
   "~:parent-id", "~u00000000-0000-0000-0000-000000000000",
   "~:frame-id", "~u00000000-0000-0000-0000-000000000000",
   "~:obj", ["^ ",
     "~:id", "~u{shape_id}",
     "~:type", "~:rect",
     "~:name", "{name}",
     "~:x", {x},
     "~:y", {y},
     "~:width", {w},
     "~:height", {h},
     "~:selrect", ["^ ", "~:x", {x}, "~:y", {y}, "~:width", {w}, "~:height", {h}, 
                   "~:x1", {x}, "~:y1", {y}, "~:x2", {x+w}, "~:y2", {y+h}],
     "~:points", [
       ["^ ", "~:x", {x}, "~:y", {y}],
       ["^ ", "~:x", {x+w}, "~:y", {y}],
       ["^ ", "~:x", {x+w}, "~:y", {y+h}],
       ["^ ", "~:x", {x}, "~:y", {y+h}]
     ],
     "~:transform", ["^ ", "~:a", 1, "~:b", 0, "~:c", 0, "~:d", 1, "~:e", 0, "~:f", 0],
     "~:transform-inverse", ["^ ", "~:a", 1, "~:b", 0, "~:c", 0, "~:d", 1, "~:e", 0, "~:f", 0],
     "~:parent-id", "~u00000000-0000-0000-0000-000000000000",
     "~:frame-id", "~u00000000-0000-0000-0000-000000000000",
     "~:rotation", 0,
     "~:fills", [["^ ", "~:fill-color", "{fill_color}", "~:fill-opacity", 1]],
     "~:strokes", [["^ ", "~:stroke-color", "{stroke_color}", "~:stroke-width", {stroke_width}, 
                   "~:stroke-opacity", 1, "~:stroke-style", "~:solid", "~:stroke-alignment", "~:center"]]
   ]
  ]"""

# Create payload with changes
payload = f"""["^ ",
"~:id", "~u{FILE_ID}",
"~:session-id", "~u{SESSION_ID}",
"~:revn", 0,
"~:vern", 0,
"~:changes", [
  {make_rect("My Rectangle", 100, 100, 200, 150, "#4A90D9")}
]
]"""

r = requests.post(f"{URL}/api/rpc/command/update-file", headers=HEADERS, data=payload)
```

## Shape Types

| Type | Use Case |
|------|----------|
| `rect` | Rectangles, buttons, cards, inputs |
| `frame` | Containers, artboards |
| `circle` | Circular elements |
| `text` | Text content (complex - needs font data) |
| `path` | Custom shapes, icons |

## Required Shape Fields

Every shape MUST have:
- `id` - UUID
- `type` - Shape type
- `name` - Display name
- `x`, `y`, `width`, `height` - Position/size
- `selrect` - Selection rectangle with x, y, width, height, x1, y1, x2, y2
- `points` - Array of 4 corner points
- `transform` - Identity matrix (a=1, b=0, c=0, d=1, e=0, f=0)
- `transform-inverse` - Same as transform
- `parent-id` - Parent shape UUID (root = 00000000-0000-0000-0000-000000000000)
- `frame-id` - Containing frame UUID
- `rotation` - Rotation in degrees
- `fills` - Array of fill objects
- `strokes` - Array of stroke objects

## Transit+JSON Format Notes

- Maps: `["^ ", "key1", value1, "key2", value2]`
- UUIDs: `"~u<uuid-string>"`
- Keywords: `"~:keyword-name"`
- Arrays: Standard JSON arrays

## Elestio Configuration

### PENPOT_FLAGS (in .env)
```
PENPOT_FLAGS='enable-smtp enable-registration enable-prepl-server disable-email-verification enable-access-tokens'
```

Key flags:
- `disable-email-verification` - Skip email verification for new users
- `enable-access-tokens` - Enable API token creation in UI

### Creating Access Tokens
1. Login to Penpot UI
2. Go to Profile → Access Tokens
3. Create new token
4. Copy and store securely

## Lessons Learned

1. **Public Penpot blocked** - Cloudflare blocks API requests, must self-host
2. **Backend startup slow** - Java/Clojure takes 30-60 seconds to start
3. **Shape geometry required** - Can't just send x/y/w/h, need full selrect/points/transforms
4. **Transit format tricky** - Use the exact format shown, very sensitive to structure
5. **revn tracking** - File revision number must increment with changes
6. **session-id required** - Each update-file call needs unique session UUID

## UX Agent Potential

With this API, a UX Agent could:
1. **Generate wireframes** from text descriptions
2. **Create UI components** (buttons, forms, cards)
3. **Build full page layouts** programmatically
4. **Read existing designs** for analysis
5. **Modify designs** based on feedback
6. **Export designs** for developer handoff

## Related Files
- Session log: `docs/gibbon/sessions/2025-12-30-planning-005.md`
- Backlog item: See BACKLOG.md for UX Agent task

---
*Created: 2025-12-30 | Last Updated: 2025-12-30*
