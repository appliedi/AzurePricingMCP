# V4 Modular Refactor + New Features Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Refactor the monolithic v2.1.0 codebase into the upstream v4.0.0 modular architecture and add 10 new MCP tools (Spot VM, Orphaned Resources, PTU Sizing, Databricks, GitHub Pricing).

**Architecture:** Split monolithic `server.py` (~1460 lines) into a service-layer architecture: `client.py` (HTTP), `config.py` (constants), `formatters.py` (response formatting), `tools.py` (tool definitions), `auth.py` (Azure AD), and a `services/` package with dedicated service classes. New tools live in `databricks/` and `github_pricing/` subpackages with their own handlers/formatters/tools.

**Tech Stack:** Python 3.10+, mcp>=1.0.0, aiohttp, pydantic, azure-identity (new), azure-mgmt-* (new)

**Reference:** All new code is ported from upstream https://github.com/msftnadavbh/AzurePricingMCP v4.0.0

---

## Chunk 1: Foundation — HTTP Client, Config, and Dependencies

### Task 1: Update pyproject.toml with new dependencies and version

**Files:**
- Modify: `pyproject.toml`

- [ ] **Step 1: Update version and add Azure SDK dependencies**

Add to `dependencies` list:
```
"azure-identity>=1.15.0",
"azure-mgmt-compute>=30.0.0",
"azure-mgmt-network>=30.0.0",
"azure-mgmt-resource>=20.0.0",
"azure-mgmt-subscription>=3.0.0",
"azure-mgmt-web>=10.1.0",
"azure-mgmt-resourcegraph>=8.0.1",
"azure-mgmt-costmanagement>=4.0.1",
```

Update version to `"4.0.0"`. Do NOT change `project.scripts` entry point yet — the `run()` function won't exist until Task 17. Add mypy override for `azure.*` imports:
```toml
[[tool.mypy.overrides]]
module = "azure.*"
ignore_missing_imports = true
```

- [ ] **Step 2: Install updated dependencies**

Run: `pip install -e ".[dev]"`
Expected: Clean install with azure SDK packages

- [ ] **Step 3: Commit**

```bash
git add pyproject.toml
git commit -m "feat: bump to v4.0.0, add Azure SDK dependencies"
```

---

### Task 2: Create `config.py` — extract constants and mappings

**Files:**
- Create: `src/azure_pricing_mcp/config.py`

- [ ] **Step 1: Create config.py**

Extract from `server.py`:
- `AZURE_PRICING_BASE_URL`, `DEFAULT_API_VERSION`, `MAX_RESULTS_PER_REQUEST`
- `MAX_RETRIES`, `RATE_LIMIT_RETRY_BASE_WAIT`, `DEFAULT_CUSTOMER_DISCOUNT`
- `SERVICE_NAME_MAPPINGS` (entire dict)

Add new constants from upstream:
- `AZURE_MANAGEMENT_SCOPE = "https://management.azure.com/.default"`
- `AZURE_RESOURCE_GRAPH_URL = "https://management.azure.com/providers/Microsoft.ResourceGraph/resources?api-version=2022-10-01"`
- `SPOT_PERMISSIONS` dict (maps tool names to required Azure permissions)
- `RETIRED_VM_SERIES` dict (maps deprecated series to alternatives)
- `GITHUB_PLANS`, `GITHUB_COPILOT_PLANS`, `GITHUB_ACTIONS_RUNNERS`, `GITHUB_SECURITY_PRODUCTS`, `GITHUB_ADDONS` (static pricing data)
- `GITHUB_PRICING_DATA_VERSION = "2025-06-01"` (date of pricing data snapshot)
- `GITHUB_PRODUCT_ALIASES` dict (maps user terms like "ci/cd" to canonical names)

Reference: upstream `config.py` for exact values of all dicts.

- [ ] **Step 2: Verify imports work**

Run: `python -c "from azure_pricing_mcp.config import AZURE_PRICING_BASE_URL, SERVICE_NAME_MAPPINGS; print('OK')"`
Expected: `OK`

- [ ] **Step 3: Commit**

```bash
git add src/azure_pricing_mcp/config.py
git commit -m "refactor: extract constants and mappings to config.py"
```

---

### Task 3: Create `client.py` — extract HTTP client with retry logic

**Files:**
- Create: `src/azure_pricing_mcp/client.py`

- [ ] **Step 1: Create client.py**

Extract from `server.py`:
- `AzurePricingClient` class (renamed from the HTTP parts of `AzurePricingServer`)
- `__aenter__` / `__aexit__` for aiohttp session management
- `_make_request()` method with retry/rate-limit logic
- `fetch_prices()` method that calls the Azure Retail Prices API

Key interface:
```python
class AzurePricingClient:
    async def __aenter__(self): ...  # creates aiohttp.ClientSession
    async def __aexit__(self, ...): ...  # closes session
    async def fetch_prices(self, filter_expression: str, ...) -> dict: ...
```

Import constants from `config.py` instead of defining them inline.

- [ ] **Step 2: Write test**

Run: `python -c "from azure_pricing_mcp.client import AzurePricingClient; print('OK')"`
Expected: `OK`

- [ ] **Step 3: Commit**

```bash
git add src/azure_pricing_mcp/client.py
git commit -m "refactor: extract HTTP client to client.py"
```

---

### Task 4: Create `auth.py` — Azure AD credential management

**Files:**
- Create: `src/azure_pricing_mcp/auth.py`

- [ ] **Step 1: Create auth.py**

Port upstream `auth.py` directly:
- `AzureCredentialManager` class with lazy `azure-identity` import
- `_check_azure_identity_available()` function
- `get_credential_manager()` singleton factory
- Methods: `is_authenticated()`, `get_token()`, `get_initialization_error()`
- Static helpers: `get_required_permissions_message()`, `get_authentication_help_message()`

This gracefully degrades when azure-identity is not installed — returns an error message instead of crashing.

- [ ] **Step 2: Verify import**

Run: `python -c "from azure_pricing_mcp.auth import get_credential_manager; print('OK')"`
Expected: `OK`

- [ ] **Step 3: Commit**

```bash
git add src/azure_pricing_mcp/auth.py
git commit -m "feat: add Azure AD credential management (auth.py)"
```

---

## Chunk 2: Service Layer — Core Services

### Task 5: Create `services/` package with `pricing.py` and `sku.py`

**Files:**
- Create: `src/azure_pricing_mcp/services/__init__.py`
- Create: `src/azure_pricing_mcp/services/pricing.py`
- Create: `src/azure_pricing_mcp/services/sku.py`

- [ ] **Step 1: Create services/__init__.py**

```python
"""Services package for Azure Pricing MCP Server."""
from .pricing import PricingService
from .sku import SKUService

__all__ = ["PricingService", "SKUService"]
```

(Will be expanded as more services are added.)

- [ ] **Step 2: Create services/pricing.py**

Extract from `server.py`:
- `normalize_sku_name()` function
- `PricingService` class containing:
  - `search_azure_prices()`, `compare_prices()`, `estimate_costs()`, `recommend_regions()`, `get_ri_pricing()`, `get_customer_discount()`
  - All fuzzy matching logic (`search_azure_prices_with_fuzzy_matching()`, `_find_similar_services()`)
- Constructor takes `AzurePricingClient` and `RetirementService` (for now, pass `None` for retirement)
- Uses `config.py` for constants/mappings instead of module-level globals

- [ ] **Step 3: Create services/sku.py**

Extract from `server.py`:
- `SKUService` class containing:
  - `discover_skus()`, `discover_service_skus()` methods
- Constructor takes `PricingService`

- [ ] **Step 4: Verify imports**

Run: `python -c "from azure_pricing_mcp.services import PricingService, SKUService; print('OK')"`
Expected: `OK`

- [ ] **Step 5: Commit**

```bash
git add src/azure_pricing_mcp/services/
git commit -m "refactor: extract PricingService and SKUService to services package"
```

---

### Task 6: Create `services/retirement.py` — VM retirement warnings

**Files:**
- Create: `src/azure_pricing_mcp/services/retirement.py`
- Modify: `src/azure_pricing_mcp/services/pricing.py` (wire retirement into constructor)
- Modify: `src/azure_pricing_mcp/services/__init__.py`

- [ ] **Step 1: Create retirement.py**

Port upstream `services/retirement.py`:
- `RetirementService` class
- Fetches live retirement data from Microsoft documentation
- `check_retirement_status(sku_name)` — returns retirement warning if SKU is deprecated
- `RETIRED_VM_SERIES` data from config.py maps series to recommended alternatives
- Constructor takes `AzurePricingClient`

- [ ] **Step 2: Update services/__init__.py**

Add `RetirementService` to imports and `__all__`.

- [ ] **Step 3: Update PricingService constructor**

Wire `RetirementService` into `PricingService` so price searches include retirement warnings.

- [ ] **Step 4: Commit**

```bash
git add src/azure_pricing_mcp/services/retirement.py src/azure_pricing_mcp/services/__init__.py
git commit -m "feat: add VM retirement warnings service"
```

---

### Task 7: Create `services/spot.py` — Spot VM tools

**Files:**
- Create: `src/azure_pricing_mcp/services/spot.py`

- [ ] **Step 1: Create spot.py**

Port upstream `services/spot.py`:
- `SpotService` class with methods:
  - `get_eviction_rates(skus, locations)` — queries Azure Resource Graph for eviction rate data
  - `get_price_history(sku, location, os_type)` — retrieves up to 90 days of Spot pricing history
  - `simulate_eviction(vm_resource_id)` — triggers eviction simulation on a Spot VM
- All methods use `AzureCredentialManager` for auth
- Returns helpful error messages when auth is missing

- [ ] **Step 2: Update services/__init__.py**

Add `SpotService` to imports and `__all__`.

- [ ] **Step 3: Commit**

```bash
git add src/azure_pricing_mcp/services/spot.py src/azure_pricing_mcp/services/__init__.py
git commit -m "feat: add Spot VM service (eviction rates, price history, simulation)"
```

---

### Task 8: Create `services/orphaned.py` — Orphaned resource detection

**Files:**
- Create: `src/azure_pricing_mcp/services/orphaned.py`

- [ ] **Step 1: Create orphaned.py**

Port upstream `services/orphaned.py`:
- `OrphanedResourcesService` class
- `find_orphaned_resources(days, all_subscriptions)` method
- Scans 11 resource types: unattached disks, unused public IPs, empty App Service Plans, empty SQL Elastic Pools, Application Gateways without backends, unattached NAT Gateways, empty Load Balancers, unlinked Private DNS Zones, disconnected Private Endpoints, Virtual Network Gateways without IP configs, unlinked DDoS Plans
- Pulls 60-day cost data via Azure Cost Management API
- Uses `AzureCredentialManager` for auth

- [ ] **Step 2: Update services/__init__.py**

Add `OrphanedResourcesService` to imports and `__all__`.

- [ ] **Step 3: Commit**

```bash
git add src/azure_pricing_mcp/services/orphaned.py src/azure_pricing_mcp/services/__init__.py
git commit -m "feat: add orphaned resource detection service"
```

---

### Task 9: Create `services/ptu.py` and `services/ptu_models.py` — PTU sizing

**Files:**
- Create: `src/azure_pricing_mcp/services/ptu.py`
- Create: `src/azure_pricing_mcp/services/ptu_models.py`

- [ ] **Step 1: Create ptu_models.py**

Port upstream `services/ptu_models.py`:
- `PTU_MODEL_TABLE` dict — specifications for 24 models (gpt-5.x, gpt-4.1, o3, o4-mini, Llama, DeepSeek)
- Each entry has: `input_tpm_per_ptu`, `output_tpm_per_ptu`, `min_ptu`, `ptu_increment`, `deployment_types`, `max_ptu`

- [ ] **Step 2: Create ptu.py**

Port upstream `services/ptu.py`:
- `PTUService` class
- `estimate_ptu_sizing(model, deployment_type, rpm, avg_input_tokens, avg_output_tokens, ...)` method
- Calculates required PTUs based on workload shape with official rounding rules
- Optionally fetches live $/PTU/hr pricing via `AzurePricingClient`
- Constructor takes `AzurePricingClient`

- [ ] **Step 3: Update services/__init__.py**

Add `PTUService` to imports and `__all__`.

- [ ] **Step 4: Commit**

```bash
git add src/azure_pricing_mcp/services/ptu.py src/azure_pricing_mcp/services/ptu_models.py src/azure_pricing_mcp/services/__init__.py
git commit -m "feat: add Azure OpenAI PTU sizing service"
```

---

### Task 10: Create `services/databricks.py` — Databricks DBU pricing

**Files:**
- Create: `src/azure_pricing_mcp/services/databricks.py`

- [ ] **Step 1: Create databricks.py**

Port upstream `services/databricks.py`:
- `DatabricksService` class
- `get_dbu_pricing(workload_type, tier, region, currency)` — fetches DBU rates from Azure Retail Prices API
- `estimate_dbu_cost(workload_type, dbu_count, hours_per_day, ...)` — monthly/annual cost projection
- `compare_workloads(workload_types, tier, regions, ...)` — side-by-side comparison
- `_resolve_workload_type()` helper with alias matching
- Constructor takes `AzurePricingClient`

- [ ] **Step 2: Update services/__init__.py**

Add `DatabricksService` to imports and `__all__`.

- [ ] **Step 3: Commit**

```bash
git add src/azure_pricing_mcp/services/databricks.py src/azure_pricing_mcp/services/__init__.py
git commit -m "feat: add Databricks DBU pricing service"
```

---

### Task 11: Create `services/github_pricing.py` — GitHub pricing

**Files:**
- Create: `src/azure_pricing_mcp/services/github_pricing.py`

- [ ] **Step 1: Create github_pricing.py**

Port upstream `services/github_pricing.py`:
- `GitHubPricingService` class
- `get_pricing(product, copilot_plan)` — returns pricing catalog from static data in config.py
- `estimate_cost(users, plan, copilot_plan, actions_minutes, ...)` — monthly cost estimation
- `_resolve_product()` helper with alias matching via `GITHUB_PRODUCT_ALIASES`
- No constructor dependencies (uses static config data)

- [ ] **Step 2: Update services/__init__.py**

Add `GitHubPricingService` to imports and `__all__`.

- [ ] **Step 3: Commit**

```bash
git add src/azure_pricing_mcp/services/github_pricing.py src/azure_pricing_mcp/services/__init__.py
git commit -m "feat: add GitHub pricing service"
```

---

## Chunk 3: Subpackages — Databricks and GitHub Pricing

### Task 12: Create `databricks/` subpackage (tools, handlers, formatters)

**Files:**
- Create: `src/azure_pricing_mcp/databricks/__init__.py`
- Create: `src/azure_pricing_mcp/databricks/tools.py`
- Create: `src/azure_pricing_mcp/databricks/handlers.py`
- Create: `src/azure_pricing_mcp/databricks/formatters.py`

- [ ] **Step 1: Create databricks/__init__.py**

```python
"""Databricks pricing subpackage."""
from .handlers import DatabricksHandlers
from .tools import get_databricks_tool_definitions

__all__ = ["DatabricksHandlers", "get_databricks_tool_definitions"]
```

- [ ] **Step 2: Create databricks/tools.py**

Port upstream — defines 3 `Tool` objects:
- `databricks_dbu_pricing`
- `databricks_dbu_cost_estimate`
- `databricks_workload_compare`

Returns list via `get_databricks_tool_definitions()`.

- [ ] **Step 3: Create databricks/formatters.py**

Port upstream — 3 formatter functions:
- `format_databricks_dbu_pricing_response(result)` — Markdown table of DBU rates
- `format_databricks_cost_estimate_response(result)` — cost breakdown with Photon info
- `format_databricks_compare_workloads_response(result)` — comparison table with savings

- [ ] **Step 4: Create databricks/handlers.py**

Port upstream — `DatabricksHandlers` mixin class with:
- `handle_databricks_dbu_pricing(args)`
- `handle_databricks_cost_estimate(args)`
- `handle_databricks_compare_workloads(args)`
Each calls the corresponding `DatabricksService` method and formats via formatters.

- [ ] **Step 5: Commit**

```bash
git add src/azure_pricing_mcp/databricks/
git commit -m "feat: add Databricks pricing subpackage (tools, handlers, formatters)"
```

---

### Task 13: Create `github_pricing/` subpackage (tools, handlers, formatters)

**Files:**
- Create: `src/azure_pricing_mcp/github_pricing/__init__.py`
- Create: `src/azure_pricing_mcp/github_pricing/tools.py`
- Create: `src/azure_pricing_mcp/github_pricing/handlers.py`
- Create: `src/azure_pricing_mcp/github_pricing/formatters.py`

- [ ] **Step 1: Create github_pricing/__init__.py**

```python
"""GitHub pricing subpackage."""
from .handlers import GitHubPricingHandlers
from .tools import get_github_pricing_tool_definitions

__all__ = ["GitHubPricingHandlers", "get_github_pricing_tool_definitions"]
```

- [ ] **Step 2: Create github_pricing/tools.py**

Port upstream — defines 2 `Tool` objects:
- `github_pricing` (catalog lookup with product/copilot_plan filters)
- `github_cost_estimate` (cost calculator with users, plan, actions, codespaces, etc.)

Returns list via `get_github_pricing_tool_definitions()`.

- [ ] **Step 3: Create github_pricing/formatters.py**

Port upstream — 2 formatter functions:
- `format_github_pricing_response(result)` — Markdown catalog with sections
- `format_github_cost_estimate_response(result)` — cost breakdown table

- [ ] **Step 4: Create github_pricing/handlers.py**

Port upstream — `GitHubPricingHandlers` mixin class with:
- `handle_github_pricing(args)`
- `handle_github_cost_estimate(args)`
Each creates `GitHubPricingService` on demand and formats via formatters.

- [ ] **Step 5: Commit**

```bash
git add src/azure_pricing_mcp/github_pricing/
git commit -m "feat: add GitHub pricing subpackage (tools, handlers, formatters)"
```

---

## Chunk 4: Formatters, Tools, and Wiring

### Task 14: Create `formatters.py` — extract response formatting from handlers

**Files:**
- Create: `src/azure_pricing_mcp/formatters.py`

- [ ] **Step 1: Create formatters.py**

Extract all Markdown formatting logic from current `handlers.py` into standalone functions:
- `format_price_search_response(result)` — from `_handle_price_search`
- `format_price_compare_response(result)` — from `_handle_price_compare`
- `format_region_recommend_response(result)` — from `_handle_region_recommend`
- `format_cost_estimate_response(result)` — from `_handle_cost_estimate`
- `format_discover_skus_response(result)` — from `_handle_discover_skus`
- `format_sku_discovery_response(result)` — from `_handle_sku_discovery`
- `format_customer_discount_response(result)` — from `_handle_customer_discount`
- `format_ri_pricing_response(result)` — from `_handle_ri_pricing`
- New: `format_spot_eviction_rates_response(result)`
- New: `format_spot_price_history_response(result)`
- New: `format_simulate_eviction_response(result)`
- New: `format_orphaned_resources_response(result)`
- New: `format_ptu_sizing_response(result)`

Each function takes a result dict and returns a formatted Markdown string.

- [ ] **Step 2: Commit**

```bash
git add src/azure_pricing_mcp/formatters.py
git commit -m "refactor: extract response formatters to formatters.py"
```

---

### Task 15: Create `tools.py` — extract tool definitions

**Files:**
- Create: `src/azure_pricing_mcp/tools.py`

- [ ] **Step 1: Create tools.py**

Port upstream `tools.py`:
- `get_tool_definitions() -> list[Tool]` function
- Contains all 13 core tool definitions (8 existing + 5 new: spot_eviction_rates, spot_price_history, simulate_eviction, find_orphaned_resources, azure_ptu_sizing)
- Imports and appends Databricks and GitHub tool definitions:
  ```python
  from .databricks.tools import get_databricks_tool_definitions
  from .github_pricing.tools import get_github_pricing_tool_definitions
  ```
- Each tool has full `inputSchema` with descriptions, types, defaults, enums

- [ ] **Step 2: Commit**

```bash
git add src/azure_pricing_mcp/tools.py
git commit -m "refactor: extract tool definitions to tools.py"
```

---

### Task 16: Rewrite `handlers.py` AND `server.py` atomically

> **CRITICAL:** These two files must be rewritten together — handlers.py depends on the new service classes, and server.py must create those services and wire them in. Doing one without the other leaves the codebase broken.

**Files:**
- Modify: `src/azure_pricing_mcp/handlers.py`
- Modify: `src/azure_pricing_mcp/server.py`
- Modify: `src/azure_pricing_mcp/__init__.py`
- Modify: `src/azure_pricing_mcp/__main__.py` (verify still works)
- Modify: `pyproject.toml` (update entry point to `server:run`)

- [ ] **Step 1: Rewrite handlers.py**

Replace current function-based handlers with upstream's class-based `ToolHandlers`:
- `ToolHandlers(DatabricksHandlers, GitHubPricingHandlers)` — inherits mixin handlers
- Constructor takes: `PricingService`, `SKUService`, optional `DatabricksService`, `SpotService`, `OrphanedResourcesService`, `PTUService`
- Sets `self._databricks_service`, `self._github_pricing_service` etc. — these are the attributes the mixin handlers (from Tasks 12-13) expect on `self`
- Each handler method calls the service, then formats via `formatters.py`
- `_resolve_discount(args)` helper for consistent discount handling with `show_with_discount` flag
- Route tool names to handler methods in `handle_tool_call(name, args)`

Key change: no more `async with pricing_server:` per call — session is managed at server level.

- [ ] **Step 2: Rewrite server.py**

Replace monolithic server with thin orchestrator:
- `AzurePricingServer` class — creates `AzurePricingClient`, all services, and `ToolHandlers`
- `__aenter__` / `__aexit__` — manages client session lifecycle
- `create_server()` — returns `(Server, AzurePricingServer)` tuple
  - Registers `list_tools` handler (from `tools.py`)
  - Registers `call_tool` handler (delegates to `ToolHandlers`)
- `main()` — async entry point with CLI args (--transport, --host, --port)
- `run()` — sync entry point for `project.scripts` console entry
- Server should be ~150-200 lines, not ~1460

- [ ] **Step 3: Update __init__.py**

Update version to `"4.0.0"` and exports.

- [ ] **Step 4: Update pyproject.toml entry point**

Change `project.scripts` from `"azure_pricing_mcp.server:main"` to `"azure_pricing_mcp.server:run"`.

- [ ] **Step 5: Verify __main__.py still works**

Current `__main__.py` does `from .server import main` and `asyncio.run(main())`. Since the new `main()` is still async, this should still work. Verify and update if needed.

- [ ] **Step 6: Verify server starts**

Run: `python -m azure_pricing_mcp --help` or `python -c "from azure_pricing_mcp import main; print('OK')"`
Expected: No import errors

- [ ] **Step 7: Commit**

```bash
git add src/azure_pricing_mcp/server.py src/azure_pricing_mcp/handlers.py src/azure_pricing_mcp/__init__.py src/azure_pricing_mcp/__main__.py pyproject.toml
git commit -m "refactor: rewrite server.py and handlers.py as thin orchestration layer"
```

---

## Chunk 5: Tests

### Task 17: Update existing tests for new architecture

**Files:**
- Modify: `tests/test_azure_pricing.py`
- Modify: `tests/test_mcp_server.py`
- Modify: `tests/test_ri_pricing.py`
- Modify: `tests/test_http_transport.py`

- [ ] **Step 1: Update test imports**

All tests must update imports to use the new module paths:
- `from azure_pricing_mcp.client import AzurePricingClient`
- `from azure_pricing_mcp.services.pricing import PricingService, normalize_sku_name`
- `from azure_pricing_mcp.services.sku import SKUService`
- `from azure_pricing_mcp.handlers import ToolHandlers`
- `from azure_pricing_mcp.config import SERVICE_NAME_MAPPINGS`

Update fixtures to create service instances via the new class constructors.

- [ ] **Step 2: Run existing tests**

Run: `pytest tests/test_azure_pricing.py tests/test_mcp_server.py tests/test_ri_pricing.py -v`
Expected: All tests pass (fix any import/API changes)

- [ ] **Step 3: Commit**

```bash
git add tests/
git commit -m "test: update existing tests for modular architecture"
```

---

### Task 18: Add new test files

**Files:**
- Create: `tests/test_databricks.py`
- Create: `tests/test_github_pricing.py`
- Create: `tests/test_orphaned_resources.py`
- Create: `tests/test_ptu_sizing.py`
- Create: `tests/test_spot.py`

- [ ] **Step 1: Create test_databricks.py**

Port upstream tests:
- `TestResolveWorkloadType` — alias/case/whitespace handling
- `TestDatabricksService` — pricing, estimation, comparison with mocked API responses
- `TestDatabricksFormatters` — verify Markdown output format
- `TestDatabricksHandlers` — handler integration tests

- [ ] **Step 2: Create test_github_pricing.py**

Port upstream tests:
- `TestGitHubPricingConfig` — static data validation
- `TestResolveProduct` — alias resolution
- `TestGetPricing` — catalog lookup with filters
- `TestEstimateCost` — cost calculations (plan, copilot, actions, codespaces, LFS, GHAS)
- `TestFormatters` — Markdown output verification
- `TestHandlerIntegration` — handler produces TextContent

- [ ] **Step 3: Create test_orphaned_resources.py**

Port upstream tests:
- `TestOrphanedResourcesService` — mocked Azure Resource Graph responses
- Tests for each of the 11 resource type detection queries
- Cost data aggregation tests

- [ ] **Step 4: Create test_ptu_sizing.py**

Port upstream tests:
- `TestPTUModelTable` — model spec validation
- `TestComputeEqTPM`, `TestComputeRawPTU`, `TestRoundToValidPTU` — calculation logic
- `TestEstimatePTUSizing` — end-to-end sizing estimation
- `TestPTUSizingFormatter` — Markdown output
- `TestHandler` — handler integration

- [ ] **Step 5: Create test_spot.py**

Port upstream tests:
- `TestSpotService` — mocked Azure Resource Graph responses for eviction rates
- `TestSpotPriceHistory` — mocked price history responses
- `TestSimulateEviction` — mocked eviction simulation
- Auth error handling when credentials are missing

- [ ] **Step 6: Run all tests**

Run: `pytest tests/ -v --tb=short`
Expected: All tests pass

- [ ] **Step 7: Commit**

```bash
git add tests/
git commit -m "test: add tests for Databricks, GitHub, orphaned resources, PTU, Spot"
```

---

## Chunk 6: Quality and Final Verification

### Task 19: Run full quality checks and fix issues

**Files:**
- Potentially any file that needs formatting/type fixes

- [ ] **Step 1: Run black**

Run: `black src/ tests/`
Fix: Any formatting issues

- [ ] **Step 2: Run ruff**

Run: `ruff check --fix src/ tests/`
Fix: Any linting issues

- [ ] **Step 3: Run mypy**

Run: `mypy src/`
Fix: Any type errors

- [ ] **Step 4: Run full test suite**

Run: `pytest tests/ -v -m "not integration"`
Expected: All tests pass

- [ ] **Step 5: Commit any fixes**

```bash
git add -A
git commit -m "fix: code quality fixes (black, ruff, mypy)"
```

---

### Task 20: Update CLAUDE.md with new architecture

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Update CLAUDE.md**

Update to reflect new modular architecture:
- New file structure overview (client.py, config.py, services/, databricks/, github_pricing/)
- Updated data flow diagram
- New tools list (18 total)
- Azure auth requirement for Spot/Orphaned tools
- New environment variables

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: update CLAUDE.md for v4.0.0 architecture"
```

---

### Task 21: Delete old monolithic code (if any remaining)

**Files:**
- Verify: `src/azure_pricing_mcp/server.py` is now thin (~150-200 lines)
- Verify: No dead code remains from the original monolithic structure

- [ ] **Step 1: Verify no dead code**

Check that the original ~1460-line server.py has been replaced by the thin version.
Confirm no orphaned imports or unused functions.

- [ ] **Step 2: Final full test run**

Run: `pytest tests/ -v -m "not integration" && black --check src/ tests/ && ruff check src/ tests/ && mypy src/`
Expected: All green

- [ ] **Step 3: Commit if needed**

```bash
git add -A
git commit -m "chore: clean up dead code from v2 monolith"
```
