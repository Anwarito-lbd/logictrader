# LogicTrader Architecture

## Data flow (top to bottom)

```
┌──────────────────────────────────────────────────────────────────┐
│  apps/web (Next.js 16)                                           │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ React Flow canvas                                          │  │
│  │   ├─ AFLNode (Monaco editor inside)                        │  │
│  │   ├─ IndicatorNode (MACD, RSI, ATR…)                       │  │
│  │   ├─ GateNode (Cross, AndIf, IIf)                          │  │
│  │   └─ ExecutionNode (Buy/Sell/StopLoss)                     │  │
│  │ Zustand store: nodes[], edges[], runState                   │  │
│  └────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬─────────────────────────────────────┘
                             │ POST /graph (JSON)
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│  apps/api (FastAPI)                                              │
│  ┌─────────────┐  ┌──────────────────┐  ┌────────────────────┐  │
│  │ /transpile  │→ │ Lark AFL parser  │→ │ Python AST emitter │  │
│  │ /backtest   │  └──────────────────┘  └─────────┬──────────┘  │
│  │ /optimize   │                                  │              │
│  │ /live       │                                  ▼              │
│  └──────┬──────┘                          graph_runner.py        │
│         │ Celery dispatch                                        │
│         ▼                                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Workers (Celery + Redis broker)                         │    │
│  │   backtest_task  →  core/backtester.py (VectorBT)       │    │
│  │   optimize_task  →  Optuna ↔ core/combo_engine.py       │    │
│  └─────────────────────────────────────────────────────────┘    │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│  core/ (existing trading engine)                                 │
│   data/        ─ parquet loader, redis cache, ccxt fetcher       │
│   meta_models/ ─ 38 signal generators                            │
│   regime/      ─ NEW: HMM/GMM regime classifier                  │
│   backtester   ─ VectorBT walk-forward + Monte Carlo             │
│   aggregator   ─ weighted combination, regime suppression        │
│   brain        ─ confluence gate, Kelly sizing, drawdown breaker │
│   executor     ─ paper/live, ccxt + alpaca + (NEW) MT5           │
│   analytics/   ─ NEW: QuantStats tearsheet wrapper               │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼
                    ┌─────────────────────┐
                    │ Postgres (graphs)   │
                    │ Redis (cache+broker)│
                    │ Parquet (market)    │
                    │ MT5/ccxt (broker)   │
                    └─────────────────────┘
```

## Service topology (docker-compose)

| Service | Port | Role |
|---|---|---|
| `web` | 3000 | Next.js canvas |
| `api` | 8000 | FastAPI router + transpiler |
| `worker` | — | Celery worker (backtest + optimize) |
| `redis` | 6379 | Broker + cache |
| `postgres` | 5432 | Graphs (JSONB), runs, users |
| `live` | — | Headless executor (Phase 8, separate VPS) |

## Decision: why this shape

- **Transpiler bridge** (not LLM-only) → deterministic, fast, testable, no API cost.
- **Lark over hand-rolled parser** → grammar lives in one file, easy to extend.
- **Celery over asyncio queue** → multi-worker scaling for Optuna trials.
- **VectorBT kept** → already integrated, fastest in class for vectorized backtests.
- **Live executor on separate VPS** → UI bugs cannot crash live trading.

## See also

- [docs/afl_grammar.md](docs/afl_grammar.md) — supported AFL subset
- [docs/node_catalog.md](docs/node_catalog.md) — every node type
- [docs/adr/](docs/adr/) — architecture decision records
