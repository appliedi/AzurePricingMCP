# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Azure Pricing MCP Server (v4.0.0) — a Python MCP server providing AI assistants with real-time Azure retail pricing data, plus Databricks, GitHub, and Azure OpenAI PTU pricing. Wraps the public Azure Retail Prices API (`https://prices.azure.com/api/retail/prices`, no auth required for pricing tools). Spot VM and orphaned resource tools require Azure AD authentication.

## Common Commands

```bash
# Setup
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"

# Run server
python -m azure_pricing_mcp                          # stdio mode (default, for VS Code/Claude Desktop)
python -m azure_pricing_mcp --transport http --port 8080  # HTTP/SSE mode (for Docker)

# Tests
pytest tests/ -v                    # All unit tests
pytest tests/ -m "not integration"  # Skip integration tests (what CI runs)
pytest tests/test_azure_pricing.py  # Single test file
pytest tests/ -k "test_name"        # Single test by name

# Code quality (all configured for line-length=120)
black src/ tests/                   # Format
ruff check --fix src/ tests/        # Lint with auto-fix
mypy src/                           # Type check
pre-commit run --all-files          # All checks at once
```

## Architecture

**Modular service-layer design:**
- `server.py` — thin orchestrator: creates client, services, wires handlers, runs transport
- `client.py` — `AzurePricingClient`: async HTTP with retry/rate-limit logic
- `config.py` — all constants, service name mappings, GitHub/Databricks static pricing data
- `auth.py` — `AzureCredentialManager`: Azure AD auth with graceful degradation
- `handlers.py` — `ToolHandlers` class (inherits `DatabricksHandlers`, `GitHubPricingHandlers` mixins): routes tool calls to services, formats responses
- `formatters.py` — all Markdown response formatting functions
- `tools.py` — all 18 tool definitions (`get_tool_definitions()`)
- `models.py` — Pydantic/dataclass domain models

**Service layer (`services/`):**
- `pricing.py` — `PricingService`: price search, compare, estimate, region recommend, RI pricing
- `sku.py` — `SKUService`: SKU discovery with fuzzy matching
- `retirement.py` — `RetirementService`: VM series deprecation warnings
- `spot.py` — `SpotService`: eviction rates, price history, eviction simulation (requires Azure auth)
- `orphaned.py` — `OrphanedResourcesService`: detects 11 types of unused resources (requires Azure auth)
- `ptu.py` / `ptu_models.py` — `PTUService`: Azure OpenAI PTU sizing for 24 models
- `databricks.py` — `DatabricksService`: DBU pricing, cost estimation, workload comparison
- `github_pricing.py` — `GitHubPricingService`: plans, Copilot, Actions, security pricing

**Subpackages** (`databricks/`, `github_pricing/`): each has own `tools.py`, `handlers.py`, `formatters.py`

**Data flow:** MCP Client → ToolHandlers → Service class → AzurePricingClient → Azure API

**Transport modes:** stdio (default, local clients) or HTTP/SSE (Starlette + uvicorn, for Docker)

**18 MCP tools:** `azure_price_search`, `azure_price_compare`, `azure_cost_estimate`, `azure_discover_skus`, `azure_sku_discovery`, `azure_region_recommend`, `azure_ri_pricing`, `get_customer_discount`, `spot_eviction_rates`, `spot_price_history`, `simulate_eviction`, `find_orphaned_resources`, `azure_ptu_sizing`, `databricks_dbu_pricing`, `databricks_cost_estimate`, `databricks_compare_workloads`, `github_pricing`, `github_cost_estimate`

## Key Patterns

- **All I/O is async** — `aiohttp.ClientSession` managed at server level (not per-call)
- **SKU normalization** — `normalize_sku_name()` in `services/pricing.py` generates search variants
- **Fuzzy service matching** — `SERVICE_NAME_MAPPINGS` in `config.py` maps user terms to Azure service names
- **Discount handling** — `show_with_discount` flag + `_resolve_discount()` in handlers for consistent discount application
- **Rate limit retry** — 3 attempts with exponential backoff on HTTP 429
- **Graceful auth degradation** — Spot/orphaned tools return helpful error messages when Azure auth is missing
- **Handler mixins** — `DatabricksHandlers` and `GitHubPricingHandlers` are mixin classes inherited by `ToolHandlers`
- **Tests mock the Azure API** — no real API calls in unit tests; integration tests marked `@pytest.mark.integration`

## Code Style

- Python 3.10+ with type hints (mypy enforced)
- Black + Ruff formatting, 120 char line length
- Ruff rules: E, W, F, I, B, C4, UP
