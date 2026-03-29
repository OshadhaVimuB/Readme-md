# Archion-Build: Comprehensive Codebase Documentation

Welcome to the comprehensive documentation for **Archion-Build**, an AI-powered architectural layout and 3D modeling generator. This document is designed to be beginner-friendly while providing deep technical insights into what every major file in the codebase does, the underlying logic, and key code snippets.

---

## 1. Project Overview

**Archion-Build** allows users to generate, edit, and visualize 2D and 3D floor plans using natural language prompts (AI) or manual drawing tools. 
The system is divided into two main parts:
1. **Backend (`/backend`)**: Built with **FastAPI** (Python). It handles LLM integration (Anthropic Claude/Groq) for intent parsing, a geometry engine for generating floor plans, and Supabase connectivty for data storage.
2. **Frontend (`/frontend`)**: Built with **Next.js** (React) and **Zustand**. It provides a 2D HTML5 Canvas for interactive floor plan editing and a 3D viewer using `@react-three/fiber`.

---

## 2. Backend Documentation (`/backend`)

The backend is responsible for processing AI prompts, running geometric layout algorithms, and saving designs to the database.

### 2.1. Core Configuration & Setup

#### `app/main.py`
**Purpose:** This is the entry point for the FastAPI application.
**Logic:** It initializes the backend server, sets up Cross-Origin Resource Sharing (CORS) so the frontend can securely communicate with it, connects to the database, and registers API routers.

#### `app/config.py` & `app/auth.py`
**Purpose:** Configuration and Security.
**Logic:** 
- `config.py` loads environment variables (API keys for Claude, Groq, and Supabase).
- `auth.py` implements Supabase JWT verification. It verifies user tokens to ensure only authenticated users can save projects, though it also provides "optional" auth for public generation.

### 2.2. Database and Data Models

#### `app/database.py` & `app/models/db_models.py`
**Purpose:** Database connection and ORM (Object-Relational Mapping).
**Logic:** Uses **SQLAlchemy** to talk to the database (SQLite for local testing, PostgreSQL for production via Supabase). `db_models.py` defines the SQL tables (e.g., `projects` table which stores the serialized JSON of the generated floor plans, and `chat_history`).

#### `app/models/floorplan.py`
**Purpose:** Defines the standard data structure for architectural elements.
**Logic:** Uses **Pydantic** to precisely define what a `Room`, `Wall`, `Door`, `Window`, and `FloorPlan` look like in code. This ensures strict type-checking across the entire backend.
> **Snippet Highlight (Pydantic Model):**
```python
class Wall(BaseModel):
    id: str = Field(default_factory=lambda: f"wall_{uuid.uuid4().hex[:8]}")
    start: Point2D
    end: Point2D
    thickness: float = 0.15
    is_exterior: bool = False
```

### 2.3. AI Intent Parsing & Routing

#### `app/routers/generate.py`
**Purpose:** The main controller for AI generation.
**Logic:** Receives the `/api/v1/generate/floorplan` HTTP requests from the frontend. It passes the user's text prompt to the Intent Parser, runs those parsed requirements through the Layout Solver, saves the resulting geometry to the database, and syncs it with the Supabase dashboard.

#### `app/services/claude_intent.py` & `app/services/groq_intent.py`
**Purpose:** Connects to LLMs (Large Language Models) to figure out what the user wants.
**Logic:** 
- `claude_intent.py` is the primary AI engine. It contains an `IntentParser` that can read text prompts OR take uploaded images/blueprints to extract structural requirements (e.g., "3 bedrooms, 1 bathroom"). It forces the LLM to reply with a structured JSON output representing the rooms needed.
- `groq_intent.py` acts as an alternative/fallback AI using the Groq API.

### 2.4. Geometry & 3D Engines

#### `app/services/geometry.py`
**Purpose:** The algorithmic heart of the backend—converting abstract room requirements into concrete shapes and coordinate points.
**Logic:** Contains the `LayoutSolver` class. It uses a **strip-packing algorithm**. It takes a list of required rooms, creates 2D rectangles for each, and mathematically packs them together into a coherent building footprint.
- **Wall Generation:** Once rooms are packed, it mathematically calculates the perimeters to generate connecting `Wall` objects.
- **Door/Window Placement:** Evaluates neighboring walls between rooms to safely place doors (e.g., ensuring a bedroom connects to a hallway).

#### `app/services/model3d.py`
**Purpose:** Translating 2D lines into 3D models.
**Logic:** Takes the solved `FloorPlan` and generates representations suitable for 3D modeling programs. It can export to **Three.js** JSON format or create a **CadQuery** Python script for parametric CAD modeling.

---

## 3. Frontend Documentation (`/frontend`)

The frontend is a highly interactive application that lets users draw, edit, and talk to the AI to construct their structures.

### 3.1. Main Layout & API

#### `src/app/page.tsx` & `src/components/FloorPlanEditor.tsx`
**Purpose:** the main UI page structure.
**Logic:** `FloorPlanEditor.tsx` acts as the layout shell. It splits the screen into three zones:
1. Left panel: Room Specifications (properties of selected items).
2. Center panel: The main canvas for 2D/3D viewing and the top Toolbar.
3. Right panel: `DesignAssistant` (The AI chat interface).

#### `src/services/api.ts`
**Purpose:** Fetches data from the backend.
**Logic:** A custom fetch wrapper that adds the Supabase JWT token into the `Authorization` header and makes calls (like `generateFloorPlan`) to the Python FastAPI backend.

### 3.2. State Management (Zustand)

#### `src/store/useFloorPlanStore.ts`
**Purpose:** Manages global application data.
**Logic:** Uses **Zustand** (a lightweight Redux alternative). It stores the currently loaded `FloorPlan`, the chat `messages`, and the `isLoading` state. It includes the `generatePlan` action which fires off the API call and automatically handles displaying the AI's response in the chat sidebar.
> **Snippet Highlight (Zustand Store):**
```typescript
generatePlan: async (prompt: string, model?: string) => {
    // 1. Add user prompt to chat
    set((state) => ({ messages: [...state.messages, userMessage], isLoading: true }));
    try {
        // 2. Call backend
        const response = await generateFloorPlan(prompt, model, get().floorPlan);
        // 3. Update floor plan canvas and add AI response
        set((state) => ({
            floorPlan: response.floorplan,
            messages: [...state.messages, assistantMessage],
            viewMode: "view2d",
            isLoading: false,
        }));
    } catch (err) { /* error handling */ }
}
```

#### `src/store/useEditorStore.ts`
**Purpose:** Manages interactive canvas state.
**Logic:** Keeps track of what tool the user is holding (e.g., `wall`, `door`, `eraser`, `select`), zoom/pan translations, selected item IDs, and grid snapping states.

### 3.3. Interactive Visualization

#### `src/components/FloorPlanViewer2D.tsx`
**Purpose:** The 2D drafting engine.
**Logic:** This is a massive, complex component that directly utilizes the HTML5 `<canvas>` API via a `requestAnimationFrame` loop. 
- **Rendering:** It iterates over the `FloorPlan` object, mathematically calculating where to draw walls, doors, and furniture based on current zoom and pan coordinates (`screenToWorld` conversions).
- **Interactions:** Implements mouse listeners (`handleMouseDown`, `handleMouseMove`) to detect if a cursor is hovering over a wall, snapping to grid points, and drawing new walls dynamically.

#### `src/components/Viewer3D.tsx`
**Purpose:** The 3D exploration view.
**Logic:** Uses `@react-three/fiber` to create a 3D WebGL context in an declarative, React-friendly way. 
- Mathematically computes the length of walls and uses `THREE.Shape` geometry along with `.holes` to cut out spaces for doors and windows in the 3D meshes.
- Automatically creates basic 3D furniture models based on properties (bed, chair, table).
- Implements `GLTFExporter` functionality to allow users to download their 3D model as a `.glb` file.

#### `src/components/DesignAssistant.tsx`
**Purpose:** The Chat Sidebar UI.
**Logic:** A simple messaging interface built with `shadcn/ui` components. It maps over `messages` from the Zustand store. Submitting a prompt triggers `generatePlan` in the store, spinning a loading icon while waiting for the FastAPI backend to do the intense geometric calculations.

---

## 4. Key Takeaways & Architecture Flow

When a user typed *"Make a 2 bedroom apartment"* into the Design Assistant, the following flow occurs:

1. **Frontend (`DesignAssistant.tsx`)** dispatches the text to the **Zustand store** (`useFloorPlanStore.ts`).
2. The store calls the **API** (`api.ts`), sending a POST request securely with an Auth token.
3. The **FastAPI Backend** (`generate.py`) receives the request and asks the LLM via `claude_intent.py` to extract structured rooms from the text.
4. The requested rooms are passed to the **Math Engine** (`geometry.py`), which strip-packs them into an apartment footprint and traces walls and doors.
5. The backend saves the generated JSON to **Supabase** via `db_models.py` and returns it.
6. The frontend store receives the JSON, updates itself, and tells `FloorPlanViewer2D.tsx` and `Viewer3D.tsx` to instantly render the gorgeous new architecture on the screen.
