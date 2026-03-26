# FinAlly — AI Trading Workstation

An AI-powered trading workstation that streams live market data, lets users trade a simulated portfolio, and integrates an LLM chat assistant that can analyze positions and execute trades. Built entirely by coding agents as a capstone for an agentic AI coding course.

## Architecture

Single Docker container, single port (8000):

- **Frontend**: Next.js static export served by FastAPI
- **Backend**: FastAPI (Python/uv) with SQLite
- **Market data**: GBM simulator (default) or Polygon.io via Massive API
- **AI**: LiteLLM via OpenRouter with structured outputs for trade execution

## Project Status

- **Market data subsystem**: Complete (simulator, Massive client, SSE streaming, price cache)
- **Database, API, portfolio, frontend, AI chat, Docker**: Not yet built

## Development

```bash
# Backend
cd backend
uv sync --extra dev
uv run --extra dev pytest -v

# Market data demo
cd backend
uv run market_data_demo.py
```

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `OPENROUTER_API_KEY` | Yes | OpenRouter API key for LLM chat |
| `MASSIVE_API_KEY` | No | Polygon.io key for real market data (simulator used if absent) |
| `LLM_MOCK` | No | Set `true` for deterministic mock LLM responses |

See [planning/PLAN.md](planning/PLAN.md) for the full specification.

## License

See [LICENSE](LICENSE).
