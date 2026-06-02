## 📚 AI Learning Accelerator Agent

[![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/)
[![LangGraph](https://img.shields.io/badge/LangGraph-latest-purple.svg)](https://langchain-ai.github.io/langgraph/)
[![Slack](https://img.shields.io/badge/Slack-Bot-4A154B.svg)](https://slack.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A **personal AI tutor** built with LangGraph that creates personalized
curricula, teaches with analogies and code examples, quizzes using
spaced repetition (SM-2), and tracks genuine understanding — delivered
daily via **Slack DM** at 7 PM.

> Built for AI engineers learning topics like Fine-Tuning, LoRA, MLOps,
> LangGraph, and Cloud Deployment.
> **Development cost: $0/day (Ollama). Production cost: ~$0.03/day.**

---

## 📋 Table of Contents

- [Why This Exists](#-why-this-exists)
- [How It Works](#-how-it-works)
- [Architecture](#-architecture)
- [Agent Design](#-agent-design)
- [Cost Strategy](#-cost-strategy)
- [Observability](#-observability-langfuse)
- [Spaced Repetition](#-spaced-repetition-sm-2)
- [Tech Stack](#-tech-stack)
- [Project Structure](#-project-structure)
- [Setup](#-setup)
- [Daily Experience](#-daily-experience-slack)
- [Technical Decisions](#-technical-decisions)
- [Future Enhancements](#-future-enhancements)

---

## 🤔 Why This Exists

Most learning apps test memory. This tests understanding.

```
What other apps do:          What this does:
━━━━━━━━━━━━━━━━━━━━━        ━━━━━━━━━━━━━━━━━━━━━━━━━━━━
"What is LoRA?"              "Given this architecture,
"Define fine-tuning"          how would you apply LoRA?
                              What rank would you choose?"

Generic curriculum           Adapts to YOUR weak areas
Manual flashcards            Auto-generated every lesson
No cost control              Hard $5/month budget cap
Recall = forgetting           SM-2 = retained for months
```

---

## ⚙️ How It Works

```
You send to Slack: "/learn Fine-Tuning with LoRA"
                        │
                        ▼
            ┌───────────────────────┐
            │  Prior Knowledge Check│ ← 3 diagnostic Q&A
            └──────────┬────────────┘
                       │
            ┌───────────────────────┐
            │   Curriculum Builder  │ ← 14-day plan (CACHED)
            └──────────┬────────────┘
                       │
            ┌───────────────────────┐
            │    Resource Finder    │ ← Best free resources (CACHED)
            └──────────┬────────────┘
                       │
            ┌───────────────────────┐   Every day 7 PM via Slack:
            │     Lesson Writer     │ ← Concept + analogy + code
            └──────────┬────────────┘   (CACHED per topic+day)
                       │
            ┌───────────────────────┐
            │    Quiz Generator     │ ← 3-level quiz (NOT cached)
            └──────────┬────────────┘
                       │
            ┌───────────────────────┐
            │  Understanding Eval   │ ← Evaluates YOUR answer
            └──────────┬────────────┘   (NOT cached)
                       │
            ┌───────────────────────┐
            │    Flashcard Maker    │ ← SM-2 cards (CACHED)
            └──────────┬────────────┘
                       │
                  Slack DM reply
```

---

## 🏗️ Architecture

```
Slack DM (interface — send lesson, receive answers)
        │
        ▼
┌──────────────────────────────────────────────────┐
│                LangGraph Workflow                 │
│                                                  │
│  Orchestrator → routes by session type           │
│       │                                          │
│  new_topic      review          quiz_only        │
│       │            │                │            │
│  Prior          Cards Due        Quiz            │
│  Knowledge      Today            Generator       │
│  Check                           Only            │
│       │                                          │
│  Curriculum Builder  ← checks Redis cache first  │
│       │                                          │
│  Resource Finder     ← checks Redis cache first  │
│       │                                          │
│  Lesson Writer       ← checks Redis cache first  │
│       │                                          │
│  Quiz Generator      ← always fresh (no cache)  │
│       │                                          │
│  Understanding Eval  ← always fresh (no cache)  │
│       │                                          │
│  Flashcard Maker     ← checks Redis cache first  │
│       │                                          │
│  END  (or retry lesson if score < 0.5)           │
└──────────────────────────────────────────────────┘
        │
        ▼
┌──────────────┐  ┌─────────────────┐  ┌──────────────┐
│   SQLite DB  │  │ Langfuse Cloud  │  │ Redis Cache  │
│  (progress,  │  │ (every LLM call │  │ (saves LLM   │
│  flashcards, │  │  traced + costed│  │  calls on    │
│  API costs)  │  │  in dashboard)  │  │  repeated    │
│              │  │                 │  │  content)    │
└──────────────┘  └─────────────────┘  └──────────────┘
```

---

## 🤖 Agent Design

### Orchestrator
- **Model:** `gpt-4o-mini` (prod) / Ollama (dev)
- **Job:** Classify session type — new lesson vs review vs quiz-only
- **Cache:** No — routing must be fresh every time

### Curriculum Builder
- **Model:** `gpt-4o` (prod) / Ollama (dev)
- **Job:** Build 14-day personalized learning plan
- **Cache:** Yes — 30 days (same topic + level = same plan)
- **Why cached:** A curriculum for "LoRA Fine-Tuning, intermediate"
  doesn't change — regenerating it wastes $0.018 every time

### Resource Finder
- **Model:** `gpt-4o-mini` + Tavily search (prod) / Ollama (dev)
- **Job:** Find 5 best free resources for current week
- **Cache:** Yes — 7 days per topic + week combination

### Prior Knowledge Checker
- **Model:** `gpt-4o-mini` (prod) / Ollama (dev)
- **Job:** 3 diagnostic questions → adjusts curriculum depth
- **Cache:** No — must reflect current user knowledge

### Lesson Writer
- **Model:** `gpt-4o` (prod) / Ollama (dev)
- **Job:** Daily lesson with concept + analogy + code example
- **Cache:** Yes — 30 days per topic + day
- **Max output:** 600 tokens (Slack-digestible)

### Quiz Generator
- **Model:** `gpt-4o-mini` (prod) / Ollama (dev)
- **Job:** 3-level quiz (recall → apply → synthesize)
- **Cache:** No — must vary to prevent answer memorization

### Understanding Evaluator
- **Model:** `gpt-4o` (prod) / Ollama (dev)
- **Job:** Evaluate your Slack reply — not right/wrong, but WHY
- **Cache:** No — evaluates your specific answer

### Flashcard Maker
- **Model:** `gpt-4o-mini` (prod) / Ollama (dev)
- **Job:** Auto-generate SM-2 flashcards from lesson key concepts
- **Cache:** Yes — 30 days per concept list

---

## 💰 Cost Strategy

Four layers work together to keep costs near zero.

### Layer 1 — Ollama for Development (Free)

```
ALL development and testing uses Ollama locally.
Zero API calls. Zero cost. Identical code path.

# Install once:
brew install ollama
ollama pull llama3.1:8b

# Set in .env:
APP_ENV=development   → uses Ollama (free)
APP_ENV=production    → uses OpenAI (paid)

Switch to GPT-4o ONLY for:
✅ Final quality check before releasing
✅ Actual daily use once satisfied with quality
```

### Layer 2 — Smart Model Selection (Saves ~80%)

```
Not all tasks need GPT-4o.

GPT-4o (expensive — use sparingly):
  lesson_writer       ← core teaching, quality critical
  curriculum_builder  ← drives everything downstream
  understanding_eval  ← nuanced judgment needed

GPT-4o-mini (cheap — use everywhere else):
  orchestrator        ← binary classification
  quiz_generator      ← structured extraction
  resource_finder     ← ranking + filtering
  prior_knowledge     ← simple Q&A
  flashcard_maker     ← text extraction

Result: ~80% cost reduction vs using GPT-4o everywhere
```

### Layer 3 — Redis Cache (Eliminates Repeat Calls)

```
Cached content (never regenerated unless expired):
  Curriculum plan     → 30-day TTL
  Lesson content      → 30-day TTL per topic+day
  Flashcard content   → 30-day TTL per concept
  Resource lists      → 7-day TTL per topic+week

NOT cached (always fresh):
  Quiz questions      → must vary or you memorize answers
  Answer evaluation   → evaluates YOUR specific response
  Routing decisions   → must reflect current session state

Practical impact:
  Day 1: Curriculum generated → $0.018 LLM call
  Day 2-30: Curriculum served from cache → $0.000
  Saves ~$0.50/month on curriculum alone
```

### Layer 4 — Hard Spending Caps

```
CostGuard (in your code):
  MONTHLY_BUDGET_USD = 5.00
  → Blocks ALL LLM calls if exceeded
  → Sends Slack warning at $3.00
  → Resets on the 1st of each month

OpenAI Dashboard (external safety net):
  platform.openai.com → Settings → Billing → Usage Limits
  Soft limit: $3.00/month → Email warning
  Hard limit: $5.00/month → API rejects calls
  → Even if CostGuard has a bug, you can't be billed above $5
```

### Monthly Cost Summary

| Scenario | Daily | Monthly | $10 Credit Lasts |
|---|---|---|---|
| Development (Ollama) | $0.00 | $0.00 | Forever |
| Production smart routing | $0.03 | $0.90 | ~11 months |
| Production all GPT-4o | $0.08 | $2.40 | ~4 months |
| Cache hit day (common) | $0.01 | $0.30 | ~33 months |

---

## 📊 Observability — Langfuse

Every LLM call is automatically traced in Langfuse.
Access your dashboard at **cloud.langfuse.com** (free tier).

### Setup (One Time)

```bash
# 1. Create free account at cloud.langfuse.com
# 2. Create a project named "learning-accelerator"
# 3. Copy Public Key + Secret Key to .env
# 4. That's it — tracing is automatic
```

### What You See

```
cloud.langfuse.com → Your Project → Traces

Session: abc-123  |  April 12, 7:02 PM  |  $0.034
─────────────────────────────────────────────────────
├── orchestrator         42ms    $0.001   gpt-4o-mini
├── curriculum_builder    0ms    $0.000   CACHE HIT ✅
├── lesson_writer         1.2s   $0.012   gpt-4o
│   └── [click: full prompt + response visible]
├── quiz_generator       310ms   $0.001   gpt-4o-mini
└── understanding_eval   890ms   $0.002   gpt-4o
```

### Cost Tracking Over Time

```
COST tab → Monthly breakdown:
lesson_writer      $0.84/month  ← most expensive
curriculum_builder $0.00/month  ← fully cached
quiz_generator     $0.04/month
Total:             $1.02/month  ← well under $5 cap

SCORES tab → Understanding over time:
LoRA Fine-Tuning:  day1=0.61  day4=0.74  day14=0.89
MLOps basics:      day1=0.71  day7=0.88
→ See exactly which topics you're improving on
```

---

## 🔁 Spaced Repetition (SM-2)

Same algorithm as Anki — proven to maximize retention.

```
You answer a flashcard. Rate yourself 0-5:
0 = Complete blackout
3 = Correct but struggled
5 = Perfect, instant recall

SM-2 schedules next review:

Card: "What is LoRA rank?"

Day 1  → Answered 4 (correct) → Review in 7 days
Day 8  → Answered 5 (perfect) → Review in 23 days
Day 31 → Answered 3 (struggled) → Review in 15 days
Day 46 → Answered 0 (forgot!) → RESET → Review tomorrow

Result after 3 months:
Easy cards  → reviewed every 60+ days
Hard cards  → reviewed every 2-3 days
= Maximum retention, minimum time wasted
```

Flashcards are **auto-generated** after every lesson — you never
create cards manually. The Flashcard Maker Agent extracts key
concepts from the lesson and creates front/back pairs automatically.

---

## 🧠 Tech Stack

| Component | Dev (Free) | Prod (Paid) | Purpose |
|---|---|---|---|
| Orchestration | LangGraph | LangGraph | Agent graph |
| LLM | Ollama llama3.1:8b | GPT-4o / mini | Generation |
| Search | Tavily free tier | Tavily | Resources |
| Interface | Slack SDK | Slack SDK | DM delivery |
| Cache | Redis (Docker) | Redis (Docker) | LLM cache |
| Database | SQLite | SQLite | Progress + costs |
| Tracing | Langfuse free | Langfuse free | Observability |
| Scheduler | APScheduler | APScheduler | 7 PM trigger |
| Config | Pydantic Settings | Pydantic Settings | Env vars |

---

## 📁 Project Structure

```
ai-learning-accelerator/
├── app/
│   ├── agents/
│   │   ├── orchestrator.py
│   │   ├── curriculum_builder.py  # cached
│   │   ├── resource_finder.py     # cached
│   │   ├── prior_knowledge.py
│   │   ├── lesson_writer.py       # cached
│   │   ├── quiz_generator.py      # NOT cached
│   │   ├── understanding_eval.py  # NOT cached
│   │   └── flashcard_maker.py     # cached
│   ├── graph/
│   │   ├── state.py               # LearningState
│   │   ├── workflow.py            # LangGraph graph
│   │   └── routing.py             # conditional routing
│   ├── spaced_repetition/
│   │   ├── sm2.py                 # SM-2 algorithm
│   │   └── scheduler.py
│   ├── db/
│   │   ├── models.py
│   │   ├── session.py
│   │   └── repository.py
│   ├── bot/
│   │   ├── slack_bot.py           # Slack DM send + poll
│   │   └── formatters.py          # Slack mrkdwn formatting
│   ├── core/
│   │   ├── config.py              # env vars + model switching
│   │   ├── logging.py
│   │   ├── cost_guard.py          # budget enforcement
│   │   ├── cache.py               # Redis LLM cache
│   │   └── observability.py       # Langfuse helpers
│   └── prompts/
│       ├── curriculum.py
│       ├── lesson.py
│       ├── quiz.py
│       └── evaluation.py
├── tests/
├── scripts/
│   └── init_db.py
├── .env.example
├── Dockerfile
├── docker-compose.yml
├── pyproject.toml
├── CLAUDE.md
└── README.md
```

---

## ⚙️ Setup

### Prerequisites

```
Required:
✅ Python 3.11+
✅ Slack workspace + Bot token (api.slack.com)
✅ Ollama installed (ollama.com) — for free dev testing
✅ Redis (via Docker)
✅ Langfuse account (cloud.langfuse.com — free)

For production only:
✅ OpenAI API key (~$10 = ~11 months of daily use)
✅ Tavily API key (free tier)

IMPORTANT: Set hard limit in OpenAI dashboard BEFORE
adding your API key: platform.openai.com →
Settings → Billing → Usage Limits → Hard limit: $5
```

### 1. Clone the repo

```bash
git clone https://github.com/hema625/ai-learning-accelerator.git
cd ai-learning-accelerator
```

### 2. Install dependencies

```bash
python -m venv .venv
source .venv/bin/activate
pip install -e .
```

### 3. Set up Ollama (free dev LLM)

```bash
# Install Ollama from https://ollama.com
ollama pull llama3.1:8b
# Ollama runs at http://localhost:11434
# Zero cost — runs completely offline
```

### 4. Configure environment

```bash
cp .env.example .env
```

Fill in `.env`:

```env
# Start in development mode — uses Ollama (free)
APP_ENV=development

# Slack (required from day 1)
SLACK_BOT_TOKEN=xoxb-your-token
SLACK_USER_ID=U07XXXXXXXX

# Langfuse (required from day 1 — free tier)
LANGFUSE_PUBLIC_KEY=pk-your-key
LANGFUSE_SECRET_KEY=sk-your-key

# OpenAI (only needed when APP_ENV=production)
OPENAI_API_KEY=sk-your-key

# Cost controls
MONTHLY_BUDGET_USD=5.00
WARN_THRESHOLD_USD=3.00
```

### 5. Start Redis

```bash
docker-compose up redis -d
```

### 6. Initialize database

```bash
python scripts/init_db.py
```

### 7. Run the app

```bash
# Development (uses Ollama — free)
APP_ENV=development python -m app.main

# Production (uses OpenAI — paid)
APP_ENV=production python -m app.main
```

### 8. Start learning

Send in your Slack DM to the bot:

```
/learn Fine-Tuning with LoRA
```

---

## 📱 Daily Experience — Slack

```
8:00 AM — Review reminder:
─────────────────────────────────────────────────
🤖 Learning Agent DM:

🔁 *3 flashcards due for review*
Topic: Fine-Tuning with LoRA

*Card 1:*
What does the LoRA rank parameter 'r' control?
→ Reply with your answer

─────────────────────────────────────────────────
7:00 PM — Daily lesson:
─────────────────────────────────────────────────
🤖 Learning Agent DM:

📚 *Day 4 of 14 — LoRA Rank Parameter*
Streak: 4 days 🔥 | Understanding: 73% → 81% ✅

*Concept:*
The rank 'r' controls how many dimensions your
low-rank matrices have. Higher r = more capacity
but more GPU memory...

*Analogy:*
Think of it like image compression — r=4 keeps
the 4 most important "directions" of updates.

```python
lora_config = LoraConfig(
    r=16,           # start here for most tasks
    lora_alpha=32,  # usually 2x the rank
    target_modules=["q_proj", "v_proj"]
)
```

*Key takeaways:*
• r=4 simple tasks, r=16 complex, r=64 rarely needed
• Higher r ≠ always better — watch memory usage
• lora_alpha controls effective learning rate scaling

🧪 *Quiz — reply with your answers:*
Q1: If r=8 and hidden_dim=1024, how many
    params does one LoRA layer add?
Q2: Why might r=64 hurt more than help?
Q3: How does LoRA rank relate to PCA?

─────────────────────────────────────────────────
You reply: "Q1: 16,384 Q2: overfitting + memory Q3: ..."

🤖 Learning Agent:
✅ Q1 correct! Adding: formula is 2 × r × d = 16,384
⚠️ Q2 partially right — also mention inference cost
✅ Q3 excellent connection!

Understanding score: 0.81 (+0.08 from last session)
3 new flashcards created and scheduled 🃏
```

---

## 🏗️ Technical Decisions

**1. Why Slack over email or Telegram?**
Slack stays open on most developers' desktops all day.
A DM notification at 7 PM appears in a context already
associated with productive work. Email requires context
switching to a different app and intent. Telegram requires
a separate download. The path of least resistance determines
whether a daily habit forms or dies in week 2.

**2. Why Ollama for development?**
Every development iteration — testing a prompt change,
fixing a bug, trying a new agent — would cost $0.02-0.05
with GPT-4o. Over 50 test runs during development, that
is $1-2.50 before the app even works correctly. Ollama
runs llama3.1:8b locally with zero cost and near-GPT-4o-mini
quality for testing purposes. The model switching is one
environment variable — no code changes.

**3. Why Redis cache for LLM responses?**
A curriculum plan for "LoRA Fine-Tuning, intermediate,
5 hours/week" is deterministic — the same inputs produce
the same optimal plan. Regenerating it every session costs
$0.018 and adds 980ms of latency for zero benefit. At 30
daily sessions per month that is $0.54 wasted on one agent
alone. The cache pays for itself immediately. Quiz questions
and answer evaluations are intentionally not cached — they
must vary and be specific to the user's actual response.

**4. Why SM-2 over a fixed review schedule?**
Fixed schedules (review every 3 days) ignore what you
actually know. SM-2 adapts per card — cards you know well
grow to 30-60 day intervals, freeing daily review time for
concepts you are actually forgetting. After 30 days the
retention difference is measurable: SM-2 achieves ~90%
retention vs ~50-60% for fixed schedules at equal review time.

**5. Why three separate cost layers?**
Any single layer can fail. CostGuard has a bug — the
OpenAI dashboard cap catches it. The dashboard cap isn't
set — CostGuard catches it. Ollama is used in dev —
neither cap matters. Defense in depth is standard
engineering practice applied to API billing.

**6. Why log understanding scores to Langfuse?**
SQLite tracks raw scores. Langfuse shows them as a time
series with correlation to other signals — which agents
ran, how long they took, what the lesson was. Seeing
"understanding drops to 0.61 every time lesson_writer
latency exceeds 2s" reveals a content quality problem
at long generation times. You cannot find this without
unified tracing.

---

## 🔮 Future Enhancements

- [ ] Voice lesson delivery via Slack voice note (ElevenLabs TTS)
- [ ] Multimodal quizzes — analyze code screenshots you share
- [ ] GitHub integration — quiz on concepts in your recent commits
- [ ] Export flashcards to Anki format for offline review
- [ ] Multi-topic concurrent learning with priority scheduling
- [ ] Streamlit dashboard for weekly progress visualization

---

## 🔗 Related Projects

Part of a production AI engineering portfolio:

| Project | Description | Link |
|---|---|---|
| `clinical-rag-assistant` | Production RAG + eval layer | [GitHub](https://github.com/hema625/clinical-rag-assistant) |
| `stateful-medical-assistant` | LangGraph guardrails + state | [GitHub](#) |
| `medical-multimodal-rag` | Text + image RAG pipeline | [GitHub](#) |
| `ai-learning-accelerator` | This repo | — |
| `medical-multi-agent` | Healthcare agent orchestration | [GitHub](#) |

---

## 👤 Author

**Hema S** | [GitHub](https://github.com/hema625)

---

*Last Updated: April 2026*
