# ArchionLabs — Codebase Deep Dive (Technical Interview Reference)

> **Archion-Build** = AI-powered floor plan generation & editing  
> **Archion-Sim** = MARL-powered pedestrian simulation & building compliance analysis

---

## Table of Contents

1. [System Architecture Overview](#1-system-architecture-overview)
2. [Tech Stack Summary](#2-tech-stack-summary)
3. [Archion-Build Backend](#3-archion-build-backend)
4. [Archion-Build Frontend](#4-archion-build-frontend)
5. [Archion-Sim Backend](#5-archion-sim-backend)
6. [Archion-Sim Frontend](#6-archion-sim-frontend)
7. [Infrastructure & Deployment](#7-infrastructure--deployment)
8. [Key Design Decisions & Interview Talking Points](#8-key-design-decisions--interview-talking-points)

---

## 1. System Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                         ArchionLabs Platform                         │
├──────────────────────────────┬───────────────────────────────────────┤
│      ARCHION-BUILD           │         ARCHION-SIM                   │
│  (Floor Plan Generator)      │  (Pedestrian Simulation)              │
│                              │                                       │
│  ┌────────────────────────┐  │  ┌─────────────────────────────────┐  │
│  │  Next.js Frontend      │  │  │  Next.js Frontend               │  │
│  │  • 2D Canvas Renderer  │  │  │  • Three.js 3D Model Viewer     │  │
│  │  • 3D Viewer           │  │  │  • Simulation Playback          │  │
│  │  • Chat Interface      │  │  │  • Analytics Dashboard          │  │
│  │  • Zustand State       │  │  │  • Compliance Panel             │  │
│  └──────────┬─────────────┘  │  └──────────┬──────────────────────┘  │
│             │ REST API       │             │ REST + SSE               │
│  ┌──────────▼─────────────┐  │  ┌──────────▼──────────────────────┐  │
│  │  FastAPI Backend       │  │  │  FastAPI Backend                │  │
│  │  • Claude LLM (NLP)    │  │  │  • SimulationEngine (MARL)     │  │
│  │  • LayoutSolver (Algo) │  │  │  • Claude LLM (Strategy+AI)    │  │
│  │  • 3D Model Gen        │  │  │  • ComplianceChecker           │  │
│  │  • SQLAlchemy ORM      │  │  │  • AnalyticsEngine             │  │
│  └──────────┬─────────────┘  │  │  • Random Forest ML            │  │
│             │                │  │  • PDF ReportGenerator          │  │
│  ┌──────────▼─────────────┐  │  │  • Trimesh 3D Geometry         │  │
│  │  PostgreSQL (Supabase) │  │  └─────────────────────────────────┘  │
│  └────────────────────────┘  │                                       │
└──────────────────────────────┴───────────────────────────────────────┘
```

---

## 2. Tech Stack Summary

| Layer | Archion-Build | Archion-Sim |
|---|---|---|
| **Frontend** | Next.js, TypeScript, Tailwind CSS, Shadcn/UI, Zustand, HTML5 Canvas | Next.js, TypeScript, Three.js |
| **Backend** | Python FastAPI, SQLAlchemy, Pydantic | Python FastAPI, Pydantic, SSE (Server-Sent Events) |
| **AI/LLM** | Anthropic Claude 3.5 Haiku / 4 Sonnet | Anthropic Claude 3 Haiku |
| **3D/Geometry** | Custom Three.js JSON generator, CadQuery scripts | Trimesh, Shapely, SciPy, NumPy |
| **ML** | — | Scikit-learn (Random Forest), joblib |
| **Database** | PostgreSQL via Supabase, SQLAlchemy ORM | In-memory state |
| **Auth** | Supabase JWT (python-jose) | Supabase JWT (python-jose) |
| **PDF Reports** | — | ReportLab, Matplotlib |
| **Deployment** | Docker + Nginx (VPS), Vercel (frontend) | Docker + Nginx (VPS), Vercel (frontend) |

### Why These Technologies?

- **FastAPI**: Async Python framework with automatic OpenAPI docs, Pydantic validation, and dependency injection. Chosen for its speed and developer experience with LLM integrations.
- **Claude (Anthropic)**: Best-in-class LLM for structured output generation. Used for natural language → floor plan conversion AND strategic agent navigation.
- **Zustand**: Lightweight state management (< 1KB) vs Redux's boilerplate. Perfect for managing complex editor state (undo/redo, tool selection, canvas transforms).
- **Shapely**: Industry-standard computational geometry library for 2D polygon operations (buffering, intersection, containment testing).
- **Trimesh**: Robust 3D mesh loading (OBJ/GLB/GLTF) with cross-section slicing for extracting 2D floor plans from 3D models.
- **ReportLab**: PDF generation library used to produce professional compliance audit reports with charts and tables.

---

## 3. Archion-Build Backend

### 3.1 Entry Point — `main.py`

```python
# FastAPI application factory
app = FastAPI(title="Archion Build API")
# CORS middleware allows frontend origins
app.add_middleware(CORSMiddleware, ...)
# Single router for generation endpoints
app.include_router(generate_router, prefix="/api/v1")
```

**Key Logic**: Configures CORS from environment variable `CORS_ORIGINS` (comma-separated list). Creates database tables on startup via SQLAlchemy `Base.metadata.create_all()`.

### 3.2 Configuration — `config.py`

| Env Variable | Purpose |
|---|---|
| `DATABASE_URL` | PostgreSQL connection string (Supabase) |
| `SUPABASE_JWT_SECRET` | JWT verification key |
| `CORS_ORIGINS` | Allowed frontend domains |
| `ANTHROPIC_API_KEY` | Claude API key |
| `ANTHROPIC_MODEL` | Default Claude model (e.g., `claude-3-5-haiku-20241022`) |

### 3.3 Database Layer — `database.py`

Uses SQLAlchemy with **conditional engine creation**:
- If `DATABASE_URL` starts with `postgresql://` → PostgreSQL engine
- Otherwise → SQLite fallback (`archion_build.db`)
- Provides `get_db` FastAPI dependency for session management

**Why**: Enables local development without a Supabase instance while maintaining the same ORM interface.

### 3.4 Authentication — `auth.py`

Two FastAPI dependencies:

| Dependency | Behaviour |
|---|---|
| `get_current_user` | **Strict**: Requires valid JWT → returns `user_id` (UUID) |
| `get_current_user_optional` | **Lenient**: Returns `user_id` or `None` if no/invalid token |

**Implementation**: Decodes Supabase JWT using `python-jose` with HS256 algorithm, extracts the `sub` claim as the user ID. Audience verification is disabled (`verify_aud=False`) because Supabase tokens don't always include a standard `aud` claim.

### 3.5 API Router — `routers/generate.py`

Five endpoints, **all public** (no auth required for generation, by design):

#### `GET /projects` — List user projects
Uses optional auth. Queries `Project` table filtered by `user_id`, returns list of saved projects.

#### `POST /floorplan` — Generate floor plan from prompt
```
User Prompt → IntentParser.parse() → LayoutSolver.solve() → FloorPlan JSON
```
1. **Intent Parsing** (via `claude_intent.py`): Extracts room requirements (type, area, count) from natural language
2. **Layout Solving** (via `geometry.py`): Algorithmically arranges rooms using strip-packing
3. **Persistence**: Saves project + chat history to PostgreSQL
4. **Dashboard Sync**: Upserts to Supabase `projects` table for the landing page dashboard

Supports **modification**: If `current_floorplan` is provided, Claude modifies the existing plan rather than generating from scratch.

#### `POST /extract-floorplan` — Extract from uploaded file
Supports images (PNG/JPG), PDF, and DXF files:
- **Images/PDF**: Base64-encoded → Claude Vision API extracts room layout
- **DXF**: Parsed using `ezdxf` library to extract geometry

#### `POST /threejs` — Generate Three.js JSON
Converts floor plan to Three.js-compatible JSON (`BufferGeometry` with materials) for 3D visualization.

#### `POST /cadquery` — Generate CadQuery script
Produces a Python script using CadQuery for CAD export (STL/STEP format).

### 3.6 Data Models

#### Pydantic Models — `models/floorplan.py`
Hierarchical floor plan structure:
```
FloorPlan
  ├── name, total_area, width, height, metadata
  └── levels: Level[]
        ├── level_number, name, height
        ├── rooms: Room[]      (id, name, room_type, bounding_box, area, vertices)
        ├── walls: Wall[]      (id, start, end, thickness, is_exterior)
        ├── doors: Door[]      (id, position, width, wall_start, wall_end, is_exterior)
        └── windows: Window[]  (id, position, width, wall_start, wall_end)
```

#### ORM Models — `models/db_models.py`
| Table | Columns | Purpose |
|---|---|---|
| `Project` | id, user_id, name, prompt, floorplan_json, model, created_at, updated_at | Persists generated floor plans |
| `ChatHistory` | id, project_id, role, content, created_at | Stores conversation history |

### 3.7 NLP Service — `services/claude_intent.py` (1,051 lines)

The core intelligence layer. Three parsing strategies:

#### Strategy 1: LLM Room Extraction (`_extract_rooms_llm`)
Sends prompt to Claude with a system prompt asking it to return a JSON list of `{room_type, area_sqm, count}`. Used for simple prompts like "3 bedroom apartment".

#### Strategy 2: Direct Full Plan Generation (`_generate_full_plan_llm`)
For complex prompts, Claude generates the **entire floor plan JSON** (rooms, walls, doors, windows) in one shot. Uses a detailed system prompt with:
- Coordinate system rules (meters, origin at top-left)
- Room type taxonomy (14 types)
- Wall generation rules (exterior + interior)
- Door/window placement logic

#### Strategy 3: Plan Modification (`_modify_floorplan_llm`)
Sends the **current floor plan JSON** + user's modification request to Claude. Claude returns a modified version of the JSON. Example: "Add a balcony to the master bedroom".

#### Fallback: Regex Parsing (`_extract_rooms_regex`)
When Claude is unavailable, uses regex patterns:
```python
# Matches: "3 bedrooms", "living room 25sqm", "2 bathrooms of 8m2"
pattern = r'(\d+)\s*(bedroom|bathroom|kitchen|living|dining|...)'
```

#### Validation & Repair (`_validate_and_repair_geometry`)
After generation, runs geometric consistency checks:
- Rooms within floor plan boundaries
- Walls aligned with room edges
- Doors/windows placed on actual walls
- No overlapping rooms
- Minimum area thresholds

### 3.8 Geometry Engine — `services/geometry.py` (520 lines)

The `LayoutSolver` class implements a **strip-packing algorithm**:

```python
class LayoutSolver:
    def solve(self, requirements: list[RoomReq]) -> FloorPlan:
        # 1. Sort rooms by area (largest first)
        # 2. Calculate total area → derive building footprint
        # 3. Pack rooms into strips (row-by-row)
        # 4. Generate walls along room boundaries
        # 5. Place doors (one per interior wall) and windows (exterior walls)
```

**Strip-packing approach**:
1. Choose building width based on aspect ratio heuristic
2. Place rooms left-to-right in rows
3. When a row fills up, start a new row below
4. Generates **exterior walls** around the perimeter
5. Generates **interior walls** between adjacent rooms
6. Places **doors** at midpoints of interior walls
7. Places **windows** on exterior walls (one per room)

### 3.9 3D Model Generator — `services/model3d.py`

Two output formats:

#### Three.js JSON
Converts each architectural element into Three.js `BufferGeometry`:
- Walls → extruded rectangles (`BoxGeometry` equivalent)
- Floors → flat planes at Z=0
- Room fills → coloured planes (room-type specific colours)
- Doors → green indicators
- Windows → blue indicators

#### CadQuery Script
Generates a Python script that uses the CadQuery library:
```python
# Generated output looks like:
result = cq.Workplane("XY")
for wall in walls:
    result = result.add(cq.Workplane("XY").box(...))
```

### 3.10 Dashboard Sync — `services/dashboard_sync.py`

Upserts project references into the Supabase `projects` table using raw SQL (`INSERT ... ON CONFLICT DO UPDATE`). This allows the landing page dashboard to display users' recent projects across all Archion apps.

---

## 4. Archion-Build Frontend

### 4.1 Tech Stack
- **Next.js** (App Router)
- **TypeScript** (strict typing)
- **Tailwind CSS** + **Shadcn/UI** (component library)
- **Zustand** (state management)
- **HTML5 Canvas** (2D floor plan rendering)
- **Lucide React** (icons)

### 4.2 API Service Layer — `services/api.ts`

Generic `fetchApi<T>()` wrapper that:
1. Reads Supabase session token via `supabase.auth.getSession()`
2. Attaches `Authorization: Bearer <token>` header
3. Provides typed error handling via custom `ApiError` class

Two endpoint functions:
- `generateFloorPlan(prompt, model?, current_floorplan?)` → POST `/generate/floorplan`
- `extractFloorPlan(file_name, mime_type, file_data, model?)` → POST `/generate/extract-floorplan`

### 4.3 State Management — Zustand Stores

#### `useFloorPlanStore` (Application State)
| State | Type | Purpose |
|---|---|---|
| `floorPlan` | `FloorPlan \| null` | Currently active floor plan |
| `projectId` | `string \| null` | Persisted project ID |
| `messages` | `ChatMessage[]` | Chat history |
| `viewMode` | `'generate' \| 'view2d' \| 'view3d'` | Top-level UI mode |
| `viewerTab` | `'2d' \| '3d'` | Active canvas tab |
| `isLoading` | `boolean` | API call in progress |

Key action: `generatePlan(prompt, model)` — sends prompt to backend, updates floor plan state, saves to recent projects, switches to 2D view.

#### `useEditorStore` (Editor State)
Manages the 2D canvas editor with:
- **8 tools**: select, wall, room, door, window, text, eraser, furniture
- **Canvas transform**: zoom (default 40px/meter), panX, panY
- **Grid & snap**: 0.5m grid, toggleable snap-to-grid
- **Selection**: multi-select via Ctrl+Click or drag-select
- **Undo/Redo**: 50-entry history using `structuredClone()` for immutable snapshots
- **Floor plan mutations**: Pure functions that return new `FloorPlan` objects (immutable state pattern)

### 4.4 Type Definitions — `types/floorplan.ts`

Mirrors the backend Pydantic models exactly. Includes:
- **14 room types** with colour mapping (`ROOM_COLORS`)
- **4 furniture types**: table, chair, bed, cupboard (with default dimensions in meters)
- **Architectural elements**: Wall, Door, Window, Room, TextElement
- **API payloads**: `GenerateRequest`, `GenerateResponse`, `ChatMessage`

### 4.5 2D Canvas Renderer — `FloorPlanViewer2D.tsx` (946 lines)

A high-performance `<canvas>` component with:

#### Rendering Pipeline (60 FPS via `requestAnimationFrame`)
1. Clear canvas with dark background (`#0a0a0a`)
2. Apply transform: centre-origin, pan, zoom
3. Draw grid (if enabled)
4. Draw room fills (coloured rectangles with room-type colours)
5. Draw walls (thick lines, exterior vs interior distinction)
6. Draw doors (arc symbols with rotation from wall angle)
7. Draw windows (parallel-line symbols)
8. Draw furniture (table/chair/bed/cupboard symbols)
9. Draw text elements
10. Draw dimension lines on selected walls
11. Draw selection highlights
12. Draw in-progress tool previews (wall points, room rectangle, selection box)
13. Draw help text overlay

#### Hit Detection System
- **Walls**: Point-to-line-segment distance test (`hitTestWall`)
- **Rooms**: Point-in-rectangle test (`hitTestRect`)
- **Doors**: Point-in-circle test (`hitTestCircle`)
- **Priority**: Furniture → Doors → Windows → Texts → Walls → Rooms (smallest objects checked first)

#### Interaction Handling
- **Select tool**: Click to select, Ctrl+Click for multi-select, drag to move, drag-on-empty for box selection
- **Wall tool**: Click to place points, close loop near first point, right-click to finish open wall
- **Room tool**: Drag to draw rectangle
- **Door/Window tools**: Click on wall to snap-place
- **Furniture tool**: Click to place at cursor position

### 4.6 Design Assistant Chat — `DesignAssistant.tsx`

Chat interface component with:
- AI model selector dropdown (Claude 3.5 Haiku / Claude 4.6 Sonnet)
- Auto-scrolling chat history
- Suggested prompts for empty state
- Loading state with "Generating architectural intent..." message
- New Chat button to clear history

---

## 5. Archion-Sim Backend

### 5.1 Entry Point — `main.py` (783 lines)

A monolithic FastAPI application with **15+ endpoints** and in-memory state management using thread-safe locks:

| State Variable | Type | Purpose |
|---|---|---|
| `_sim_status` | `str` | Simulation status: idle/running/done/error |
| `_sim_trajectories` | `dict` | Agent position data per frame |
| `_compliance_report` | `dict` | Last compliance audit result |
| `_cached_geometry` | `dict` | Geometry from last model upload |
| `_analytics_data` | `dict` | Computed analytics metrics |
| `_sim_config` | `dict` | Custom role configurations |

**Why in-memory**: The simulation is stateless between sessions. No persistence needed — users upload a model, run a simulation, view results, and download a report in one session.

### 5.2 API Endpoints Overview

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/health` | GET | Health check |
| `/api/process-model` | POST | Upload 3D model (OBJ/GLB/GLTF), extract 2D geometry |
| `/api/compliance/init` | POST | Set building type for compliance rules |
| `/api/compliance/report` | GET | Get latest compliance audit report |
| `/api/ai-consultant` | POST | Get AI recommendation for a specific violation |
| `/api/simulation/configure` | POST | Configure roles (agent types, areas, colours) |
| `/api/simulation/start` | POST | Start batch simulation (background thread) |
| `/api/simulation/stream` | GET | Start SSE-streamed simulation (real-time) |
| `/api/get-trajectories` | GET | Get completed simulation trajectories |
| `/api/analytics` | GET | Compute and return analytics metrics |
| `/api/generate-report` | POST | Generate PDF compliance report |
| `/api/download-report/{filename}` | GET | Download generated PDF |
| `/api/heatmap-image` | GET | Generate and return heatmap PNG |
| `/api/simulation/marl` | POST | Run trained MARL agents (Actor-Critic inference) |

### 5.3 3D Geometry Extraction — `core/geometry.py` (238 lines)

Converts 3D building models into 2D navigation data:

```
OBJ/GLB/GLTF File
    ↓ trimesh.load()
3D Mesh
    ↓ _normalize_up_axis()  (Y-up → Z-up rotation for GLB/GLTF)
    ↓ Floor normalization (shift Z so bottom = 0)
Normalized Mesh
    ↓ _extract_wall_segments() (cross-section at Z=1.5m)
    ↓ Concave hull or wall-based footprint
ExtractionResult {
    boundaries,        # buffered exterior polygon
    raw_boundaries,    # exact exterior polygon
    obstacles,         # wall + furniture segments [x1,y1,x2,y2]
    center_offset,     # centroid for re-centering
    floor_area,        # m²
    floor_z,           # actual floor height
    mesh_vertices      # centered vertices for compliance analysis
}
```

**Two footprint extraction methods**:
1. **Wall-based** (preferred): Buffers extracted wall segments → unions → gets exterior → inward buffer for walkable area
2. **Point-cloud fallback**: Concave hull on floor-level vertices (if walls insufficient)

### 5.4 Simulation Engine — `sim/engine.py` (687 lines)

A **Multi-Agent Reinforcement Learning (MARL)-powered** pedestrian simulation engine.

#### Architecture: Three-Tier AI System

```
┌────────────────────────────────────────┐
│  TIER 1: Claude Strategic Brain        │
│  • Every 50 frames (5 seconds)         │
│  • Generates exploration waypoints     │
│  • Runs async in background thread     │
└────────────────┬───────────────────────┘
                 ↓ Waypoints
┌────────────────────────────────────────┐
│  TIER 2: MARL Motor Brain             │
│  • Every frame (100ms)                 │
│  • HTTP POST to brain_api (port 8001)  │
│  • Returns action: 0=Forward, 1=Left,  │
│    2=Right, 3=Interact                 │
└────────────────┬───────────────────────┘
                 ↓ Actions
┌────────────────────────────────────────┐
│  TIER 3: Claude Wall Override          │
│  • Triggered when ray_front < 1.2m     │
│  • Immediate heuristic (left/right)    │
│  • Claude called async for refinement  │
│  • 20-frame cooldown per agent         │
└────────────────────────────────────────┘
```

#### Simulation Constants
| Constant | Value | Meaning |
|---|---|---|
| `SIM_HZ` | 10 | Steps per second |
| `SIM_DURATION` | 60 | Seconds of simulation |
| `TOTAL_STEPS` | 600 | Total frames (60s × 10Hz) |
| `STEP_SIZE` | 0.09 | Metres per tick (~0.9 m/s walking speed) |
| `TURN_RATE` | 0.15 | Max radians heading change per tick |
| `WALL_MARGIN` | 0.15 | Buffer from boundary polygon |

#### Agent Sensor System (Ray Casting)
Each agent has 3 Shapely-based ray sensors:
- **Front ray** (0° from heading, 5m range)
- **Left ray** (+0.5 rad, 5m range)
- **Right ray** (−0.5 rad, 5m range)

Rays intersect with the walkable area boundary to measure distance to nearest wall.

#### Anti-Looping Mechanisms

1. **Path Locking**: Records visited grid cells (`LOCK_GRID_SIZE = 0.4m`). Cells become locked after agent moves 2m away. Prevents re-traversal. TTL of 200 frames (20 seconds).

2. **Cycle Detection**: Tracks position history over 40-frame windows. If displacement < 1.0m → agent is "stuck". Stuck agents are assigned a random far-away waypoint and forced to navigate there.

3. **Physical Unsticking**: Uses Shapely `nearest_points()` to push agents back inside the walkable area if they clip through boundaries.

#### Two Execution Modes

| Mode | Method | Use Case |
|---|---|---|
| **Batch** | `run()` | Pre-computes all 600 frames in a background thread |
| **Streaming** | `stream()` | Generator yielding frames with 50ms delay for real-time SSE |

### 5.5 Compliance Checking — `core/compliance.py` + `core/filtered_rules.py`

The `ComplianceChecker` class runs a full audit against Sri Lankan UDA regulations + ISO 21542:2011:

**Violation Types Checked**:
| Type | What It Checks | Standard |
|---|---|---|
| `corridor_width` | Minimum corridor width (building-type dependent) | UDA Section 3.2.1 / ISO 21542 Clause 13 |
| `door_width` | Minimum door clear opening | UDA Section 3.2.2 / ISO 21542 Clause 15 |
| `turning_space` | Wheelchair turning diameter at junctions | ISO 21542 Section 4.5 |
| `ramp_gradient` | Maximum ramp slope ratio | UDA Section 7.1 / ISO 21542 Clause 10 |
| `bottleneck` | Crowd density threshold (persons/m²) | UDA Emergency Egress Standards |

`filtered_rules.py` is a bridge that selects which rules apply based on building type (residential/office/hospital/educational/commercial/industrial).

### 5.6 Knowledge Base — `core/knowledge_base.py` (505 lines)

A comprehensive static knowledge base containing:

- **Regulations**: Per-violation-type, per-building-type minimum/recommended values with UDA section references
- **Safety Impact**: Classification of each violation type's danger level, affected populations, failure consequences
- **Cost Ranges (LKR)**: Minor/moderate/major repair cost estimates per violation type
- **Time Estimates**: Remediation timelines per violation type
- **Sri Lankan Construction Context**: Labour rates (LKR 3,500/day unskilled, 5,500/day skilled), common materials, regulatory bodies, approval process

### 5.7 AI Consultant — `core/ai_consultant.py` (423 lines)

A **hybrid AI system** that combines Claude LLM with local knowledge:

```
Violation
    ↓
Parameter Extractor (core/parameter_extractor.py)
    → Extracts spatial data from geometry + wall segments
    ↓
Knowledge Base Lookup
    → Regulatory context, safety impact, cost ranges
    ↓
Prompt Assembly
    → System prompt (senior Sri Lankan architect persona)
    → Violation data + spatial parameters + regulatory context
    → ML risk score injection (if available)
    ↓
Claude API Call
    → model: claude-3-haiku-20240307
    → temperature: 0.25 (deterministic)
    → max_tokens: 4096
    ↓
Validator (core/validator.py)
    → Checks required fields, normalises complexity
    → Cross-validates cost against knowledge base
    → Computes confidence score (0.1 – 1.0)
    ↓
Recommendation JSON {
    analysis, solution, implementation_steps,
    complexity, estimated_cost_lkr,
    regulation_reference, alternative_solutions,
    _confidence
}
```

**Fallback mechanism**: If Claude API fails (quota, timeout, invalid JSON), builds a complete recommendation from the knowledge base alone with `_confidence: 0.40` and `_is_fallback: true`.

**Caching**: Results are cached by violation ID to avoid redundant API calls.

**Batch processing**: `get_batch_compliance_advice()` processes multiple violations with 90-second timeout, sorted by severity (critical first).

### 5.8 ML Crowd Risk Predictor — `core/ml_predictor.py`

A **Random Forest regression model** (`archion_insight_model.pkl`) trained to predict crowd crush risk:

**Input Features**:
| Feature | Source |
|---|---|
| `total_agents` | Simulation summary |
| `floor_area_sqm` | Geometry extraction |
| `avg_velocity_ms` | Analytics computation |
| `peak_congestion_pct` | Analytics computation |

**Output**: Crowd Crush Risk Score (0–10)

**Integration**: The risk score is injected into Claude's prompt when generating compliance recommendations, influencing urgency assessment.

### 5.9 Analytics Engine — `core/analytics.py` (370 lines)

Computes 7 performance metrics from simulation trajectories:

| Metric | Method | Output |
|---|---|---|
| **Flow Rate** | Counts agents reaching boundary per 10s window | agents/minute timeline |
| **Congestion Index** | % of agent-frames where velocity < 0.2 m/s | percentage |
| **Efficiency Score** | Ratio of ideal path length to actual path length | 0–1 per agent |
| **Density Heatmap** | 2D histogram + Gaussian blur | Normalised grid (0–1) |
| **Congestion Timeline** | Congestion % per 5-second window | time series |
| **Velocity Timeline** | Average velocity per 1-second bucket | time series |
| **Summary** | Aggregate stats (total agents, duration, distances) | dict |

**Heatmap generation**: Uses `numpy.histogram2d()` with `scipy.ndimage.gaussian_filter()` for smooth density visualisation. Exported as PNG via Matplotlib.

### 5.10 Validation Engine — `core/validator.py`

Cross-validates AI-generated recommendations:

1. **Schema validation**: Ensures all 7 required fields exist with correct types
2. **Complexity normalisation**: Converts to `low`/`medium`/`high` or `unknown`
3. **Cost plausibility check**: Compares AI cost estimate against knowledge base range. Overrides if AI estimate is >2.5× above or <0.3× below the KB range
4. **Minimum step count**: Ensures at least 4 implementation steps (adds generic construction steps if needed)
5. **Analysis quality check**: Penalises responses with <80 character analysis
6. **Confidence scoring**: Deducts from a base of 1.0 for each quality issue. Final score: `max(0.1, min(1.0, score))`

### 5.11 PDF Report Generator — `core/report_gen.py` (1,068 lines)

Generates an **11-page professional PDF** using ReportLab:

| Page | Content |
|---|---|
| 1 | Cover page (title, compliance score badge, building metadata) |
| 2 | Executive summary (key metrics table, key findings bullets) |
| 3 | Building information (project name, type, area, simulation params) |
| 4 | Compliance analysis (violations table, bar chart) |
| 5 | AI-powered recommendations (per-violation analysis, solutions, steps) |
| 6 | Compliance breakdown (radar chart, severity pie chart, category table) |
| 7 | Recommendations summary (condensed table, priority action items) |
| 8 | Performance metrics (velocity timeline, flow rate chart) |
| 9 | Density heatmap (full-page heatmap image) |
| 10 | Conclusion (overall assessment, next steps) |
| 11 | Appendix (methodology, standards referenced) |

**Charts generated**: Matplotlib figures converted to ReportLab `Image` objects via in-memory `BytesIO` buffers.

### 5.12 MARL Training — `marl/` directory

Optional Actor-Critic MARL training system:
- `gym_environment.py`: Custom Gymnasium environment wrapping the building floor plan
- `networks.py`: PyTorch Actor-Critic neural network (A2C)
- `train.py`: Training loop

**Brain API** (`sim/brain_api.py`): FastAPI server on port 8001 that loads trained models and provides `/act_batch` endpoint for inference during simulation.

### 5.13 Schemas — `schemas.py`

Pydantic models for API serialisation:
- `AgentPosition` (id, x, y, type)
- `Obstacle` (points list)
- `SimulationFrame` (frame_id, data)
- `Violation` (id, type, severity, coordinate, measured/required values, description, regulation)
- `ComplianceReport` (standard, building_type, total_violations, violations list, compliance_score, status, summary)

---

## 6. Archion-Sim Frontend

### 6.1 Key Components

- **ModelRenderer**: Three.js-based 3D model viewer that loads uploaded OBJ/GLB/GLTF files
- **SimViewer**: Replays simulation trajectories as animated dots on the 2D floor plan
- **AnalyticsDashboard**: Visualises analytics data (charts, heatmaps, statistics)
- **CompliancePanel**: Displays compliance violations with severity badges, AI consultant recommendations

### 6.2 SSE Integration

For real-time simulation streaming:
```typescript
const eventSource = new EventSource('/api/simulation/stream?n_standard=2');
eventSource.addEventListener('frame', (e) => {
    const { frame, agents } = JSON.parse(e.data);
    // Update agent positions in Three.js scene
});
eventSource.addEventListener('done', () => eventSource.close());
```

---

## 7. Infrastructure & Deployment

### Docker Compose Setup
```yaml
services:
  archion-build-backend:
    build: ./archion-build/backend
    ports: ["8000:8000"]
  archion-sim-backend:
    build: ./archion-sim/backend
    ports: ["8002:8000"]
  nginx:
    image: nginx
    ports: ["80:80", "443:443"]
```

### Nginx Reverse Proxy
| Domain | Backend |
|---|---|
| `api.archionlabs.com/build/` | → `archion-build-backend:8000` |
| `api.archionlabs.com/sim/` | → `archion-sim-backend:8000` |

### SSL: Let's Encrypt (Certbot) for HTTPS on API domain.

### Frontend Deployment: Vercel with `NEXT_PUBLIC_API_URL` environment variable pointing to the backend API.

---

## 8. Key Design Decisions & Interview Talking Points

### 8.1 Why Hybrid AI (LLM + Algorithmic)?
**Archion-Build** uses Claude for understanding natural language intent but relies on an **algorithmic strip-packing solver** for actual room layout. This gives deterministic, geometrically valid results while still allowing natural language input. The LLM handles the "what" (room requirements), the algorithm handles the "how" (spatial arrangement).

### 8.2 Why Three-Tier AI in Simulation?
The simulation combines three AI layers because each excels at a different timescale:
- **Claude** (slow, 5s intervals): High-level strategic planning (exploration waypoints)
- **MARL Brain** (fast, every frame): Learned motor control (navigation actions)
- **Heuristic + Claude Wall Override**: Immediate obstacle avoidance

This mirrors real-world autonomous systems where different controllers operate at different frequencies.

### 8.3 Why SSE for Simulation Streaming?
Server-Sent Events (SSE) was chosen over WebSockets because:
- **Unidirectional**: Simulation only needs server → client data flow
- **Auto-reconnection**: SSE reconnects automatically on network drops
- **Simpler**: No need for WebSocket handshake or keep-alive management
- **HTTP-compatible**: Works through standard HTTP proxies and load balancers

### 8.4 Why Immutable State in the Editor?
The `useEditorStore` uses **pure functions** that return new `FloorPlan` objects instead of mutating state. Benefits:
- Enables undo/redo with `structuredClone()` snapshots
- Prevents UI rendering bugs from shared mutable references
- Allows React's reference equality checks for efficient re-renders

### 8.5 Why Knowledge Base + LLM (Hybrid AI Consultant)?
The AI Consultant doesn't rely solely on Claude. It:
1. Uses a **domain-specific knowledge base** for Sri Lankan construction standards
2. Uses a **validator** to catch implausible LLM outputs (cost sanity checks)
3. Uses a **fallback system** that works entirely offline when the LLM is unavailable
4. Injects **ML predictions** (crowd risk score) into the LLM prompt for data-driven assessments

This "trust but verify" approach ensures reliable recommendations even when the LLM hallucinates.

### 8.6 Why Canvas over SVG for Floor Plan Rendering?
HTML5 Canvas was chosen for the 2D editor because:
- **Performance**: Direct pixel manipulation scales better with thousands of elements
- **Custom rendering**: Full control over architectural symbol drawing (arcs for doors, parallel lines for windows)
- **Zoom/Pan**: Simple affine transform matrix application
- **Real-time**: 60 FPS render loop with `requestAnimationFrame`

### 8.7 Why Supabase for Auth?
- **Managed PostgreSQL**: No database administration overhead
- **Built-in JWT**: Standard JWT tokens that can be verified by any backend
- **Row-Level Security**: Database-level access control
- **Free tier**: Suitable for a university project with < 100K monthly active users

### 8.8 Why No Database in Archion-Sim?
The simulation is **session-scoped**: users upload a model, run a simulation, and download results in one sitting. Persistent storage would add complexity without value. The in-memory state with thread-safe locks is sufficient and much simpler.

### 8.9 Why Random Forest for Crowd Risk Prediction?
- **Interpretable**: Feature importances show which factors contribute most to risk
- **Small model**: The `.pkl` file is lightweight enough for real-time inference
- **Tabular data**: Random Forests excel on structured tabular features (agent count, floor area, velocity, congestion)
- **No deep learning overhead**: No GPU required for inference

### 8.10 Why Generate PDF Reports Server-Side?
ReportLab generates pixel-perfect PDFs with embedded charts on the server, ensuring consistent output regardless of browser. This is critical for compliance documentation that may be submitted to regulatory authorities.
