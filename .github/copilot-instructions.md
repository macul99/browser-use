# Browser-Use Development Guide

Browser-Use is an async Python ≥3.11 library for AI-driven browser automation via LLMs + CDP (Chrome DevTools Protocol). AI agents autonomously navigate, interact with elements, and complete tasks by processing HTML and making LLM-driven decisions.

## Architecture Overview

Event-driven architecture with these core components:
- **Agent** (`browser_use/agent/service.py`): Orchestrates tasks, manages browser sessions, executes LLM-driven action loops
- **BrowserSession** (`browser_use/browser/session.py`): Manages browser lifecycle, CDP connections, coordinates watchdog services via `bubus` event bus
- **Tools** (`browser_use/tools/service.py`): Action registry mapping LLM decisions to browser operations (click, type, scroll, extract, etc.)
- **DomService** (`browser_use/dom/service.py`): Extracts/processes DOM content, handles element highlighting, accessibility tree generation
- **LLM Integration** (`browser_use/llm/`): Abstraction layer for OpenAI, Anthropic, Google, Groq, etc.

### Event-Driven Pattern
BrowserSession uses `bubus.EventBus` to coordinate watchdogs:
- **DownloadsWatchdog**: PDF auto-download, file management
- **PopupsWatchdog**: JavaScript dialogs/popups
- **SecurityWatchdog**: Domain restrictions, security policies
- **DOMWatchdog**: DOM snapshots, screenshots, element highlighting
- **AboutBlankWatchdog**: Empty page redirects

Events are defined in `browser_use/browser/events.py` and follow the pattern: Agent/Tools emit events → BrowserSession watchdogs handle them.

### CDP Integration
Uses [`cdp-use`](https://github.com/browser-use/cdp-use) for typed CDP protocol access:
```python
cdp_client.send.DOMSnapshot.enable(session_id=session_id)
cdp_client.send.Target.attachToTarget(params=ActivateTargetParameters(targetId=target_id, flatten=True))
cdp_client.register.Browser.downloadWillBegin(callback_func)  # NOT cdp_client.on(...)
```

## Development Workflow

**Setup:**
```bash
uv venv --python 3.11
source .venv/bin/activate
uv sync
```

**Testing:**
- CI tests: `./bin/test.sh` or `uv run pytest --numprocesses auto tests/ci`
- Single test: `uv run pytest -vxs tests/ci/test_specific_test.py`
- Tests in `tests/ci/` run automatically in CI

**Quality Checks:**
```bash
./bin/lint.sh              # Full check: formatter + linter + type checker (~5s)
./bin/lint.sh --quick      # Skip pyright (~2s)
./bin/lint.sh --staged     # Only staged files
uv run pyright             # Type checking only
uv run ruff check --fix    # Linting
uv run ruff format         # Formatting
```

**MCP Server Mode:**
```bash
uvx browser-use[cli] --mcp
```

## Code Style Requirements

- **Async Python**: Use async/await throughout
- **Indentation**: Tabs only (not spaces) in all Python code
- **Modern typing**: `str | None` not `Optional[str]`, `list[str]` not `List[str]`, `dict[str, Any]` not `Dict[str, Any]`
- **Logging methods**: Prefix with `_log_*` (e.g., `_log_pretty_path()`) to separate from main logic
- **Pydantic v2**: Use for internal data and user-facing APIs
  - Configure with `model_config = ConfigDict(extra='forbid', validate_by_name=True, ...)`
  - Use `Annotated[..., AfterValidator(...)]` for validation logic
- **File organization**: 
  - Main logic in `service.py`
  - Pydantic models in `views.py`
  - Event definitions in `events.py`
- **Runtime assertions**: Use at function start/end to enforce constraints
- **IDs**: `from uuid_extensions import uuid7str` + `id: str = Field(default_factory=uuid7str)`

## Testing Requirements

**Critical**: Never mock anything except LLM responses! Use real objects.
- **LLM mocking**: Use pytest fixtures in `conftest.py` to set up LLM responses
- **Browser scenarios**: Use `pytest-httpserver` to set up test HTML (see `tests/ci/` examples)
- **No real URLs**: Never use `https://google.com` or `https://example.com` - use pytest-httpserver instead
- **Modern pytest-asyncio**: No `@pytest.mark.asyncio` decorator needed, just use async functions
- **No event_loop argument**: Use `asyncio.get_event_loop()` inside tests if needed
- **Move passing tests to `tests/ci/`**: Files there run automatically in CI

## Key Patterns

**Service Pattern**: Each component has a `service.py` with main logic
**Views Pattern**: Pydantic models in `views.py`
**Events Pattern**: Event-driven communication via `bubus.EventBus`
**Browser Configuration**: `browser_use/browser/profile.py` manages launch args, extensions, display detection
**System Prompts**: Agent prompts in `browser_use/agent/system_prompt*.md`

## Important Constraints

- **Always use `uv`** instead of `pip` for dependency management
- **Never create random example files** when implementing features - test inline if needed
- **Use real model names** - don't replace user-specified models (users try new models we don't know yet)
- **Return `ActionResult`** from tools with structured content to help agents reason
- **Run pre-commit** before PRs: `./bin/lint.sh`
- **Default model**: Recommend `ChatBrowserUse` - best for browser automation (accuracy + speed + cost)

## Production & Cloud

**Sandbox Decorator**: Deploy local code to production with `@sandbox()`:
```python
from browser_use import Browser, sandbox, ChatBrowserUse
from browser_use.agent.service import Agent

@sandbox(cloud_profile_id='your-profile-id', cloud_proxy_country_code='us')
async def production_task(browser: Browser):
    agent = Agent(task="Your task", browser=browser, llm=ChatBrowserUse())
    await agent.run()
```

**Cloud Browser**: Use `Browser(use_cloud=True)` for remote browsers with captcha bypass, auth handling, lowest latency. Requires `BROWSER_USE_API_KEY` environment variable.

## Development Strategy

When making significant changes:
1. Find/write tests verifying current behavior
2. Write failing tests for new design
3. Implement changes, run tests during development
4. Run full `tests/ci` suite - confirm new design works + backward compatibility
5. Condense test logic into one file, deduplicate
6. Update `docs/` and `examples/`

For massive refactors: Use event buses and job queues to break systems into isolated services managing subcomponents.
