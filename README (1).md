# AlphaQuant — Impulse Flow Alpha AI Agent

> *"Don't read the strategy. Ask it."*

**Team Codex | IIT (BHU) Varanasi | Nasiko AI Agent Hackathon 2026**

---

## Table of Contents

- [Overview](#overview)
- [The Strategy](#the-strategy)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [The Five Tools](#the-five-tools)
- [A2A Protocol Compliance](#a2a-protocol-compliance)
- [Installation & Setup](#installation--setup)
- [Running the Agent](#running-the-agent)
- [API Usage](#api-usage)
- [Sample Requests & Responses](#sample-requests--responses)
- [Environment Variables](#environment-variables)
- [AgentCard Registration](#agentcard-registration)
- [Endpoints](#endpoints)
- [Stress Testing Results](#stress-testing-results)
- [Customisation](#customisation)
- [Troubleshooting](#troubleshooting)
- [Performance Metrics](#performance-metrics)

---

## Overview

AlphaQuant is a **production-grade, modular AI agent** built for the Nasiko AI Agent Hackathon. It wraps the *Impulse Flow Alpha* quantitative trading strategy — a pattern-recognition-based long/short system backtested on four years of BTC-USDT daily OHLC data — inside a fully compliant Nasiko A2A JSON-RPC 2.0 service.

The agent exposes **five structured analytical tools** accessible through plain English queries:

| # | Tool | Purpose |
|---|---|---|
| 1 | `explain_strategy` | Plain-English breakdown of the ICB pattern |
| 2 | `analyze_performance` | Interprets all backtest metrics with qualitative ratings |
| 3 | `evaluate_risk` | Summarises all 4 stress tests and forward-looking risks |
| 4 | `generate_signal` | Runs live signal engine → LONG / SHORT / FLAT |
| 5 | `run_backtest` | Full live backtest on any OHLC dataset |

**Key properties:**
- Every tool is deterministic — identical input always produces identical output
- Zero hardcoded responses in Tools 4 and 5 — all computed live from raw data
- Full A2A JSON-RPC 2.0 compliance — tested live against the Nasiko gateway
- One-command Docker deployment
- Accepts OHLC data as JSON arrays or CSV text blocks

---

## The Strategy

### Impulse Flow Alpha — Conceptual Intuition

Price often moves in a predictable **three-phase structure** that repeats across trending assets:

```
IMPULSE  ──►  CONSOLIDATION  ──►  BREAKOUT
  │                 │                 │
Sharp           Shallow           Continuation
directional     pullback          in original
move            holds             direction
                structure
```

The strategy enters a trade **only when all three phases are confirmed**, significantly reducing false signals and improving entry quality.

### Signal Parameters

| Parameter | Long Setup | Short Setup |
|---|---|---|
| Impulse bar return | `+0.13%` to `+0.27%` | `-7.0%` to `-2.0%` |
| Maximum retrace | `≤ 3.0%` | `≤ 1.0%` |
| Breakout range | `+1.9%` to `+5.0%` | `-6.30%` to `-5.80%` |
| Lookback window | 40 bars | 40 bars |
| Transaction cost | 0.15% per trade | 0.15% per trade |

### Backtest Performance (BTC-USDT, 2020–2024)

| Metric | Value |
|---|---|
| Net Return | **4307.12%** |
| Annualised CAGR | **157.32%** |
| Sharpe Ratio | **1.9659** |
| Sortino Ratio | **3.0001** |
| Calmar Ratio | **5.1007** |
| Maximum Drawdown | **30.84%** |
| Total Days | 1,462 |
| Round-trip Trades | 14 |
| Starting Capital | $1,000,000 |

---

## Architecture

AlphaQuant follows a clean **three-layer architecture** with strict separation of concerns:

```
┌─────────────────────────────────────────┐
│         User / Nasiko Gateway           │
│         POST /  (JSON-RPC 2.0)          │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│        Layer 3 — A2A Executor           │
│      openai_agent_executor.py           │
│  • Parses A2A JSON-RPC 2.0 requests     │
│  • Wraps responses in compliant format  │
│  • Handles errors with proper envelopes │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│        Layer 2 — Agent Router           │
│           openai_agent.py               │
│  • Priority-ordered keyword routing     │
│  • GPT-4o-mini LLM fallback             │
│  • JSON + CSV OHLC data extraction      │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│        Layer 1 — Core Toolset           │
│           agent_toolset.py              │
│  • 5 async tools                        │
│  • Zero external API dependency         │
│  • NumPy/Pandas computations            │
│  • Direct port of backtester logic      │
└─────────────────────────────────────────┘
```

### Layer Responsibilities

**Layer 1 — Toolset (`agent_toolset.py`)**
- Contains all 5 tools as `async` Python methods in `QuantTradingToolset`
- All strategy parameters defined as module-level constants
- Signal engine is a direct, faithful port of the original backtester notebook
- All metrics (Sharpe, Sortino, Calmar, drawdown, win rate, profit factor) computed from scratch using NumPy
- Can run completely offline — zero external API calls

**Layer 2 — Agent (`openai_agent.py`)**
- Priority-ordered keyword intent router (`_detect_intent`)
- `run_backtest` checked before `generate_signal` to prevent routing conflicts
- GPT-4o-mini fallback at temperature 0.3, max 1024 tokens
- `_extract_ohlc` parses OHLC from JSON arrays or CSV blocks in free text
- Auto-escalates `generate_signal` to `run_backtest` when 30+ bars supplied

**Layer 3 — Executor (`openai_agent_executor.py`)**
- Receives raw A2A JSON-RPC 2.0 dicts from FastAPI
- Extracts user message from `params.message.parts`
- Wraps response in compliant envelope: `contextId`, `artifactId`, `status`, `artifacts`
- Returns clean JSON-RPC 2.0 error envelopes on failure — never raw exceptions

---

## Project Structure

```
AlphaQuant/
│
├── src/
│   ├── __init__.py                  # Package initialiser
│   ├── __main__.py                  # FastAPI app — 3 endpoints
│   ├── agent_toolset.py             # All 5 tools (Layer 1) — 14 KB
│   ├── openai_agent.py              # Intent routing + LLM (Layer 2) — 6 KB
│   └── openai_agent_executor.py     # A2A compliance (Layer 3) — 5 KB
│
├── AgentCard.json                   # Nasiko registry metadata (5 skills)
├── Dockerfile                       # Python 3.11-slim container
├── docker-compose.yml               # One-command deployment
├── pyproject.toml                   # Project dependencies
├── README.md                        # This file
└── CUSTOMIZE.md                     # Extension guide
```

---

## The Five Tools

### Tool 1: `explain_strategy`

**Trigger keywords:** `explain`, `how does`, `what is impulse`, `describe strategy`

**Input:** Optional `strategy_text` string for additional context

**Output:** Structured multi-section report covering:
- Three-phase ICB conceptual intuition
- Exact numerical thresholds for long and short setups (read live from constants)
- Execution assumptions: lookback window, transaction cost, position types

**Example query:** `"Explain the Impulse Flow Alpha strategy."`

---

### Tool 2: `analyze_performance`

**Trigger keywords:** `performance`, `metrics`, `sharpe`, `sortino`, `calmar`, `annualised`, `drawdown`

**Input:** Seven numerical parameters (net return, CAGR, Sharpe, Sortino, Calmar, max DD, days, trades) — defaults to official BTC-USDT backtest results

**Output:** Structured report with:
- Return metrics section
- Risk-adjusted metrics with qualitative ratings (Excellent / Good / Acceptable / Poor)
- Drawdown assessment
- Contextual interpretation commentary

**Rating thresholds:**
- Sharpe: `≥2.0` = Excellent, `≥1.5` = Good, `≥1.0` = Acceptable
- Sortino: `≥3.0` = Excellent, `≥2.0` = Good
- Calmar: `≥3.0` = Excellent, `≥2.0` = Good

---

### Tool 3: `evaluate_risk`

**Trigger keywords:** `risk`, `stress`, `robust`, `overfitting`, `monte carlo`, `perturbation`

**Input:** None

**Output:** Structured report covering:
- All 4 stress test results with specific numerical findings
- 5 forward-looking risks listed explicitly
- Conclusion with recommended next steps

---

### Tool 4: `generate_signal`

**Trigger keywords:** `signal`, `generate signal`, `ohlc`, `predict`, `trade now`, `go long`, `go short`

**Input:** JSON array of OHLC dicts (minimum 5 bars, maximum recommended 29 bars)

```json
[
  {"Date": "2024-01-01", "Open": 42000, "High": 43000, "Low": 41500, "Close": 42800},
  {"Date": "2024-01-02", "Open": 42800, "High": 44000, "Low": 42500, "Close": 43500}
]
```

**Output:** Signal report containing:
- Signal label: `LONG ▲`, `SHORT ▼`, or `FLAT —`
- Number of bars analysed
- Latest close price
- Trigger detail: impulse bar index, return %, retrace %, breakout %

**Note:** Automatically escalates to `run_backtest` when 30+ bars supplied.

---

### Tool 5: `run_backtest`

**Trigger keywords:** `backtest`, `run backtest`, `full backtest`, `simulate`, `test this data`

**Input:** OHLC data as JSON array **or** CSV text block (minimum 10 bars)

**CSV format accepted:**
```
Date,Open,High,Low,Close
2020-01-01,7200,7400,7100,7350
2020-01-02,7350,7600,7300,7450
```

**Output:** Full backtest report containing:
- Input parameters summary (bars loaded, date range, starting capital, TCS)
- Return metrics: Net Return, Final Capital, Annualised CAGR, Total Days
- Risk-adjusted metrics: Sharpe, Sortino, Calmar with ratings, Max Drawdown
- Trade statistics: Total trades, Win rate, Gross profit/loss, Profit factor, Avg win/loss
- Trade log: Last 10 OPEN/CLOSE events with date, direction, price, PnL%, capital
- Dynamic interpretation based on actual computed values

---

## A2A Protocol Compliance

AlphaQuant strictly follows the Nasiko A2A JSON-RPC 2.0 specification.

### Request Format

```json
{
  "jsonrpc": "2.0",
  "id": "req-001",
  "method": "tasks/send",
  "params": {
    "id": "task-001",
    "message": {
      "role": "user",
      "parts": [
        {
          "kind": "text",
          "text": "Your query here"
        }
      ]
    }
  }
}
```

### Response Format

```json
{
  "jsonrpc": "2.0",
  "id": "req-001",
  "result": {
    "id": "task-001",
    "contextId": "<uuid>",
    "kind": "task",
    "status": {
      "state": "completed",
      "timestamp": "2026-03-29T22:24:24.522514+00:00"
    },
    "artifacts": [
      {
        "artifactId": "<uuid>",
        "name": "alpha_quant_response",
        "parts": [
          {
            "kind": "text",
            "text": "=== Response content here ==="
          }
        ]
      }
    ]
  }
}
```

### Error Response Format

```json
{
  "jsonrpc": "2.0",
  "id": "req-001",
  "error": {
    "code": -32000,
    "message": "Execution error: <description>",
    "data": {
      "timestamp": "2026-03-29T22:24:24+00:00"
    }
  }
}
```

---

## Installation & Setup

### Prerequisites

- Docker Desktop 4.0+ (with Docker Compose V2)
- OpenAI API key (`sk-...`)
- Port 10000 available on your machine

### Clone / Download

```bash
# If using git
git clone <your-repo-url>
cd AlphaQuant

# Or navigate to your project folder
cd path/to/Alphaquant_AI_agent
```

### Environment Setup

Create a `.env` file in the project root (optional — you can also pass inline):

```env
OPENAI_API_KEY=sk-your-key-here
```

---

## Running the Agent

### Option 1 — Docker (Recommended, One Command)

**Windows PowerShell:**
```powershell
$env:OPENAI_API_KEY="sk-your-key-here"; docker compose up --build -d
```

**Windows CMD:**
```cmd
set OPENAI_API_KEY=sk-your-key-here && docker compose up --build -d
```

**Linux / Mac:**
```bash
OPENAI_API_KEY=sk-your-key-here docker compose up --build -d
```

The `-d` flag runs the container in detached (background) mode.

### Option 2 — Local Python (Without Docker)

```bash
# Install dependencies
pip install fastapi "uvicorn[standard]" openai pandas numpy python-dotenv

# Set environment variable
export OPENAI_API_KEY=sk-your-key-here   # Linux/Mac
$env:OPENAI_API_KEY="sk-..."             # PowerShell

# Run the agent
python -m src
```

### Verify the Agent is Running

```powershell
Invoke-RestMethod -Uri "http://localhost:10000/health"
```

Expected output:
```
status  agent
------  -----
ok      AlphaQuant
```

---

## API Usage

All requests go to `POST http://localhost:10000/` with `Content-Type: application/json`.

### PowerShell (Windows)

```powershell
Invoke-RestMethod -Method POST `
  -Uri "http://localhost:10000/" `
  -ContentType "application/json" `
  -Body '{"jsonrpc":"2.0","id":"req-001","method":"tasks/send","params":{"id":"task-001","message":{"role":"user","parts":[{"kind":"text","text":"YOUR QUERY HERE"}]}}}' `
  | ConvertTo-Json -Depth 10
```

### curl (Linux/Mac)

```bash
curl -X POST http://localhost:10000/ \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":"req-001","method":"tasks/send","params":{"id":"task-001","message":{"role":"user","parts":[{"kind":"text","text":"YOUR QUERY HERE"}]}}}'
```

---

## Sample Requests & Responses

### 1. Explain Strategy

**Request text:** `"Explain the Impulse Flow Alpha strategy."`

**Response text (truncated):**
```
=== Impulse Flow Alpha — Strategy Explanation ===

CONCEPT
-------
The strategy exploits a 3-phase market structure...

LONG SETUP (buy signal)
-----------------------
  • Impulse bar return : 0.13% – 0.27%
  • Max retrace        : 3.0%
  • Breakout range     : 1.9% – 5.0%
...
```

---

### 2. Analyze Performance

**Request text:** `"Analyze the performance metrics."`

**Response text (truncated):**
```
=== Performance Analysis — Impulse Flow Alpha ===

RETURN METRICS
--------------
  Net Return        : 4307.12%
  Annualised Return : 157.32% CAGR

RISK-ADJUSTED METRICS
---------------------
  Sharpe  Ratio : 1.9659  → Good
  Sortino Ratio : 3.0001  → Excellent
  Calmar  Ratio : 5.1007  → Excellent
...
```

---

### 3. Evaluate Risk

**Request text:** `"Evaluate the risks and stress tests."`

**Response text (truncated):**
```
=== Risk Evaluation — Impulse Flow Alpha ===

STRESS TEST RESULTS
-------------------
1. Trade Connectivity Test (20% Random Dropout)
   • Avg simulated annualised return : ~114%
   ...
2. Monte-Carlo Shuffle (2 000 iterations)
   • Actual Max DD : 30.84%  |  Avg shuffled DD : 41.74%
   ...
```

---

### 4. Generate Signal

**Request text:**
```
Generate signal [{"Date":"2024-01-01","Open":42000,"High":43000,"Low":41500,"Close":42800},...]
```

**Response text:**
```
=== Impulse Flow Alpha — Signal Report ===

  Bars analysed     : 5
  Latest Close      : 42800.0000
  Signal            : FLAT —
  Trigger detail    : No pattern matched within lookback window.
```

---

### 5. Run Backtest

**Request text:**
```
Run backtest [{"Date":"2020-01-01","Open":7200,"High":7400,"Low":7100,"Close":7350},...]
```

**Response text (truncated):**
```
=== Impulse Flow Alpha — Full Backtest Report ===

INPUT PARAMETERS
----------------
  Bars loaded       : 10
  Starting capital  : $1,000,000.00
  Transaction cost  : 0.15% per side

RETURN METRICS
--------------
  Net Return        : 0.0000%
  Annualised Return : 0.0000% CAGR

TRADE STATISTICS
----------------
  Total Trades      : 0  (round-trips)
  Win Rate          : 0.00%
...
```

---

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `OPENAI_API_KEY` | ✅ Yes | Your OpenAI API key for LLM fallback |

---

## AgentCard Registration

The `AgentCard.json` file registers AlphaQuant with the Nasiko gateway registry. It declares:

- Agent name, description, version, and URL
- Default input/output modes (`text/plain`)
- Capabilities (streaming: false, pushNotifications: false)
- All 5 skills with IDs, descriptions, tags, and example queries

Served automatically at:
```
GET http://localhost:10000/.well-known/agent.json
```

---

## Endpoints

| Method | Path | Description |
|---|---|---|
| `POST` | `/` | Main A2A endpoint — all agent queries |
| `GET` | `/.well-known/agent.json` | AgentCard for Nasiko registry discovery |
| `GET` | `/health` | Liveness check |

---

## Stress Testing Results

| Test | Method | Result | Conclusion |
|---|---|---|---|
| Trade Connectivity | 20% random dropout, 1000 simulations | Avg return ~114% (vs 157%) | Alpha broadly distributed |
| Monte Carlo Shuffle | 2000 trade sequence shuffles | Actual DD 30.84% < Avg DD 41.74% | Timing is genuine |
| Gaussian Diffusion | Thousands of lognormal paths | Real curve > 95th percentile | Returns not from luck |
| Parameter Perturbation | ±5% on all 11 params | Most = zero sensitivity | Strategy robust |

---

## Customisation

See `CUSTOMIZE.md` for full details. Quick reference:

### Adding a New Tool

1. Add `async def new_tool(self, ...) -> str` to `QuantTradingToolset` in `agent_toolset.py`
2. Register it in `get_tools()` dict
3. Add keyword cluster to `OpenAIAgent._detect_intent()` in `openai_agent.py`
4. Add a skill entry to `AgentCard.json`
5. Add routing logic in `OpenAIAgent.run()` — no other files need changes

### Changing Strategy Parameters

All 11 parameters are constants at the top of `agent_toolset.py`:

```python
IMPULSE_MIN_LONG   = 0.0013
IMPULSE_MAX_LONG   = 0.0027
BREAKOUT_MIN_LONG  = 0.019
# ... etc
```

Changing these values automatically updates `explain_strategy` output and all signal/backtest computations.

---

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| `docker compose up` fails with exit code 2 | Old Dockerfile with `pip install -e .` | Use the fixed Dockerfile that installs packages directly |
| `curl` not found in PowerShell | PowerShell aliases curl to Get-Process | Use `curl.exe` or `Invoke-RestMethod` instead |
| Agent returns `"Error in AlphaQuant agent"` | Missing or invalid `OPENAI_API_KEY` | Set `OPENAI_API_KEY` environment variable |
| Port 10000 already in use | Another process using the port | Change port in `docker-compose.yml` and `__main__.py` |
| `>>` appears in PowerShell | Unclosed quote in command | Press Ctrl+C and retype with matching quotes |
| 0 trades in backtest | Test data doesn't match ICB pattern | Expected — strategy is selective. Use full BTC dataset |

---

## Performance Metrics

AlphaQuant was evaluated against all four Nasiko judging criteria:

| Criterion | Assessment | Evidence |
|---|---|---|
| **Problem Value** | Strong | Real quant use case; 4-year backtest + 4 stress tests |
| **Gateway Compliance** | Full | Live-tested; all A2A fields correct and verified |
| **Code Reliability** | Strong | 3-layer separation; full input validation; exception handling |
| **Output Predictability** | Full | All 5 tools deterministic; identical input = identical output |

---

## Team

**Team Codex**
Indian Institute of Technology (BHU), Varanasi
Nasiko AI Agent Hackathon | 2026

---

*AlphaQuant — Where quantitative intelligence meets conversational simplicity.*