# Self-Healing CI/CD Pipeline with Agentic AI — Final Implementation Plan

## Table of Contents
1. [Project Overview](#1-project-overview)
2. [System Architecture](#2-system-architecture)
3. [Agent Details](#3-agent-details)
4. [Agent Brainstorming Mechanism](#4-agent-brainstorming-mechanism)
5. [Complete Data Flow](#5-complete-data-flow)
6. [Tech Stack](#6-tech-stack)
7. [Project File Structure](#7-project-file-structure)
8. [Step-by-Step Implementation Guide](#8-step-by-step-implementation-guide)
9. [Code Architecture Per File](#9-code-architecture-per-file)
10. [Demo Plan](#10-demo-plan)
11. [Timeline](#11-timeline)
12. [Future Roadmap](#12-future-roadmap)

---

## 1. Project Overview

### One-Line Pitch
> When a Jenkins build fails, AI agents autonomously detect the failure, brainstorm together to diagnose the root cause, and send a Slack notification with the fix — all in under 10 seconds.

### Hackathon Category
**Delete, Delegate, Automate** — Automating the most time-consuming daily DevOps task: debugging failed builds.

### Problem Statement
```
Developers spend 30-60 minutes per build failure:
  1. Open Jenkins → find failed build
  2. Scroll through 1,000+ log lines
  3. Find the error (if lucky)
  4. Google the fix
  5. Apply fix → rerun build → hope it works
  6. Repeat weekly for the same errors

Result: Hundreds of engineering hours wasted per month
```

### Solution
```
Build fails → AI agents detect in 1 second → 
Orchestrator classifies error → 
Specialized agents analyze in parallel → 
Agents brainstorm (share findings, refine) → 
Slack notification with root cause + fix → 
Developer applies fix in 1 minute

Result: 93% reduction in debugging time
```

---

## 2. System Architecture

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DOCKER COMPOSE ENVIRONMENT                        │
│                                                                      │
│  ┌──────────────────────┐        ┌────────────────────────────────┐ │
│  │                      │        │                                │ │
│  │   JENKINS SERVER     │ webhook│   SELF-HEALING ENGINE          │ │
│  │   (Docker)           │───────▶│   (Python FastAPI)             │ │
│  │                      │        │                                │ │
│  │   Port: 8080         │◀───────│   Port: 5000                  │ │
│  │                      │ API    │                                │ │
│  │   • Build Pipelines  │        │   ┌────────────────────────┐  │ │
│  │   • Console Logs     │        │   │   DETECTION AGENT 🔍   │  │ │
│  │   • Webhooks         │        │   │   (Webhook Listener)    │  │ │
│  │                      │        │   └──────────┬─────────────┘  │ │
│  └──────────────────────┘        │              │                │ │
│                                  │              ▼                │ │
│                                  │   ┌────────────────────────┐  │ │
│                                  │   │  ORCHESTRATOR AGENT 🧠 │  │ │
│                                  │   │  (The Brain)            │  │ │
│                                  │   │                         │  │ │
│                                  │   │  • Classifies error     │  │ │
│                                  │   │  • Selects agents       │  │ │
│                                  │   │  • Runs Round 1         │  │ │
│                                  │   │  • Builds shared ctx    │  │ │
│                                  │   │  • Runs Round 2         │  │ │
│                                  │   │  • Synthesizes result   │  │ │
│                                  │   └──┬──┬──┬──────────────┘  │ │
│                                  │      │  │  │                  │ │
│                                  │      ▼  ▼  ▼                  │ │
│                                  │   ┌──────────────────────┐    │ │
│                                  │   │  SPECIALIZED AGENTS   │    │ │
│                                  │   │                       │    │ │
│                                  │   │  📋 Diagnosis Agent   │    │ │
│                                  │   │  💻 Code Agent        │    │ │
│                                  │   │  🧪 Test Agent        │    │ │
│                                  │   │  📦 Dependency Agent  │    │ │
│                                  │   └──────────┬───────────┘    │ │
│                                  │              │                │ │
│                                  │              ▼                │ │
│                                  │   ┌────────────────────────┐  │ │
│                                  │   │   PATCH AGENT 💊       │  │ │
│                                  │   │   (Fix Suggester)      │  │ │
│                                  │   └──────────┬─────────────┘  │ │
│                                  │              │                │ │
│                                  └──────────────┼────────────────┘ │
│                                                 │                   │
│  ┌──────────────────────┐                       │                   │
│  │                      │                       │                   │
│  │   OLLAMA             │◀──── LLM API calls ───┘                   │
│  │   (Local LLM)        │                       │                   │
│  │                      │                       │                   │
│  │   Port: 11434        │                       ▼                   │
│  │   Model: llama3.1    │        ┌────────────────────────────────┐ │
│  │                      │        │  OUTPUT LAYER                  │ │
│  │   OR                 │        │                                │ │
│  │   OpenAI API         │        │  💬 Slack Webhook (notify)    │ │
│  │   (if key available) │        │  🗃️ SQLite DB (healing log)   │ │
│  └──────────────────────┘        │  📊 Web Dashboard (history)   │ │
│                                  └────────────────────────────────┘ │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Layer Breakdown

```
LAYER 1: CI/CD (Jenkins)
├── Jenkins server running in Docker
├── Sample pipelines (designed to succeed and fail)
├── Webhook configured → POST to http://healing-engine:5000/webhook/jenkins
└── REST API for fetching build logs

LAYER 2: Self-Healing Engine (Python FastAPI)
├── Detection Agent → catches webhooks
├── Orchestrator Agent → classifies error, decides agents, manages brainstorming
├── Specialized Agents → each analyzes one aspect (logs, code, tests, deps)
├── Patch Agent → formats final output
└── Dashboard → web UI for healing history

LAYER 3: AI Brain (Ollama OR OpenAI)
├── Option A: Ollama Docker container running llama3.1 (free, local)
├── Option B: OpenAI API gpt-4o-mini (paid, cloud)
└── Switchable via .env config (AI_PROVIDER=ollama or openai)

LAYER 4: Output
├── Slack webhook → sends formatted notification
├── SQLite → stores all healing events
└── Web Dashboard → view history, agent activity
```

---

## 3. Agent Details

### Agent Overview Table

```
┌──────────────────┬─────────────────────┬──────────────────────────────────┐
│ Agent            │ Role                │ What It Does                     │
├──────────────────┼─────────────────────┼──────────────────────────────────┤
│ 🔍 Detection     │ Webhook Listener    │ Catches build failure events     │
│                  │                     │ from Jenkins, triggers healing   │
├──────────────────┼─────────────────────┼──────────────────────────────────┤
│ 🧠 Orchestrator  │ Decision Maker      │ Classifies error type, decides   │
│                  │                     │ which agents to activate,        │
│                  │                     │ manages brainstorm rounds        │
├──────────────────┼─────────────────────┼──────────────────────────────────┤
│ 📋 Diagnosis     │ Log Analyzer        │ Reads build logs, identifies     │
│                  │                     │ error messages, stack traces     │
├──────────────────┼─────────────────────┼──────────────────────────────────┤
│ 💻 Code          │ File Tracker        │ Identifies affected files, line  │
│                  │                     │ numbers, recent code changes     │
├──────────────────┼─────────────────────┼──────────────────────────────────┤
│ 🧪 Test          │ Test Analyzer       │ Identifies failing tests,        │
│                  │                     │ flaky vs real failures           │
├──────────────────┼─────────────────────┼──────────────────────────────────┤
│ 📦 Dependency    │ Dep Checker         │ Checks for missing or broken     │
│                  │                     │ dependencies in build config     │
├──────────────────┼─────────────────────┼──────────────────────────────────┤
│ 💊 Patch         │ Fix Formatter       │ Formats final diagnosis, sends   │
│                  │                     │ to Slack, logs to database       │
└──────────────────┴─────────────────────┴──────────────────────────────────┘
```

### Orchestrator Decision Tree

```
Build Failed
    │
    ▼
Orchestrator fetches logs from Jenkins API
    │
    ▼
Quick Classification (regex-based, no AI):
    │
    ├── Contains "cannot find symbol" / "compilation error" / "syntax error"
    │   └── Type: COMPILATION
    │       └── Activate: Diagnosis Agent + Code Agent
    │
    ├── Contains "test failed" / "AssertionError" / "junit"
    │   └── Type: TEST_FAILURE
    │       └── Activate: Diagnosis Agent + Test Agent
    │
    ├── Contains "could not resolve" / "dependency" / "404 not found"
    │   └── Type: DEPENDENCY
    │       └── Activate: Diagnosis Agent + Dependency Agent
    │
    ├── Contains "timeout" / "OutOfMemoryError" / "connection refused"
    │   └── Type: INFRASTRUCTURE
    │       └── Activate: Diagnosis Agent + Code Agent
    │
    └── None of the above
        └── Type: UNKNOWN
            └── Activate: ALL agents (Diagnosis + Code + Test + Dependency)
```

---

## 4. Agent Brainstorming Mechanism

### How Agents Communicate and Share Context

```
THE BRAINSTORMING PROTOCOL
══════════════════════════

Step 1: Orchestrator fetches build logs from Jenkins API

Step 2: Orchestrator classifies error → selects agents

Step 3: ROUND 1 — Independent Analysis (Parallel)
        ┌─────────────────────────────────────────────────┐
        │ Each selected agent receives: raw build logs     │
        │ Each agent calls LLM independently               │
        │ Each agent returns its initial finding            │
        │                                                   │
        │ Diagnosis: "NullPointerException at line 23"     │
        │ Code:      "App.java modified in commit abc"     │
        │ Test:      "3 tests failed in UserModule"        │
        └─────────────────────────────────────────────────┘
                              │
                              ▼
Step 4: BUILD SHARED CONTEXT
        ┌─────────────────────────────────────────────────┐
        │ Orchestrator collects ALL Round 1 results        │
        │ Creates a shared context dictionary:             │
        │                                                   │
        │ shared_context = {                                │
        │   "Diagnosis Agent": "NullPointer at line 23",   │
        │   "Code Agent": "App.java modified in abc",      │
        │   "Test Agent": "3 tests failed in UserModule"   │
        │ }                                                 │
        └─────────────────────────────────────────────────┘
                              │
                              ▼
Step 5: ROUND 2 — Brainstorm & Refine (Parallel)
        ┌─────────────────────────────────────────────────┐
        │ Each agent receives:                             │
        │   - Original build logs                          │
        │   - Its own Round 1 finding                      │
        │   - ALL other agents' Round 1 findings           │
        │                                                   │
        │ Each agent calls LLM with enriched prompt:       │
        │ "Here's what other agents found: [...]           │
        │  Refine your analysis. Connect the dots."        │
        │                                                   │
        │ Diagnosis: "NullPointer is IN the new code       │
        │             from commit abc!" (learned from Code)│
        │ Code:      "Change broke getUserSession() used   │
        │             by 3 tests" (learned from Test)      │
        │ Test:      "All 3 fail at same null pointer —    │
        │             confirms root cause" (from Diagnosis)│
        └─────────────────────────────────────────────────┘
                              │
                              ▼
Step 6: FINAL SYNTHESIS
        ┌─────────────────────────────────────────────────┐
        │ Orchestrator combines refined analyses            │
        │ Calculates overall confidence                     │
        │ Creates final diagnosis document                  │
        │ Passes to Patch Agent                             │
        └─────────────────────────────────────────────────┘
```

### Brainstorming in Code (Pseudocode)

```python
async def orchestrate(self, logs):
    # CLASSIFY
    error_type = self.quick_classify(logs)        # regex, no AI
    agents = self.select_agents(error_type)        # decision logic
    
    # ROUND 1: Independent (parallel)
    round1 = {}
    for agent in agents:
        round1[agent.name] = await agent.analyze(logs)
    
    # BUILD SHARED CONTEXT (just a dictionary!)
    shared_context = round1.copy()
    
    # ROUND 2: Brainstorm (parallel)
    round2 = {}
    for agent in agents:
        round2[agent.name] = await agent.refine(logs, shared_context)
    
    # SYNTHESIZE
    return self.synthesize(round2, error_type)
```

### What Makes Each Agent's Prompt Different

```
┌─────────────────────────────────────────────────────────────────┐
│ DIAGNOSIS AGENT PROMPT                                           │
│                                                                  │
│ Role: "You are a CI/CD build failure expert"                    │
│ Focus: Error messages, stack traces, exception types             │
│ Output: Root cause, error category, severity                    │
│                                                                  │
│ Round 2 addition: "Other agents found: [shared context]         │
│ Does this change your diagnosis? Connect the dots."             │
├─────────────────────────────────────────────────────────────────┤
│ CODE ANALYZER AGENT PROMPT                                       │
│                                                                  │
│ Role: "You are a source code analyst"                           │
│ Focus: File paths, line numbers, recent changes, diffs          │
│ Output: Affected file, line number, what changed                │
│                                                                  │
│ Round 2 addition: "Other agents found: [shared context]         │
│ Which file/change caused the issue they identified?"            │
├─────────────────────────────────────────────────────────────────┤
│ TEST ANALYZER AGENT PROMPT                                       │
│                                                                  │
│ Role: "You are a test failure analyst"                          │
│ Focus: Test names, assertion errors, flaky vs real              │
│ Output: Failing tests, failure reason, is it flaky?             │
│                                                                  │
│ Round 2 addition: "Other agents found: [shared context]         │
│ Do the test failures match the root cause they found?"          │
├─────────────────────────────────────────────────────────────────┤
│ DEPENDENCY AGENT PROMPT                                          │
│                                                                  │
│ Role: "You are a dependency and build config expert"            │
│ Focus: pom.xml, package.json, missing libs, version conflicts   │
│ Output: Missing dependency, version mismatch, fix command       │
│                                                                  │
│ Round 2 addition: "Other agents found: [shared context]         │
│ Is the error caused by a dependency issue?"                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. Complete Data Flow

### Trigger: Jenkins Build #42 Fails

```
TIME         EVENT                                    COMPONENT
─────────────────────────────────────────────────────────────────

0:00.000     Build #42 finishes with FAILURE          Jenkins
             Jenkins fires webhook POST               Jenkins

0:00.100     Webhook received at /webhook/jenkins      Detection Agent
             Payload: {job: "my-app", build: 42,
                       status: "FAILURE"}
             
             Detection Agent checks: FAILURE? → YES
             Passes to Orchestrator

0:00.200     Orchestrator receives event               Orchestrator
             Calls Jenkins API:
             GET /job/my-app/42/consoleText
             Receives 500 lines of build logs

0:00.500     Quick classification (regex scan):        Orchestrator
             Found "cannot find symbol" → COMPILATION
             Decision: activate Diagnosis + Code agents

0:01.000     ═══ ROUND 1: Independent Analysis ═══

             Diagnosis Agent → LLM call:               Diagnosis Agent
             "Analyze these logs for errors..."
             
             Code Agent → LLM call (parallel):         Code Agent
             "Find affected files and changes..."

0:03.000     Round 1 results collected:                Orchestrator
             - Diagnosis: "NullPointerException 
               in App.java at line 23"
             - Code: "App.java modified in 
               recent commit abc123"

0:03.100     Shared context built:                     Orchestrator
             {
               "Diagnosis Agent": "NullPointer...",
               "Code Agent": "App.java modified..."
             }

0:03.200     ═══ ROUND 2: Brainstorming ═══

             Diagnosis Agent → LLM call:               Diagnosis Agent
             "Other agents found: [Code Agent 
              says App.java modified]. Refine."
             
             Code Agent → LLM call (parallel):         Code Agent
             "Other agents found: [Diagnosis 
              says NullPointer]. Refine."

0:06.000     Round 2 results collected:                Orchestrator
             - Diagnosis: "NullPointer IS in the 
               new code from commit abc123"
             - Code: "Change in App.java broke 
               the getUserSession() method"

0:06.500     Final synthesis:                          Orchestrator
             {
               error_type: "COMPILATION",
               root_cause: "NullPointerException 
                 in new code from commit abc123",
               affected_file: "App.java",
               line: 23,
               fix: "Add null check before 
                 calling getUserSession()",
               confidence: 95,
               agents_used: ["Diagnosis", "Code"],
               brainstorm_rounds: 2
             }
             
             Passes to Patch Agent

0:07.000     Patch Agent formats Slack message         Patch Agent
             Sends to Slack webhook URL
             Saves to SQLite healing history
             Updates web dashboard

0:07.500     ═══ SLACK NOTIFICATION ARRIVES ═══        Slack

             ┌─────────────────────────────────┐
             │ 🔴 Build Failed: my-app #42     │
             │                                 │
             │ 🔍 Root Cause:                  │
             │ NullPointerException in          │
             │ App.java:23 (commit abc123)     │
             │                                 │
             │ 💊 Suggested Fix:               │
             │ Add null check before           │
             │ getUserSession()                │
             │                                 │
             │ 📊 Confidence: 95%              │
             │ 🤝 Agents: Diagnosis + Code     │
             │ 🔄 Brainstorm Rounds: 2         │
             │                                 │
             │ [✅ Retrigger] [📋 Details]     │
             └─────────────────────────────────┘

0:07.500     TOTAL TIME: ~7.5 seconds
             (vs 30-60 minutes manually)
```

---

## 6. Tech Stack

```
┌──────────────────┬─────────────────┬─────────────────────────────┐
│ Component        │ Technology      │ Why                         │
├──────────────────┼─────────────────┼─────────────────────────────┤
│ CI/CD Server     │ Jenkins (LTS)   │ Industry standard, Docker   │
│ Backend          │ Python 3.11     │ Fast dev, async support     │
│ API Framework    │ FastAPI         │ Async, auto-docs, fast      │
│ AI (Option A)    │ OpenAI API      │ GPT-4o-mini, if key avail.  │
│ AI (Option B)    │ Ollama + Llama  │ 100% local, free, open src  │
│ Notifications    │ Slack Webhooks  │ Rich messages, buttons      │
│ Database         │ SQLite          │ Zero setup, file-based      │
│ Dashboard        │ HTML/CSS/JS     │ Simple, no build needed     │
│ Containerization │ Docker Compose  │ One command startup         │
│ HTTP Client      │ httpx (Python)  │ Async HTTP calls            │
└──────────────────┴─────────────────┴─────────────────────────────┘

ALL open source. ALL free. ALL local.
```

### AI Backend Configuration (.env)

```env
# Option A: OpenAI (if you have the key)
AI_PROVIDER=openai
OPENAI_API_KEY=sk-your-key-here
OPENAI_MODEL=gpt-4o-mini

# Option B: Ollama (free, local, open source)
AI_PROVIDER=ollama
OLLAMA_URL=http://ollama:11434
OLLAMA_MODEL=llama3.1
```

---

## 7. Project File Structure

```
self-healing-cicd/
│
├── docker-compose.yml                  # Starts everything: Jenkins + Engine + Ollama
├── .env                                # AI_PROVIDER config (openai or ollama)
├── .env.example                        # Template for .env
│
├── healing-engine/                     # Main Python application
│   ├── Dockerfile                      # Python app container
│   ├── requirements.txt                # Python dependencies
│   ├── main.py                         # FastAPI entry point + routes
│   │
│   ├── agents/                         # All AI agents
│   │   ├── __init__.py
│   │   ├── base_agent.py              # Base class (LLM calling logic)
│   │   ├── detection_agent.py         # Webhook listener
│   │   ├── orchestrator_agent.py      # The brain — decision + brainstorming
│   │   ├── diagnosis_agent.py         # Log analyzer
│   │   ├── code_agent.py             # File/line identifier
│   │   ├── test_agent.py             # Test failure analyzer
│   │   ├── dependency_agent.py       # Dependency checker
│   │   └── patch_agent.py            # Fix formatter + notifier
│   │
│   ├── services/                       # External integrations
│   │   ├── __init__.py
│   │   ├── ai_service.py             # OpenAI / Ollama switcher
│   │   ├── jenkins_service.py        # Jenkins REST API client
│   │   └── slack_service.py          # Slack webhook sender
│   │
│   ├── models/                         # Data models
│   │   ├── __init__.py
│   │   └── schemas.py                 # Pydantic models
│   │
│   ├── database/                       # Storage
│   │   ├── __init__.py
│   │   └── healing_history.py         # SQLite CRUD for healing events
│   │
│   └── dashboard/                      # Web UI (served by FastAPI)
│       ├── index.html                  # Main dashboard page
│       ├── style.css                   # Dashboard styling
│       └── script.js                   # Dashboard logic
│
├── jenkins-config/                     # Jenkins setup
│   ├── Dockerfile                      # Jenkins with plugins pre-installed
│   ├── plugins.txt                     # Required Jenkins plugins
│   └── jobs/                           # Sample pipeline jobs
│       ├── Jenkinsfile-success         # Pipeline that always passes
│       ├── Jenkinsfile-compile-error   # Pipeline: compilation failure
│       ├── Jenkinsfile-test-failure    # Pipeline: test failure
│       └── Jenkinsfile-dependency-err  # Pipeline: missing dependency
│
└── README.md                           # Setup instructions
```

---

## 8. Step-by-Step Implementation Guide

### PHASE 1: Foundation (Day 1-2)

#### Step 1.1: Create Project Structure
```
ACTION: Create all folders and empty files
WHERE TO START: Create self-healing-cicd/ folder
```

#### Step 1.2: Write docker-compose.yml
```yaml
# Creates 3 services: Jenkins + Healing Engine + Ollama
# All networked together
# Volumes for persistence

services:
  jenkins:
    build: ./jenkins-config
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_data:/var/jenkins_home
    networks:
      - healing-network

  healing-engine:
    build: ./healing-engine
    ports:
      - "5000:5000"
    env_file: .env
    depends_on:
      - jenkins
      - ollama
    networks:
      - healing-network

  ollama:
    image: ollama/ollama:latest
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    networks:
      - healing-network

volumes:
  jenkins_data:
  ollama_data:

networks:
  healing-network:
    driver: bridge
```

#### Step 1.3: Jenkins Dockerfile + Sample Pipelines
```
ACTION: Create Jenkins container with webhook plugin
        Create 3-4 sample Jenkinsfiles that fail in predictable ways
```

#### Step 1.4: Healing Engine Dockerfile + requirements.txt
```
ACTION: Python 3.11 container
        Install: fastapi, uvicorn, httpx, python-dotenv, aiosqlite
```

#### Step 1.5: Verify
```
RUN: docker-compose up -d
CHECK: Jenkins at http://localhost:8080
CHECK: FastAPI at http://localhost:5000/docs
CHECK: Ollama at http://localhost:11434
```

---

### PHASE 2: Detection Agent + Jenkins Service (Day 3)

#### Step 2.1: Build Jenkins Service
```python
# services/jenkins_service.py
# - get_build_logs(job_name, build_number) → string
# - get_build_info(job_name, build_number) → dict
# - trigger_build(job_name) → bool
# - Uses httpx to call Jenkins REST API
```

#### Step 2.2: Build Detection Agent
```python
# agents/detection_agent.py
# - Receives POST /webhook/jenkins
# - Validates payload
# - Checks if status == "FAILURE"
# - If yes → calls orchestrator.orchestrate()
```

#### Step 2.3: Wire Up FastAPI Route
```python
# main.py
# POST /webhook/jenkins → detection_agent.handle_webhook()
# GET /health → health check
```

#### Step 2.4: Configure Jenkins Webhook
```
ACTION: In Jenkins job config → Post-build Actions
        → Add webhook: http://healing-engine:5000/webhook/jenkins
```

#### Step 2.5: Verify
```
TEST: Trigger a failing build in Jenkins
CHECK: Detection Agent logs "Build failure detected: my-app #42"
```

---

### PHASE 3: AI Service + Base Agent (Day 4)

#### Step 3.1: Build AI Service (Dual Backend)
```python
# services/ai_service.py
# - ask(prompt) → string
# - Checks AI_PROVIDER env var
# - If "openai" → calls OpenAI API
# - If "ollama" → calls Ollama local API
# - Both return plain text response
```

#### Step 3.2: Build Base Agent
```python
# agents/base_agent.py
# - BaseAgent class with name and role
# - analyze(logs) → calls LLM with role-specific prompt
# - refine(logs, shared_context) → calls LLM with enhanced prompt
# - All specialized agents inherit from this
```

#### Step 3.3: Pull Ollama Model
```
RUN: docker exec -it ollama ollama pull llama3.1
WAIT: Model downloads (~4.7GB)
```

#### Step 3.4: Verify
```
TEST: Call ai_service.ask("What is 2+2?")
CHECK: Returns "4" (from Ollama or OpenAI)
```

---

### PHASE 4: Specialized Agents (Day 5)

#### Step 4.1: Diagnosis Agent
```python
# agents/diagnosis_agent.py
# Inherits BaseAgent
# Role: "CI/CD build failure expert — find root cause"
# Focus: error messages, stack traces, exception types
```

#### Step 4.2: Code Analyzer Agent
```python
# agents/code_agent.py
# Inherits BaseAgent
# Role: "Source code analyst — find affected files"
# Focus: file paths, line numbers, recent changes
```

#### Step 4.3: Test Analyzer Agent
```python
# agents/test_agent.py
# Inherits BaseAgent
# Role: "Test failure analyst"
# Focus: test names, assertion errors, flaky detection
```

#### Step 4.4: Dependency Agent
```python
# agents/dependency_agent.py
# Inherits BaseAgent
# Role: "Dependency and build config expert"
# Focus: pom.xml, package.json, version conflicts
```

#### Step 4.5: Verify
```
TEST: Call each agent's analyze() with sample failure logs
CHECK: Each returns reasonable analysis
```

---

### PHASE 5: Orchestrator + Brainstorming (Day 6)

#### Step 5.1: Build Orchestrator Agent
```python
# agents/orchestrator_agent.py
# - quick_classify(logs) → error_type (regex)
# - select_agents(error_type) → list of agents
# - orchestrate(logs) → full brainstorm flow:
#     Round 1 → shared context → Round 2 → synthesize
```

#### Step 5.2: Wire Complete Flow
```
Detection → Orchestrator → Agents → Brainstorm → Result
Full end-to-end test with real Jenkins failure
```

#### Step 5.3: Verify
```
TEST: Trigger compile-error pipeline in Jenkins
CHECK: Orchestrator classifies as COMPILATION
CHECK: Activates Diagnosis + Code agents
CHECK: Round 1 results appear
CHECK: Round 2 refined results appear
CHECK: Final synthesis has root cause + fix
```

---

### PHASE 6: Slack + Patch Agent + Database (Day 7)

#### Step 6.1: Build Slack Service
```python
# services/slack_service.py
# - send_notification(diagnosis) → bool
# - Formats rich message with blocks
# - Sends to Slack incoming webhook URL
```

#### Step 6.2: Build Patch Agent
```python
# agents/patch_agent.py
# - Takes final synthesis from Orchestrator
# - Formats for Slack (root cause, fix, confidence)
# - Calls slack_service.send_notification()
# - Saves to healing history database
```

#### Step 6.3: Build Healing History Database
```python
# database/healing_history.py
# SQLite with table: healing_events
# Columns: id, timestamp, job_name, build_number,
#           error_type, root_cause, suggested_fix,
#           confidence, agents_used, round1_data, round2_data
```

#### Step 6.4: Verify
```
TEST: Full end-to-end: Jenkins fail → Slack notification
CHECK: Slack message appears with root cause and fix
CHECK: Healing event saved in SQLite
```

---

### PHASE 7: Web Dashboard (Day 8)

#### Step 7.1: Build Dashboard HTML
```
- Healing history table (recent events)
- Agent activity feed (which agents ran, what they found)
- Build status cards (pass/fail counts)
- Real-time updates (polling every 5 seconds)
```

#### Step 7.2: Add FastAPI Routes
```python
# GET /dashboard → serves index.html
# GET /api/healing-history → returns healing events JSON
# GET /api/stats → returns summary stats
```

#### Step 7.3: Verify
```
TEST: Open http://localhost:5000/dashboard
CHECK: Shows healing events with timeline
CHECK: Shows agent activity for each event
```

---

### PHASE 8: Retrigger Build Flow (Day 8)

#### Step 8.1: Slack Interactive Button
```
- When "Retrigger Build" clicked in Slack
- Slack sends callback to /slack/actions
- Engine calls Jenkins API to trigger new build
- Sends Slack confirmation: "Build #43 triggered"
```

#### Step 8.2: Build Monitoring
```
- After retrigger, poll Jenkins for build result
- When build #43 completes:
  - If SUCCESS → Slack: "✅ Build #43 passed! Self-healing worked!"
  - If FAILURE → Slack: "❌ Build #43 still failing. Manual review needed."
```

---

### PHASE 9: Polish + PPT (Day 9)

#### Step 9.1: Error Handling
```
- Add try/except everywhere
- Handle: Jenkins unreachable, LLM timeout, Slack failure
- Graceful fallbacks for each
```

#### Step 9.2: Logging
```
- Add structured logging with timestamps
- Log every agent action for debugging
- Log brainstorming rounds clearly
```

#### Step 9.3: Finalize PPT
```
- Use architect-pitch.html as reference
- Create PowerPoint with 10 slides
- Practice the story: Problem → Solution → Architecture → Demo → Impact
```

---

### PHASE 10: Demo Video (Day 10-11)

#### Step 10.1: Demo Script
```
1. Show Jenkins with a failed build (the problem)
2. Show the self-healing in action:
   - Build fails → agents detect → brainstorm → Slack notification
3. Show the Slack message with root cause + fix
4. Click retrigger → build passes
5. Show the dashboard with healing history
6. Show the architecture diagram
```

#### Step 10.2: Record
```
- Screen record with OBS or Windows built-in
- Record narration separately if needed
- Keep it under 5 minutes
```

---

## 9. Code Architecture Per File

### main.py (FastAPI Entry Point)
```
Routes:
  POST /webhook/jenkins          → Detection Agent
  POST /slack/actions            → Handle Slack button clicks
  GET  /api/healing-history      → List healing events
  GET  /api/healing-history/{id} → Get single event details
  GET  /api/stats                → Summary statistics
  GET  /dashboard                → Serve web dashboard
  GET  /health                   → Health check
```

### agents/base_agent.py
```
class BaseAgent:
  __init__(name, role_description)
  async analyze(logs) → str           # Round 1
  async refine(logs, shared_ctx) → str # Round 2
  _build_analyze_prompt(logs) → str
  _build_refine_prompt(logs, ctx) → str
```

### agents/orchestrator_agent.py
```
class OrchestratorAgent:
  __init__()                                    # creates all agent instances
  quick_classify(logs) → str                    # regex classification
  select_agents(error_type) → list[BaseAgent]   # decision logic
  async orchestrate(job, build, logs) → dict    # full brainstorm flow
  async _run_round1(agents, logs) → dict        # parallel Round 1
  async _run_round2(agents, logs, ctx) → dict   # parallel Round 2
  synthesize(round2_results, error_type) → dict # final synthesis
```

### services/ai_service.py
```
class AIService:
  __init__()                        # reads AI_PROVIDER from env
  async ask(prompt) → str           # routes to correct backend
  async _ask_openai(prompt) → str   # OpenAI API call
  async _ask_ollama(prompt) → str   # Ollama local API call
```

### services/jenkins_service.py
```
class JenkinsService:
  __init__(url, user, token)
  async get_build_logs(job, build) → str
  async get_build_info(job, build) → dict
  async trigger_build(job) → bool
```

### services/slack_service.py
```
class SlackService:
  __init__(webhook_url)
  async send_healing_notification(diagnosis) → bool
  async send_build_result(job, build, status) → bool
  _format_slack_blocks(diagnosis) → dict
```

### database/healing_history.py
```
class HealingHistory:
  async init_db()
  async save_event(event) → int
  async get_all_events() → list
  async get_event(id) → dict
  async get_stats() → dict
```

---

## 10. Demo Plan

### Demo Flow (5 minutes)

```
PART 1: THE PROBLEM (1 min)
├── Show Jenkins dashboard
├── Click on failed build #42
├── Show 1000+ lines of console output
└── Say: "This is what developers deal with daily"

PART 2: THE SELF-HEALING (2 min)
├── Trigger a build that will fail
├── Watch terminal: "🔍 Detection Agent: Build failure detected!"
├── Watch terminal: "🧠 Orchestrator: COMPILATION error, activating Diagnosis + Code"
├── Watch terminal: "═══ ROUND 1 ═══"
├── Watch terminal: "📋 Diagnosis: NullPointerException..."
├── Watch terminal: "💻 Code: App.java modified..."
├── Watch terminal: "═══ ROUND 2: BRAINSTORMING ═══"
├── Watch terminal: "📋 Diagnosis (refined): Error is in new code from commit..."
├── Watch terminal: "💊 Patch Agent: Sending to Slack..."
└── Show Slack notification appearing within 10 seconds

PART 3: THE SLACK MESSAGE (1 min)
├── Show the rich Slack message
├── Root cause, affected file, suggested fix
├── 95% confidence from agent consensus
├── Click "Retrigger Build" button
└── Show: "✅ Build #43 passed!"

PART 4: THE DASHBOARD (0.5 min)
├── Open http://localhost:5000/dashboard
├── Show healing history timeline
└── Show agent activity for this event

PART 5: WRAP-UP (0.5 min)
├── Show architecture diagram (from PPT)
├── "Total time: 10 seconds vs 30 minutes"
└── "Questions?"
```

---

## 11. Timeline

```
┌─────┬──────────┬───────────────────────────────────────────┬──────────┐
│ Day │ Date     │ Task                                      │ Output   │
├─────┼──────────┼───────────────────────────────────────────┼──────────┤
│  1  │ Mar 3    │ Docker Compose + Jenkins setup             │ Jenkins  │
│     │          │ Create sample failing pipelines            │ running  │
├─────┼──────────┼───────────────────────────────────────────┼──────────┤
│  2  │ Mar 4    │ FastAPI skeleton + project structure       │ API up   │
│     │          │ Jenkins webhook integration                │          │
├─────┼──────────┼───────────────────────────────────────────┼──────────┤
│  3  │ Mar 5    │ Detection Agent + Jenkins Service          │ Webhook  │
│     │          │ Test: Jenkins fail → agent catches it      │ works    │
├─────┼──────────┼───────────────────────────────────────────┼──────────┤
│  4  │ Mar 6    │ AI Service (OpenAI + Ollama dual backend) │ AI calls │
│     │          │ Base Agent + Diagnosis Agent               │ work     │
├─────┼──────────┼───────────────────────────────────────────┼──────────┤
│  5  │ Mar 7    │ Code Agent + Test Agent + Dependency Agent│ All      │
│     │          │ Test each agent independently              │ agents   │
├─────┼──────────┼───────────────────────────────────────────┼──────────┤
│  6  │ Mar 8    │ Orchestrator (classification + brainstorm)│ Brain    │
│     │          │ Full Round 1 → Round 2 flow               │ works    │
├─────┼──────────┼───────────────────────────────────────────┼──────────┤
│  7  │ Mar 9    │ Slack notification + Patch Agent           │ Slack    │
│     │          │ SQLite healing history                     │ notifies │
├─────┼──────────┼───────────────────────────────────────────┼──────────┤
│  8  │ Mar 10   │ Web Dashboard + Polish + PPT              │ SUBMIT   │
│     │          │ ⭐ SUBMIT IDEA TO ARCHITECT ⭐             │ PPT      │
├─────┼──────────┼───────────────────────────────────────────┼──────────┤
│  9  │ Mar 11   │ Retrigger build flow                      │ End-to-  │
│     │          │ Full end-to-end testing                    │ end      │
├─────┼──────────┼───────────────────────────────────────────┼──────────┤
│ 10  │ Mar 12   │ Record demo video                         │ Video    │
│     │          │ Final bug fixes                            │ ready    │
├─────┼──────────┼───────────────────────────────────────────┼──────────┤
│ 11  │ Mar 13   │ ⭐ SUBMIT DEMO VIDEO ⭐                   │ DONE!    │
└─────┴──────────┴───────────────────────────────────────────┴──────────┘
```

---

## 12. Future Roadmap

These are NOT built in the hackathon but mentioned in the PPT as "where this can go":

```
NOW (Hackathon)
├── ✅ Detection Agent (webhook)
├── ✅ Orchestrator Agent (classification + brainstorming)  
├── ✅ Diagnosis Agent (log analysis)
├── ✅ Code Agent (file tracking)
├── ✅ Test Agent (test analysis)
├── ✅ Patch Agent (Slack notification)
├── ✅ Healing History Database
└── ✅ Web Dashboard

NEXT (Post-Hackathon)
├── 🔬 Validation Agent — auto-test AI-generated patches
├── 🚀 Deployment Agent — auto-apply approved fixes
├── 📈 ML Pattern Matching — learn from past fixes
└── 🌐 Cross-Service Healing — trace failures across microservices

FUTURE (Vision)
├── 🔄 Fully autonomous fix-deploy-verify cycle
├── 🧬 Agent-driven code evolution (optimize, reduce debt)
└── ♾️ Closed-loop development (zero human intervention)
```

---

## Quick Start Command

Once everything is built:

```bash
# One command to start the entire self-healing system
docker-compose up -d

# Check services
# Jenkins:  http://localhost:8080
# Engine:   http://localhost:5000
# Dashboard: http://localhost:5000/dashboard
# Ollama:   http://localhost:11434
```

---

**Ready to build. Let's go! 🚀**
