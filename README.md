# LogicTrader — Alpha Factory

Visual node-based quant research, backtesting, optimization, sentiment, regime,
agentic strategy synthesis, and execution platform.

**KPI**: $1,000 / week revenue by 2026-06-30.
**Primary asset**: XAUUSD (Gold). Secondary: Qatar Stock Exchange (Sun–Thu trading).

## Stack at a glance

| Layer | What ships |
|---|---|
| Frontend (`frontend/`) | Next.js + React Flow + Zustand + Monaco Editor + TradingView Lightweight Charts |
| Backend (`backend/app/`) | FastAPI · SQLAlchemy async · Alembic |
| Workers | Celery + Redis broker, Flower task UI |
| Data | TimescaleDB (Postgres) · Polars ingestion · pandas-market-calendars (Qatar Sun–Thu calendar) |
| Transpiler | Lark grammar → AST → Python ([backend/app/transpiler](backend/app/transpiler)) |
| Backtest | NumPy/Pandas vectorized engine + bid/ask spread reality filter ([backend/app/backtesting/spread.py](backend/app/backtesting/spread.py)) |
| Optimization | Optuna walk-forward (70/30 split) |
| Regime / ML | hmmlearn · scikit-learn · FinBERT (transformers) · Darts forecasting · statsmodels |
| Research synthesis | LangGraph + AutoGen workflows · arXiv + GitHub sources · sandboxed promotion |
| Execution | Adapters for MT5 · DWX-ZeroMQ · Interactive Brokers (`ib_insync`) · CCXT · FIX (stub) · kill-switch |
| Legacy | `legacy_core/` — original 38 meta-models, brain, executor (kept for reference) |

## Quick start

```bash
task setup              # uv venv + install backend deps
task ingest-synthetic   # write synthetic XAUUSD parquet to data/parquet/
task smoke              # end-to-end check (no docker required)
task dev                # full docker stack (postgres + redis + api + worker + flower)
```

URLs once `task dev` is up:
- API docs: http://localhost:8000/docs
- Flower:   http://localhost:5555
- Frontend: http://localhost:3000 (run `task web` separately if not in compose)

## Recommended first run inside the UI

1. Open the canvas (`task web` then http://localhost:3000)
2. Connect: **Data Source → AFL Strategy → Risk → Order**
3. Click **Compile** — the graph posts to `POST /api/v1/graphs/compile`
4. Click **Backtest** — sandboxed, returns Sharpe/Sortino/MaxDD + equity curve
5. Try **Research → Synthesize** with a plain-English thesis
   (e.g. *"Buy XAUUSD when 50 SMA crosses above 200 SMA and DXY is dropping"*)
6. Strategy is auto-promoted to the node library only if Sharpe ≥ 1.2 and ≥ 25 trades

## Repo map

```
LogicTrader/
├── backend/               FastAPI + workers + transpiler + adapters
│   ├── app/api/v1/        REST routes (graphs, backtests, optimize, research, ml, ingest, assets, execution)
│   ├── app/backtesting/   indicators, spread reality filter, vectorbt-style engine
│   ├── app/data/          calendar (Qatar Sun-Thu), Polars ingestion, paths, API source
│   ├── app/execution/     gateway + MT5/DWX/IB/CCXT/FIX adapters + kill_switch + signal_router
│   ├── app/feeds/         Finnhub client, news
│   ├── app/forecasting/   Darts + statsmodels adapters
│   ├── app/indicators/    pandas-ta adapter
│   ├── app/ml/            regime (HMM), sentiment (FinBERT)
│   ├── app/models/        graph, asset, session, strategy_library
│   ├── app/nodes/         code_file_node (sandboxed Python upload runner)
│   ├── app/optimization/  Optuna walk-forward
│   ├── app/research/      agentic synthesis, LangGraph + AutoGen workflows, arXiv/GitHub sources
│   └── app/transpiler/    Lark grammar (afl.lark), parser, afl_to_python, graph_compiler
├── frontend/              Next.js + React Flow canvas
│   └── src/components/nodes/  AflNode, RiskNode, OrderNode, ResearchNode,
│                              FileDataNode, ApiDataNode, CodeFileNode, DataSourceNode
├── legacy_core/           Original LogicTrader engine (38 meta-models, brain, executor)
├── infra/postgres/        TimescaleDB init + hypertable creation
├── shared/schemas/        sample_graph.json + JSON contracts
├── sample_data/           qse_sample.csv + sample_strategy_node.py
├── scripts/               smoke_test, ingest, regime, backtest
├── docs/                  REPO_MAP, FILE_CODE_API_NODES, FINAL_STATUS, adr/
├── REPOS.md               Catalog of upstream reference repos (use, don't clone)
├── MONEY_KPI.md           North-star revenue target
├── RUNBOOK.md             Daily operator loop
└── Taskfile.yml           task setup | dev | test | smoke | validate | dev | api | worker | web
```

## Non-negotiables

- `task validate` must pass before commit
- All work ties to [`MONEY_KPI.md`](MONEY_KPI.md)
- No secrets in code — env vars only (see `.env.example`)
- No live-fund testing — paper / sandbox first
- Reversible commits, feature-flagged risky changes

## Reference repos

This project depends on (and learns from) a curated set of upstream projects.
We use them as libraries or reference architecture — we do **not** vendor or
clone their code. See [REPOS.md](REPOS.md).
