# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Crypto Day Trading Arena is a multi-agent cryptocurrency trading simulation platform. AI agents with different strategies trade against live Coinbase market data, with all communication flowing through a Kafka broker. Each component runs as an independent process.

## Commands

**Package manager: `uv`**

```bash
uv sync                          # Install dependencies
```

**Running the system (each component is a separate process):**

```bash
# 1. Stream live market data to Kafka
uv run python coinbase_connector.py --bootstrap-servers <broker-url>

# 2. Deploy shared trading tools + portfolio dashboard
uv run python tools_and_dashboard.py --bootstrap-servers <broker-url>

# 3. Deploy an LLM inference node (one per model/provider)
uv run python deploy_chat_node.py \
    --name <node-name> --model-id <model> \
    --bootstrap-servers <broker-url> --api-key <api-key>

# DeepSeek example (DEEPSEEK_API_KEY is read from .env automatically)
uv run python deploy_chat_node.py \
    --name deepseek --model-id deepseek-chat \
    --base-url https://api.deepseek.com/v1 \
    --bootstrap-servers <broker-url>

# 4. Deploy an agent router (one per agent)
uv run python deploy_router_node.py \
    --name <agent-name> --chat-node-name <node-name> \
    --strategy <strategy> --bootstrap-servers <broker-url>

# 5. (Optional) Watch agent reasoning and tool calls
uv run python response_viewer.py --bootstrap-servers <broker-url>
```

There are no tests or linting commands — this is a runtime-only simulation application.

## Architecture

All components communicate via **Kafka** and run as independent async processes. No component directly calls another; all coordination is message-based.

```
Coinbase WebSocket
       │
       ▼
coinbase_connector.py  ──►  Kafka Broker  ──►  Agent Routers (deploy_router_node.py)
                                  │                     │
                                  │                     ▼
                                  │             Chat Nodes (deploy_chat_node.py)
                                  │                     │
                                  └──►  Tools & Dashboard (tools_and_dashboard.py)
```

### Key modules

| File | Role |
|------|------|
| `trading_tools.py` | Core trading logic: `AccountStore`, `AgentAccount`, portfolio dashboard, `execute_trade`, `get_portfolio`, `calculator` tools |
| `coinbase_consumer.py` | Coinbase WebSocket + REST consumers; `PriceBook` and `CandleBook` for 1/5/15-min candles |
| `coinbase_kafka_connector.py` | Publishes Coinbase ticks to Kafka as `TickerMessage` |
| `deploy_router_node.py` | Agent router: selects strategy prompt, targets a named ChatNode, runs the agentic loop |
| `deploy_chat_node.py` | LLM inference node: wraps OpenAI-compatible providers (OpenAI, DeepSeek, Gemini, etc.) |
| `data_recorder.py` | Optional CSV export: `TradeRow` and `SnapshotRow` schemas, line-buffered writes |
| `response_viewer.py` | Terminal dashboard for real-time agent activity |

### Design patterns

**Agent identity via ToolContext**: Tools are deployed once but serve all agents. Each tool call carries `ctx.agent_name`, which is used to look up the correct `AgentAccount` in the shared `AccountStore`.

**Named ChatNodes for model routing**: Each `deploy_router_node.py` instance targets a specific `--chat-node-name`. Multiple agents can share one ChatNode, or each can have its own — enabling per-agent model selection (e.g., one agent uses Claude, another uses GPT-4o).

**Candle data**: Three timeframes (1m, 5m, 15m) are fetched from Coinbase REST API on a 60-second interval and injected into agent prompts alongside live ticker data.

**Strategies**: Defined as system prompt strings in `deploy_router_node.py`. Available: `default`, `momentum`, `brainrot`, `scalper`.

**Data recording**: `DataRecorder` in `data_recorder.py` writes to `data/` directory (gitignored). Enabled via `--record-data` flag on `tools_and_dashboard.py`. Schema documented in `docs/csv-data-recording.md`.

## Environment

Copy `.env.example` to `.env`. Required vars:
- `KAFKA_BOOTSTRAP_SERVERS` — broker URL (required for all components)
- `OPENAI_API_KEY` or `DEEPSEEK_API_KEY` — for `deploy_chat_node.py` (or pass via `--api-key`); both are checked automatically

See `CLI_REFERENCE.md` for all CLI flags.
