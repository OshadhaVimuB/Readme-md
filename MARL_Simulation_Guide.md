# ArchionLabs: Pedestrian Simulation & MARL Implementation Guide

Welcome! If you're a beginner reading this, this guide will walk you through exactly how the **pedestrian simulation** and **MARL (Multi-Agent Reinforcement Learning)** are built in the ArchionLabs codebase.

At its core, the project asks: **"If we drop a lot of people (agents) into a 2D floor plan, how do they behave, navigate, and get around obstacles?"**

To solve this, ArchionLabs uses a **hybrid AI approach**, combining a fast motor-control neural network (MARL) with an intelligent, strategic "thinker" (Claude LLM). Let's break down exactly how this works under the hood.

---

## 1. The Core Components

The simulation lives inside `archion-sim/backend/sim/`. There are three main files that do all the heavy lifting:

1. **`engine.py` (The Conductor)** 
   The heart of the simulation. It keeps track of time (which happens in "frames"—10 times a second), the floor plan walls (boundaries and obstacles), and where each agent is currently standing. It handles physics, like checking if an agent bumped into a wall.
   
2. **`brain_api.py` (The Reflexes / Motor Control)** 
   This is a FastAPI server running a pre-trained PyTorch neural network model (`archion_marl_brain_v1.pth`). 
   Think of this as the agent's spinal cord and fast reflexes. It receives raw sensory data (like "wall 1 meter ahead") and instantly outputs a basic physical action (e.g., "turn left," "move forward").
   
3. **`llm_brain.py` / Claude Integration (The Strategist)** 
   While the neural network handles the immediate walking, Claude (an AI Large Language Model) handles the long-term planning. It tells agents *"Hey, instead of wandering around here, go explore that coordinate on the other side of the building."*

---

## 2. How the Agents "See" the World (Virtual Sensors)

In `engine.py`, agents don't have eyes; they have virtual laser pointers (Raycasting). 

Every frame, the simulation shoots out **3 invisible rays** from each agent:
1. One pointing straight forward.
2. One pointing slightly to the left.
3. One pointing slightly to the right.

The engine calculates how far each ray travels before hitting a wall. This provides three numbers: `[ray_front, ray_left, ray_right]`.

The agent also tracks the **angle towards its destination**. Together, these numbers form the agent's "State." This State is sent to the MARL Brain (`brain_api.py`) to decide the next move.

---

## 3. The AI Decision Loop

The simulation loop in `engine.py` executes 10 times per second. Here is the step-by-step logic it follows:

### Step A: The Strategic LLM Check (Every 50 frames)
Every 5 seconds in simulation time, Claude is provided with the boundaries of the building and the current locations of all agents. It acts as an orchestrator and assigns a high-level **Waypoint** (`[x, y]` coordinates) for each agent to walk towards. This ensures the agents explore the entire building rather than roaming randomly.

### Step B: The MARL Reflexes (Every single frame)
1. The engine collects the sensor data (the 3 rays) and the angle to the current waypoint for **all agents**.
2. It sends this data in a batch to `brain_api.py`.
3. The MARL neural network instantly returns a motor action for every agent. 
   - `0`: Move Forward
   - `1`: Turn Left
   - `2`: Turn Right
   - `3`: Stop / Interact

### Step C: Movement and Physics (Every single frame)
The engine translates the AI's action into math. If the action is `1` (Turn Left), the engine alters the agent's heading angle. If the action is `0` (Move Forward), the engine moves the agent's X and Y coordinates forward by a tiny step. 
Finally, it uses a geometry library (Shapely) to ensure the new coordinates aren't inside a wall. If they are, the agent is pushed back into the walkable area.

---

## 4. Handling Tricky Situations (Heuristics)

Sometimes, AI gets confused. The engine has several built-in safety nets to keep the simulation looking surprisingly realistic:

* **Stuck Mode (Cycle Detection):** 
  The engine continuously records the last 40 frames of an agent's path. If it notices the agent hasn't moved more than 1 meter in that time (meaning it's walking in tiny circles or stuck in a corner), it triggers a "Stuck Escape." The engine temporarily takes control away from the MARL brain, finds a random open space far away, and forces the agent to walk perfectly straight there to break the cycle.
  
* **Doorway Crowding (Path Locking):** 
  If 50 agents all decide to go through the same 1-meter door at the same time, they'll overlap. The engine implements a "lock" system. Once an agent steps into a grid cell, it "locks" that cell. Other agents are preventing from stepping on that grid until the first agent moves away. This forces a realistic, single-file line through narrow bottlenecks!

* **The "Claude Wall" Override:** 
  If an agent is moving toward a wall and gets too close (less than 1.2 meters), it sometimes freezes or makes bad choices. In this case, the engine fires off a quick SOS prompt to Claude: *"Agent is 1m from a wall. Left is clear 5m, right is clear 2m. Should it turn left or right?"* Claude responds, and the engine executes the turn immediately to avoid a collision.

---

## Technical Summary
By separating **Macro-Strategy** (Claude LLM) from **Micro-Movement** (PyTorch Neural Net), the ArchionLabs simulation achieves fast, realistic pedestrian flow. The heavy math (raycasting, wall collisions, cycle detection) is handled locally in Python, meaning thousands of steps can happen rapidly while visually streaming to the frontend dashboard.
