# HoloSwarm

### A Self-Correcting Multi-Agent Architecture for Gesture-Controlled Holographic Interfaces
[![WeaveHacks 2026](https://img.shields.io/badge/Built%20at-WeaveHacks%202026-blueviolet?style=for-the-badge)](https://github.com/)
[![Hackathon Project](https://img.shields.io/badge/Hackathon-48--Hour-orange?style=for-the-badge)](https://github.com/)
[![Tech Stack](https://img.shields.io/badge/Stack-Node.js%20%7C%20Three.js%20%7C%20Redis%20%7C%20MediaPipe-brightgreen?style=for-the-badge)](https://github.com/)

HoloSwarm is a camera-only, gesture-controlled 3D interface driven by a coordinated swarm of five specialized agents. A dedicated **Critic** agent audits the system state in real time, detecting anomalies and publishing correction vectors dynamically over Redis pub/sub to achieve self-correcting loop stability.

## 1. Motivation

Traditional gesture-controlled interfaces require specialized hardware: depth cameras, sensory gloves, or infrared emitters. **HoloSwarm** answers a simpler question:

> *Can a standard laptop webcam plus a swarm of specialized agents deliver a responsive, holographic-style interface—and can the system dynamically correct its own tracking and classification failures during runtime?*
Single-model pipelines typically fail silently. If hand tracking drifts or an intent is misclassified, errors propagate directly to the render layer, causing glitchy and erratic visual behavior. HoloSwarm solves this architecturally by introducing a dedicated **Critic** agent that audits the intermediate streams and feeds corrections back into the message loop recursively.

## 2. Architecture

The system is composed of five specialized agents running as independent processes and communicating asynchronously over **Redis pub/sub**.

mermaid
flowchart TD
    Webcam[ Webcam] -->|Video Stream| Perception[Perception Agent]
    Perception -->|Normalized Landmarks| Intent[ Intent Agent]
    Intent -->|Gesture Intents| Scene[ Scene Agent]
    Scene -->|Scene Graph Mutations| Render[ Render Agent]
    Render -->|Frame Updates| Screen[ Holographic Display]

    %% Critic feedback loops
    Critic[ Critic Agent] -.->|Anomalies Detected| Critic
    Critic -.->|Correction: Re-anchor| Perception
    Critic -.->|Correction: Hold Intent| Intent
    Critic -.->|Correction: Bound Mutations| Scene

    %% Critic auditing lines
    Perception -.->|Audits landmarks| Critic
    Intent -.->|Audits gestures| Critic
    Scene -.->|Audits mutations| Critic

    style Critic fill:#ff3366,stroke:#333,stroke-width:2px,color:#fff
    style Webcam fill:#444,stroke:#333,color:#fff
    style Screen fill:#444,stroke:#333,color:#fff
    style Perception fill:#00ccff,stroke:#333,color:#000
    style Intent fill:#00ccff,stroke:#333,color:#000
    style Scene fill:#00ccff,stroke:#333,color:#000
    style Render fill:#00ccff,stroke:#333,color:#000


### Swarm Roles & Responsibilities

| Agent | Responsibility | Data Handled |
| :--- | :--- | :--- |
| ** Perception** | Normalizes webcam video feeds into 21-point hand skeletons frame-by-frame. | MediaPipe Hands Landmark Streams |
| **Intent** | Classifies gesture sequences into structural commands (*grab*, *rotate*, *swipe*, *pinch-zoom*). | Gesture Intent Vectors |
| **Scene** | Manages the 3D scene state and translates gesture commands into coordinates/rotations. | Three.js Scene-Graph Mutations |
| **Render** | Drives the WebGL render loop, drawing real-time hover glows, grab highlights, and 3D assets. | In-Browser WebGL Canvas |
| **Critic** | Audits landmark stability, intent transitions, and scene mutations, injecting corrections back into the swarm. | Multi-Channel Audit Logs & Redis Channels |

### The Redis Pub/Sub Advantage

Using Redis pub/sub provides two critical architectural benefits:
1. **Decoupled Fan-Out:** The Critic agent can listen to all streams transparently without the other agents knowing they are being monitored.
2. **Homogeneous Correction Pathway:** The Critic publishes correction frames to the *same* Redis channels that downstream agents are already subscribing to. This ensures corrections flow naturally alongside normal messages, requiring zero custom logic or special-case bypass code in the receiving agents.

### The Critic Loop (Core Innovation)

The Critic maintains a rolling sliding-window buffer of output frames from each agent and executes rule-based validation checks:

* **Temporal Plausibility:** Hand landmarks that jump instantly across the coordinate space indicate tracking loss. The Critic instructs the **Perception** agent to re-anchor rather than forwarding garbage data downstream.
* **Intent Stability:** High-frequency oscillation between gestures (e.g., *grab -> release -> grab -> release* in $< 200\text{ ms}$) indicates classification jitter. The Critic suppresses the oscillation and holds the last stable state.
* **Scene-State Sanity:** Proposed 3D object mutations that would translate items outside the viewport boundaries are intercepted and rejected before hitting the **Render** engine.

> **Graceful Degradation:** When tracking fails, HoloSwarm holds its last known stable state instead of rendering visual jitter, delivering a smoother user experience.


## 3. Technology Stack

* **Hand Tracking:** MediaPipe Hands (Camera-only 21-landmark tracking, no specialized depth sensors needed)
* **3D Engine:** Three.js (WebGL rendering)
* **Message Broker:** Redis (Asynchronous pub/sub orchestration & correction pipelines)
* **Runtime:** Node.js (For backend agent processes) & Browser (For the Render layer)
## 4. Retrospective (Hackathon Lessons)

### What Worked 
* **The Unified Correction Channel:** Routing corrections through the existing pub/sub channels allowed us to add self-correction features without complicating the individual agent code. Agents process Critic messages seamlessly as if they were standard upstream events.

### What Didn't (Yet) 
* **Intent Classification Jitter:** In poor lighting conditions, MediaPipe landmark jitter increases, triggering misclassifications in the Intent agent. While the Critic successfully masks these visual glitches, it acts as a filter rather than a solution. We need a confidence-weighted gesture classifier.

> [!NOTE]
> *HoloSwarm was developed during a 8-hour hackathon. The performance observations above are qualitative and based on live demonstration testing. Structured latency and accuracy benchmarking are planned.*

## 5. Roadmap

- [ ] **Phase 1: Instrumentation**
  - Track end-to-end latency (from webcam frame capture to Render mutation output) and publish performance benchmarks.
- [ ] **Phase 2: Classifier Improvements**
  - Implement a confidence-weighted intent classifier with per-gesture thresholds.
- [ ] **Phase 3: Multi-Hand Interaction**
  - Enable bimanual gesture controls (e.g., simultaneous rotation and scaling).
- [ ] **Phase 4: ML-Powered Critic**
  - Replace the Critic's rule-based validation script with a learned anomaly detection model, closing the recursive self-improvement loop.

## 6. Why This Architecture Generalizes

The Critic design pattern is not restricted to computer vision or gesture recognition. It solves a fundamental problem in any multi-agent pipeline where upstream errors propagate silently:

```
Input -> [Agent A] -> [Agent B] -> [Agent C] -> Output
            ^            ^            ^
            |            |            |
            +------ [Critic Agent] ---+
```

Examples where this architecture fits perfectly:
- **Retrieval-Augmented Generation (RAG):** Retrieval $\rightarrow$ Extraction $\rightarrow$ Synthesis pipelines, where a Critic agent audits intermediate context chunks and intercepts hallucinations.
- **Autonomous Systems:** Robotics perception-action loops, where a Critic monitors sensory drift and applies corrections.
- **General Agent Swarms:** Any sequential process where a supervisor needs to dynamically intercept, mutate, or reject signals without changing the core agent codebases.
