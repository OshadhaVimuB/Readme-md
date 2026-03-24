# Archion Build: NLP (Natural Language Processing) Implementation Guide

Welcome! This beginner-friendly guide explains how **Archion Build** turns natural human language (like *"I want a 3-bedroom house"*) into a fully structured 3D floor plan.

At its core, the project uses **Anthropic's Claude** as the "translator." Instead of generating code or text for the user to read, the system asks Claude to generate **JSON** (a formatted data structure) that our 3D engine can understand.

---

## 1. The Core Components

The NLP logic lives inside the `archion-build` backend directory, specifically in `archion-build/backend/app/`.

1. **`routers/generate.py` (The API Endpoint)** 
   This is the front door. When the frontend (the chat dashboard) sends a text prompt or an image, it arrives here. This file figures out which AI model to use and manages saving the resulting floor plan to the database.
   
2. **`services/claude_intent.py` (The AI Brain)** 
   This is the heart of the NLP system. It contains the `IntentParser` class, which connects to the Anthropic API (Claude) and handles the actual "translation" between human words and floor plan coordinates.

---

## 2. How the AI Understands Requests (The 4 Pipelines)

Depending on what the user asks for, the NLP system uses one of four different pipelines.

### Pipeline A: Direct Text-to-Floorplan (The Default)
If a user types: *"Build me a modern apartment with a large balcony."*
1. **The System Prompt:** The backend wraps this text in a massive, strict set of instructions called the `FULL_GENERATION_PROMPT`. It tells Claude: *"You are an architect. Output ONLY valid JSON. All walls must be horizontal or vertical. Use real-world meters from 0 to 25m."*
2. **The Output:** Claude uses spatial reasoning to literally write out the X and Y coordinates for every room, wall, door, and window.

### Pipeline B: The Fallback Pipeline
If Claude gets confused or a weaker AI model is used, generating exact X and Y coordinates can fail. In this case, the system has a safety net:
1. **Keyword Extraction:** The AI simply extracts the *intent* as a list: `[1x Living Room, 1x Kitchen, 2x Bedroom]`. 
2. **The Layout Solver:** The backend hands this simple list over to a traditional math engine (`LayoutSolver` in `geometry.py`), which geometrically arranges the rooms into a physical shape without using AI.

### Pipeline C: Modifying an Existing Plan
If a user already has a generated floor plan and types: *"Add a bathroom next to the master bedroom."*
1. This is the hardest task for an AI. The backend sends Claude **both** the text prompt **AND** the entire current Floor Plan JSON.
2. Claude reads the current coordinates, finds the master bedroom, calculates where to place the new walls for the bathroom, and returns the modified JSON.

### Pipeline D: Vision Extraction (Image-to-Floorplan)
If the user uploads an image/PDF/DXF of an old 2D blueprint:
1. The backend uses **Claude Vision**.
2. The AI visually "looks" at the blueprint image, identifies where the lines and rooms are, estimates real-world sizes (knowing a standard door is ~0.9m), and outputs the exact coordinates as JSON.

---

## 3. The Auto-Repair System (Safety Nets)

AI is incredibly smart, but it can make silly spatial mistakes—like forgetting to put a door between a hallway and a bedroom. 

To fix this, `claude_intent.py` includes a crucial function called `_validate_and_repair`. Every time Claude generates a floor plan, the system runs it through a "car wash" of geometric checks before saving it:

1. **Missing Doors/Windows:** If two rooms touch (share a logical bounding box edge) but there is no door connecting them, the system automatically calculates the midpoint of the wall and adds a door.
2. **Illegal Room Types:** If the AI makes up a room name like `"master_bedroom"`, the system corrects it to standard types like `"bedroom"` so the 3D renderer doesn't crash.
3. **Wall Generation:** The AI places "bounding boxes" (rectangles) for rooms. The engine automatically traces the edges of these boxes to generate the physical walls and detects which walls are "exterior" walls.

---

## Technical Summary
By treating Large Language Models as **JSON Coordinate Generators** rather than text chatbots, Archion Build achieves impressive generative design. The heavy lifting is done by tightly engineered "System Prompts" that force the AI to adhere to strict architectural rules, combined with a robust post-processing Python script that fixes any geometric errors before the user ever sees them.
