# Self-Healing CI/CD — FINAL BUILD GUIDE (Step-by-Step)

**Start Date**: March 4, 2026
**Deadline 1**: March 10 — Architect PPT submission
**Deadline 2**: March 13 — Working demo video submission

---

## Your Confirmed Setup

| Item | Value |
|------|-------|
| OS | Windows |
| RAM | 16GB+ |
| Docker Desktop | ✅ Installed |
| Python | ✅ Available |
| Slack | ✅ Available |
| Email SMTP | ✅ Available |
| AI Primary | OpenAI API (if key available) |
| AI Fallback | Ollama Docker (llama3.1, local) |

---

## Final Agent Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ TIER 0   🔍 Detection Agent        [No LLM]   ~100ms       │
│                    │                                         │
│                    ▼                                         │
│          🧠 Orchestrator Agent      [No LLM]   ~50ms        │
│             (+ Confidence Scoring)                           │
│                    │                                         │
│          Classifies error → selects agents                   │
│                    │                                         │
│ TIER 1   ─────────┼──────────────   [No LLM]   ~200ms      │
│          │                   │       (parallel)              │
│          ▼                   ▼                               │
│     📜 Log Parser      📝 Git Diff                          │
│     (regex)            (Jenkins API)                         │
│          │                   │                               │
│          └─────────┬─────────┘                               │
│                    │ structured data                          │
│                    ▼                                         │
│ TIER 2   🔬 Root Cause Agent        [LLM]      ~3-4s       │
│              │                                               │
│              │ brainstorm (share context)                     │
│              ▼                                               │
│ TIER 3   💊 Fix Agent               [LLM]      ~2-3s       │
│              │                                               │
│              ▼                                               │
│ OUTPUT   💬 Notify Agent            [No LLM]   ~500ms      │
│              │         │         │                           │
│              ▼         ▼         ▼                           │
│           Slack     Email     Dashboard                      │
│                                  │                           │
│                                  ▼                           │
│                              SQLite DB                       │
│                          (healing history)                    │
│                                                              │
│ TOTAL: 3 LLM calls | ~7-10s (OpenAI) | ~25-35s (Ollama)   │
└─────────────────────────────────────────────────────────────┘
```

---

## Final Project File Structure

```
self-healing-cicd/
│
├── docker-compose.yml
├── .env
├── .env.example
├── README.md
│
├── healing-engine/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── main.py                         # FastAPI entry point
│   │
│   ├── agents/
│   │   ├── __init__.py
│   │   ├── base_agent.py              # Base class with LLM calling
│   │   ├── detection_agent.py         # Tier 0: webhook handler
│   │   ├── orchestrator_agent.py      # Brain + confidence scoring
│   │   ├── log_parser_agent.py        # Tier 1: regex extraction
│   │   ├── git_diff_agent.py          # Tier 1: commit info
│   │   ├── root_cause_agent.py        # Tier 2: LLM reasoning
│   │   ├── fix_agent.py              # Tier 3: LLM fix generation
│   │   └── notify_agent.py           # Output: Slack + email + DB
│   │
│   ├── services/
│   │   ├── __init__.py
│   │   ├── ai_service.py             # OpenAI / Ollama switcher
│   │   ├── jenkins_service.py        # Jenkins REST API client
│   │   ├── slack_service.py          # Slack webhook sender
│   │   └── email_service.py          # SMTP email sender
│   │
│   ├── models/
│   │   ├── __init__.py
│   │   └── schemas.py                 # Pydantic data models
│   │
│   ├── database/
│   │   ├── __init__.py
│   │   └── healing_history.py         # SQLite operations
│   │
│   └── dashboard/
│       ├── index.html
│       ├── style.css
│       └── script.js
│
├── jenkins-config/
│   ├── Dockerfile
│   ├── plugins.txt
│   └── jobs/
│       ├── Jenkinsfile-success
│       ├── Jenkinsfile-compile-error
│       ├── Jenkinsfile-test-failure
│       └── Jenkinsfile-dependency-err
│
└── docs/
    ├── DISCUSSION_SUMMARY.md
    ├── FINAL_IMPLEMENTATION_PLAN.md
    ├── architect-pitch.html
    └── gitnation_comparison.md
```

---

## DAY-BY-DAY BUILD GUIDE

---

### DAY 1 — March 4 (Today): Foundation Setup

**Goal**: Get Jenkins + Python app + Ollama running in Docker, all talking to each other.

#### Step 1.1: Create project folder
```powershell
mkdir C:\Users\dell\Downloads\Hackathon\self-healing-cicd
cd C:\Users\dell\Downloads\Hackathon\self-healing-cicd
```

#### Step 1.2: Create docker-compose.yml
```yaml
version: '3.8'

services:
  jenkins:
    build: ./jenkins-config
    container_name: jenkins
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_data:/var/jenkins_home
    networks:
      - healing-network
    restart: unless-stopped

  healing-engine:
    build: ./healing-engine
    container_name: healing-engine
    ports:
      - "5000:5000"
    env_file: .env
    depends_on:
      - jenkins
    volumes:
      - ./healing-engine:/app
      - healing_db:/app/data
    networks:
      - healing-network
    restart: unless-stopped

  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    networks:
      - healing-network
    restart: unless-stopped

volumes:
  jenkins_data:
  ollama_data:
  healing_db:

networks:
  healing-network:
    driver: bridge
```

#### Step 1.3: Create .env file
```env
# AI Configuration
AI_PROVIDER=ollama
# If you have OpenAI key, change to:
# AI_PROVIDER=openai
# OPENAI_API_KEY=sk-your-key-here
# OPENAI_MODEL=gpt-4o-mini

# Ollama Configuration
OLLAMA_URL=http://ollama:11434
OLLAMA_MODEL=llama3.1

# Jenkins Configuration
JENKINS_URL=http://jenkins:8080
JENKINS_USER=admin
JENKINS_TOKEN=will-be-set-after-jenkins-setup

# Slack Configuration
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL

# Email Configuration
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASSWORD=your-app-password
NOTIFICATION_EMAIL=your-email@gmail.com

# Brainstorm Configuration
BRAINSTORM_ENABLED=true

# App Configuration
APP_PORT=5000
DATABASE_PATH=/app/data/healing_history.db
```

#### Step 1.4: Create jenkins-config/Dockerfile
```dockerfile
FROM jenkins/jenkins:lts

# Skip initial setup wizard
ENV JAVA_OPTS="-Djenkins.install.runSetupWizard=false"

# Install required plugins
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt
```

#### Step 1.5: Create jenkins-config/plugins.txt
```
workflow-aggregator
git
pipeline-stage-view
http-request
generic-webhook-trigger
```

#### Step 1.6: Create healing-engine/Dockerfile
```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Create data directory for SQLite
RUN mkdir -p /app/data

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "5000", "--reload"]
```

#### Step 1.7: Create healing-engine/requirements.txt
```
fastapi==0.109.0
uvicorn==0.27.0
httpx==0.26.0
python-dotenv==1.0.0
aiosqlite==0.19.0
pydantic==2.5.3
jinja2==3.1.3
aiosmtplib==3.0.1
```

#### Step 1.8: Create minimal healing-engine/main.py
```python
from fastapi import FastAPI, Request
from fastapi.responses import HTMLResponse
import os
from dotenv import load_dotenv

load_dotenv()

app = FastAPI(title="Self-Healing CI/CD Engine")

@app.get("/health")
async def health():
    return {"status": "healthy", "service": "self-healing-engine"}

@app.post("/webhook/jenkins")
async def jenkins_webhook(request: Request):
    payload = await request.json()
    print(f"🔍 Webhook received: {payload}")
    return {"status": "received", "payload": payload}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=5000)
```

#### Step 1.9: Create empty __init__.py files
```powershell
# Run from self-healing-cicd directory
New-Item -ItemType File -Path healing-engine/agents/__init__.py -Force
New-Item -ItemType File -Path healing-engine/services/__init__.py -Force
New-Item -ItemType File -Path healing-engine/models/__init__.py -Force
New-Item -ItemType File -Path healing-engine/database/__init__.py -Force
```

#### Step 1.10: Start everything
```powershell
docker-compose up -d
```

#### Step 1.11: Pull Ollama model (while Jenkins sets up)
```powershell
docker exec -it ollama ollama pull llama3.1
```

#### Step 1.12: Verify
```
CHECK: http://localhost:8080       → Jenkins is running
CHECK: http://localhost:5000/health → {"status": "healthy"}
CHECK: http://localhost:5000/docs  → FastAPI docs page
```

#### ✅ Day 1 Done When:
- [ ] Docker Compose starts all 3 services
- [ ] Jenkins accessible at :8080
- [ ] FastAPI accessible at :5000
- [ ] Ollama model downloaded

---

### DAY 2 — March 5: Jenkins Pipelines + Jenkins Service

**Goal**: Create sample pipelines that fail, build Jenkins API client.

#### Step 2.1: Get Jenkins admin password
```powershell
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```
Use this to login at http://localhost:8080 (if wizard isn't skipped)

#### Step 2.2: Create API token in Jenkins
```
Jenkins → Your Profile → Configure → API Token → Generate
Copy the token → paste in .env as JENKINS_TOKEN
```

#### Step 2.3: Create sample Jenkinsfiles

**jenkins-config/jobs/Jenkinsfile-success** (always passes):
```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                echo 'Building application...'
                sh 'echo "Compilation successful"'
            }
        }
        stage('Test') {
            steps {
                echo 'Running tests...'
                sh 'echo "All 15 tests passed"'
            }
        }
    }
}
```

**jenkins-config/jobs/Jenkinsfile-compile-error** (compilation failure):
```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                echo 'Building application...'
                sh '''
                    echo "Compiling src/main/java/com/example/App.java..."
                    echo "src/main/java/com/example/App.java:23: error: cannot find symbol"
                    echo "    user.getSession().save();"
                    echo "        ^"
                    echo "  symbol:   method getSession()"
                    echo "  location: variable user of type User"
                    echo ""
                    echo "src/main/java/com/example/App.java:45: error: incompatible types"
                    echo "    int result = calculateTotal();"
                    echo "                 ^"
                    echo "  required: int"
                    echo "  found:    String"
                    echo ""
                    echo "2 errors"
                    echo "BUILD FAILED"
                    exit 1
                '''
            }
        }
    }
}
```

**jenkins-config/jobs/Jenkinsfile-test-failure** (test failure):
```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                echo 'Building application...'
                sh 'echo "BUILD SUCCESS"'
            }
        }
        stage('Test') {
            steps {
                sh '''
                    echo "Running test suite..."
                    echo "com.example.UserServiceTest > testGetUser PASSED"
                    echo "com.example.UserServiceTest > testCreateUser PASSED"
                    echo "com.example.AuthModuleTest > testLogin FAILED"
                    echo ""
                    echo "java.lang.AssertionError: Expected status 200 but got 401"
                    echo "    at com.example.AuthModuleTest.testLogin(AuthModuleTest.java:42)"
                    echo "    at org.junit.runner.JUnitCore.run(JUnitCore.java:137)"
                    echo ""
                    echo "com.example.AuthModuleTest > testLogout FAILED"
                    echo "java.lang.NullPointerException"
                    echo "    at com.example.AuthModuleTest.testLogout(AuthModuleTest.java:67)"
                    echo ""
                    echo "com.example.PaymentTest > testPayment PASSED"
                    echo ""
                    echo "Tests run: 5, Failures: 2, Errors: 0, Skipped: 0"
                    echo "BUILD FAILED"
                    exit 1
                '''
            }
        }
    }
}
```

**jenkins-config/jobs/Jenkinsfile-dependency-err** (dependency error):
```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh '''
                    echo "Resolving dependencies..."
                    echo "Downloading com.google.guava:guava:31.1..."
                    echo "Downloading org.springframework:spring-core:5.3.23..."
                    echo ""
                    echo "FAILURE: Could not resolve dependency:"
                    echo "  com.internal.lib:payment-sdk:2.4.1"
                    echo "  Required by: com.example:my-app:1.0.0"
                    echo ""
                    echo "Possible solution:"
                    echo "  - Check that the repository 'https://repo.internal.com/maven' is accessible"
                    echo "  - Verify the artifact 'payment-sdk:2.4.1' exists"
                    echo ""
                    echo "BUILD FAILED"
                    exit 1
                '''
            }
        }
    }
}
```

#### Step 2.4: Create pipeline jobs in Jenkins UI
```
For each Jenkinsfile:
1. Jenkins → New Item → Pipeline → name it (e.g., "demo-compile-error")
2. Pipeline → Definition: Pipeline script
3. Paste the Jenkinsfile content
4. Save
5. Build Now → verify it fails as expected
```

#### Step 2.5: Build Jenkins Service
Create `healing-engine/services/jenkins_service.py`:
```python
import httpx
import os
import base64

class JenkinsService:
    def __init__(self):
        self.base_url = os.getenv("JENKINS_URL", "http://jenkins:8080")
        self.user = os.getenv("JENKINS_USER", "admin")
        self.token = os.getenv("JENKINS_TOKEN", "")
    
    def _get_auth_header(self):
        credentials = f"{self.user}:{self.token}"
        encoded = base64.b64encode(credentials.encode()).decode()
        return {"Authorization": f"Basic {encoded}"}
    
    async def get_build_logs(self, job_name: str, build_number: int) -> str:
        url = f"{self.base_url}/job/{job_name}/{build_number}/consoleText"
        async with httpx.AsyncClient() as client:
            response = await client.get(url, headers=self._get_auth_header(), timeout=10.0)
            return response.text
    
    async def get_build_info(self, job_name: str, build_number: int) -> dict:
        url = f"{self.base_url}/job/{job_name}/{build_number}/api/json"
        async with httpx.AsyncClient() as client:
            response = await client.get(url, headers=self._get_auth_header(), timeout=10.0)
            return response.json()
    
    async def trigger_build(self, job_name: str) -> bool:
        url = f"{self.base_url}/job/{job_name}/build"
        async with httpx.AsyncClient() as client:
            response = await client.post(url, headers=self._get_auth_header(), timeout=10.0)
            return response.status_code in [200, 201]
```

#### ✅ Day 2 Done When:
- [ ] 4 pipeline jobs created in Jenkins (1 success, 3 failures)
- [ ] Each pipeline runs and fails/passes as expected
- [ ] Jenkins Service can fetch build logs via API
- [ ] Jenkins API token configured in .env

---

### DAY 3 — March 6: AI Service + Tier 1 Agents

**Goal**: Build AI backend (OpenAI/Ollama) + Log Parser + Git Diff agents.

#### Step 3.1: Build AI Service
Create `healing-engine/services/ai_service.py`
- `ask(prompt)` → routes to OpenAI or Ollama based on `.env`
- Handles timeouts, retries, JSON parsing from LLM output
- Include the `parse_llm_response()` helper for extracting JSON from messy LLM text

#### Step 3.2: Build Log Parser Agent
Create `healing-engine/agents/log_parser_agent.py`
- NO LLM — pure regex
- `parse(logs)` → returns `StructuredLogData`:
  - `error_lines[]`, `stack_traces[]`, `warnings[]`
  - `failed_stage`, `build_duration`, `last_50_lines`

#### Step 3.3: Build Git Diff Agent
Create `healing-engine/agents/git_diff_agent.py`
- NO LLM — Jenkins API call
- `get_changes(job_name, build_number)` → returns `CommitData`:
  - `commit_hash`, `author`, `message`, `files_changed[]`

#### Step 3.4: Verify
```
TEST: Call ai_service.ask("Say hello in 5 words")
CHECK: Gets response from Ollama or OpenAI

TEST: Feed sample failure logs to Log Parser
CHECK: Returns structured error data

TEST: Call Git Diff Agent for a real build
CHECK: Returns commit info
```

#### ✅ Day 3 Done When:
- [ ] AI Service works with both OpenAI and Ollama
- [ ] Log Parser extracts errors from all 3 failure types
- [ ] Git Diff Agent returns commit info

---

### DAY 4 — March 7: Root Cause Agent + Fix Agent + Brainstorming

**Goal**: Build the LLM-powered agents and the brainstorming flow.

#### Step 4.1: Build Base Agent
Create `healing-engine/agents/base_agent.py`
- Abstract class with `analyze()` and `refine()` methods
- Handles LLM prompt building and response parsing

#### Step 4.2: Build Root Cause Agent
Create `healing-engine/agents/root_cause_agent.py`
- Inherits BaseAgent
- Takes Tier 1 data (parsed logs + git diff) as input
- LLM prompt focused on: error type, root cause, affected file/line
- Returns structured `RootCauseAnalysis`

#### Step 4.3: Build Fix Agent
Create `healing-engine/agents/fix_agent.py`
- Inherits BaseAgent
- Takes Root Cause analysis as input
- LLM prompt focused on: fix steps, code change, explanation
- Returns structured `FixSuggestion`

#### Step 4.4: Implement Brainstorming in Orchestrator
- Root Cause Agent → Round 1 analysis
- Fix Agent → Round 1 analysis
- Share context between them
- Root Cause Agent → Round 2 (refine with Fix Agent's insights)
- Fix Agent → Round 2 (refine with Root Cause's insights)
- Configurable via `BRAINSTORM_ENABLED` env var

#### ✅ Day 4 Done When:
- [ ] Root Cause Agent provides accurate diagnosis for all 3 failure types
- [ ] Fix Agent generates reasonable fix suggestions
- [ ] Brainstorming improves the quality of both agents' output
- [ ] End-to-end: logs → parse → analyze → fix suggestion works

---

### DAY 5 — March 8: Orchestrator + Full Pipeline

**Goal**: Build the Orchestrator brain, connect all agents end-to-end.

#### Step 5.1: Build Orchestrator Agent
Create `healing-engine/agents/orchestrator_agent.py`
- `quick_classify(parsed_logs)` → error type (regex-based, no LLM)
- `select_agents(error_type)` → which agents to activate
- `orchestrate(job, build)` → full tiered flow:
  1. Fetch logs from Jenkins
  2. Run Tier 1 (parallel)
  3. Run Tier 2 + brainstorm
  4. Run Tier 3
  5. Calculate confidence score
  6. Return final result
- Confidence scoring logic (rule-based, no LLM):
  - Both agents agree? +30
  - Known error pattern? +25
  - Single file affected? +15
  - etc.

#### Step 5.2: Build Detection Agent
Create `healing-engine/agents/detection_agent.py`
- Receives webhook POST
- Validates payload
- If `status == FAILURE` → calls `orchestrator.orchestrate()`
- If `status == SUCCESS` → logs and ignores

#### Step 5.3: Wire everything in main.py
```python
POST /webhook/jenkins → detection_agent → orchestrator → agents → result
```

#### Step 5.4: Full integration test
```
1. Trigger "demo-compile-error" in Jenkins
2. Jenkins fails → sends webhook
3. Detection Agent catches it
4. Orchestrator classifies → COMPILATION
5. Log Parser + Git Diff run (Tier 1)
6. Root Cause Agent analyzes (Tier 2)
7. Fix Agent generates fix (Tier 3)
8. Brainstorm refines both
9. Final result printed in console
```

#### ✅ Day 5 Done When:
- [ ] Full pipeline works: Jenkins fail → webhook → agents → result
- [ ] Orchestrator correctly classifies all 3 error types
- [ ] Confidence score calculated for each event
- [ ] Console shows complete healing flow with timing

---

### DAY 6 — March 9: Notifications (Slack + Email) + Database

**Goal**: Send results to Slack and email, store in SQLite.

#### Step 6.1: Set up Slack incoming webhook
```
1. Go to api.slack.com → Create New App
2. Incoming Webhooks → Activate
3. Add Webhook to Workspace → select channel
4. Copy webhook URL → paste in .env
```

#### Step 6.2: Build Slack Service
Create `healing-engine/services/slack_service.py`
- `send_healing_notification(diagnosis)` → formatted Slack blocks
- Rich format: root cause, fix, confidence, retrigger button

#### Step 6.3: Build Email Service
Create `healing-engine/services/email_service.py`
- `send_healing_email(diagnosis)` → HTML email
- Uses aiosmtplib for async SMTP
- Gmail app password for auth

#### Step 6.4: Build Notify Agent
Create `healing-engine/agents/notify_agent.py`
- Takes final result from Orchestrator
- Sends to Slack AND email
- Saves to SQLite database
- Updates dashboard

#### Step 6.5: Build Healing History DB
Create `healing-engine/database/healing_history.py`
- SQLite async operations
- `save_event()`, `get_all_events()`, `get_event()`, `get_stats()`
- Table: id, timestamp, job_name, build_number, error_type, root_cause, suggested_fix, confidence, agents_used

#### ✅ Day 6 Done When:
- [ ] Slack notification appears when build fails
- [ ] Email sent with diagnosis
- [ ] Healing event saved in SQLite
- [ ] Full end-to-end: Jenkins fail → agents → Slack + email + DB

---

### DAY 7 — March 10: Dashboard + Polish + SUBMIT PPT

**Goal**: Build web dashboard, polish everything, submit PPT to architect.

#### Step 7.1: Build Dashboard
Create `healing-engine/dashboard/index.html`, `style.css`, `script.js`
- Healing history timeline (recent events)
- Agent activity feed (which agents ran, timing)
- Build health cards (pass/fail counts)
- Auto-refresh every 5 seconds

#### Step 7.2: Add Dashboard API routes
```python
GET /dashboard          → serve HTML
GET /api/history        → return healing events
GET /api/stats          → return summary stats
GET /api/history/{id}   → return single event detail
```

#### Step 7.3: Final polish
- Error handling (try/except everywhere)
- Structured logging with timestamps
- Clean up console output with emojis for demo visibility

#### Step 7.4: ⭐ SUBMIT PPT TO ARCHITECT
```
Use architect-pitch.html as reference
Create 10-slide PowerPoint
Practice the pitch (5 minutes)
Submit to architect for approval
```

#### ✅ Day 7 Done When:
- [ ] Dashboard shows healing history
- [ ] PPT submitted to architect
- [ ] System runs reliably end-to-end

---

### DAY 8 — March 11: Retrigger Flow + Edge Cases

**Goal**: Add Slack retrigger button, handle edge cases.

#### Step 8.1: Slack Interactive Button
- Add `/slack/actions` endpoint
- When "Retrigger Build" clicked → call jenkins_service.trigger_build()
- Send Slack update: "Build #43 triggered"

#### Step 8.2: Build Result Monitoring
- After retrigger, poll Jenkins for result
- When complete → Slack: "✅ Build #43 passed!" or "❌ Still failing"

#### Step 8.3: Edge Case Handling
- Jenkins unreachable → graceful error in Slack
- LLM timeout → retry once, then send raw error logs
- Slack webhook fails → fall back to email only
- Empty build logs → special "no logs available" message

#### ✅ Day 8 Done When:
- [ ] Retrigger from Slack works
- [ ] Build result feedback appears in Slack
- [ ] Edge cases handled gracefully

---

### DAY 9 — March 12: Demo Recording + History Agent (Bonus)

**Goal**: Record demo video, optionally add History Agent.

#### Step 9.1: Add History Agent (if time permits)
Create `healing-engine/agents/history_agent.py`
- Query SQLite for past similar errors
- If match found → include "past fix" in current diagnosis
- Makes the system "learn" from previous healings

#### Step 9.2: Practice demo run
```
1. Show Jenkins with pipelines
2. Trigger compile-error → watch healing flow → Slack notification
3. Click retrigger → build passes
4. Trigger test-failure → different agents activate → Slack
5. Show dashboard with history
6. Show architecture diagram
```

#### Step 9.3: Record demo video
```
Tool: OBS Studio or Windows Win+G
Duration: 5 minutes max
Narration: follow speaker notes from architect-pitch.html
```

#### ✅ Day 9 Done When:
- [ ] Demo video recorded
- [ ] Video shows at least 2 different failure types being healed
- [ ] Dashboard visible in video

---

### DAY 10 — March 13: Final Polish + SUBMIT DEMO

**Goal**: Final testing, polish video, submit.

#### Step 10.1: Final end-to-end test
- Run all 3 failure types → verify all work
- Check Slack, email, dashboard for each
- Verify confidence scores make sense

#### Step 10.2: Polish demo video
- Trim any dead time
- Ensure audio is clear
- Add title slide at beginning if needed

#### Step 10.3: ⭐ SUBMIT DEMO VIDEO

#### ✅ Day 10 Done When:
- [ ] Demo video submitted
- [ ] All systems working
- [ ] 🎉 DONE!

---

## Quick Reference: API Endpoints

```
POST /webhook/jenkins              → Detection Agent (webhook handler)
POST /webhook/manual               → Manual trigger for testing
POST /slack/actions                 → Handle Slack button clicks
GET  /api/history                   → List all healing events
GET  /api/history/{id}              → Single event detail
GET  /api/stats                     → Summary statistics
GET  /dashboard                     → Web dashboard
GET  /health                        → Health check
GET  /docs                          → FastAPI auto-docs
```

## Quick Reference: Docker Commands

```powershell
# Start everything
docker-compose up -d

# View logs
docker-compose logs -f healing-engine
docker-compose logs -f jenkins

# Restart after code change
docker-compose restart healing-engine

# Pull Ollama model
docker exec -it ollama ollama pull llama3.1

# Stop everything
docker-compose down

# Reset everything (fresh start)
docker-compose down -v
```

---

## Enterprise Integration (For Your Architect)

```
HOW TO CONNECT TO ANY EXISTING PIPELINE:

Option 1: Global Webhook (affects ALL pipelines)
  Jenkins → Manage Jenkins → Configure System → Notification → Add URL
  URL: http://healing-engine:5000/webhook/jenkins
  Effort: 2 minutes

Option 2: Per-Pipeline (selective)
  Each pipeline job → Post-build Actions → HTTP Request
  URL: http://healing-engine:5000/webhook/jenkins
  Effort: 1 minute per pipeline

Option 3: Jenkinsfile post block (5 lines of code)
  post {
      failure {
          httpRequest url: "http://healing-engine:5000/webhook/jenkins",
                     httpMode: 'POST',
                     requestBody: '{"job":"${JOB_NAME}","build":${BUILD_NUMBER},"status":"FAILURE"}'
      }
  }

ZERO changes to pipeline logic. ZERO risk. Plug and play.
```

---

**Let's start building! 🚀**
