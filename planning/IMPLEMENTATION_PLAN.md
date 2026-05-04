# Implementation Plan — Nebius Token Factory × LangSmith Fleet

Companion to `INTEGRATION_PLAN.md`. Where INTEGRATION_PLAN says *what* to do,
this doc says *how* — concrete diffs, PR sequencing, branch names, and the
demo notebook structure.

**Strategy:** fork-and-PR-upstream against `nebius/langchain-nebius`. Six PRs
sequenced by risk, plus parallel Fleet config work and the demo notebook.

---

## Phase 0: Setup (~30 min)

### 0.1 Fork and clone

```bash
# Fork github.com/nebius/langchain-nebius via GitHub UI to your account first.

cd /Users/colin/Code/langchain-nebius
git init
git remote add upstream https://github.com/nebius/langchain-nebius.git
git remote add origin git@github.com:<your-fork>/langchain-nebius.git
git fetch upstream
git checkout -b main upstream/main
git push -u origin main
```

### 0.2 Local dev env

```bash
cd libs/nebius
uv venv
source .venv/bin/activate
uv pip install -e ".[dev]"
uv pip install langchain-tests pytest pytest-asyncio pytest-socket syrupy
```

### 0.3 Live test creds

```bash
# Add to ~/.zshrc or .envrc — needed for integration tests + the demo
export NEBIUS_API_KEY="<from console.nebius.com IAM service account>"
export TAVILY_API_KEY="<from tavily.com>"
export FIREWORKS_API_KEY="<for the cost-comparison run only>"
export LANGSMITH_API_KEY="<from smith.langchain.com>"
export LANGSMITH_TRACING=true
```

### 0.4 Pre-flight smoke

```bash
# Confirm the legacy and new domains both work today (resolves U1)
curl -sS https://api.studio.nebius.ai/v1/models -H "Authorization: Bearer $NEBIUS_API_KEY" | head -c 200
curl -sS https://api.tokenfactory.nebius.com/v1/models -H "Authorization: Bearer $NEBIUS_API_KEY" | head -c 200

# If both return 200 with a models list, U1 is "alias is live, default flip is safe."
# If only the new one works, file an issue with Nebius — existing users may break.
```

### 0.5 Reach out to Akim

Email `aktsvigun@nebius.com` before opening the first PR. One paragraph:
"We're positioning Token Factory as a LangSmith Fleet provider. Want to land
a series of small PRs against langchain-nebius. Mind sharing your release
cadence and any upcoming v0.2 plans so we don't collide?"

This is the single highest-leverage 5 minutes in the whole project.

---

## Phase 1: Six upstream PRs

Sequenced low-risk → high-risk. Each PR is independently mergeable and
revertible.

### PR-1: Cleanup (G7, G10) — ~30 min

**Branch:** `cleanup/docstrings-and-python-floor`

**Files & diffs:**

`libs/nebius/langchain_nebius/chat_models.py`
```diff
-        api_key: Optional[str]
-            Nebius API key. If not passed in will be read from env var OPENAI_API_KEY.
+        api_key: Optional[str]
+            Nebius API key. If not passed in will be read from env var NEBIUS_API_KEY.
@@
-            from langhcain_nebius import ChatNebius
+            from langchain_nebius import ChatNebius
```

`libs/nebius/pyproject.toml`
```diff
-python = ">=3.8.1,<4.0"
-langchain-core = ">=0.3.35"
+python = ">=3.10.0,<4.0"
+langchain-core = ">=0.3.35,<2.0.0"
@@
 classifiers = [
     "Development Status :: 4 - Beta",
     ...
-    "Programming Language :: Python :: 3.8",
-    "Programming Language :: Python :: 3.9",
     "Programming Language :: Python :: 3.10",
```

**Verify:** `pytest tests/unit_tests` still passes; `uv sync` resolves.

**Commit msg:** `cleanup: fix docstring typos and bump python floor to 3.10`

---

### PR-2: Standard conformance tests (G3) — ~2 hours

**Branch:** `tests/standard-conformance`

**New file** `libs/nebius/tests/unit_tests/test_standard.py`:
```python
"""Standard LangChain interface tests."""
from langchain_core.language_models import BaseChatModel
from langchain_tests.unit_tests import ChatModelUnitTests

from langchain_nebius import ChatNebius


class TestNebiusStandard(ChatModelUnitTests):
    @property
    def chat_model_class(self) -> type[BaseChatModel]:
        return ChatNebius

    @property
    def chat_model_params(self) -> dict:
        return {
            "model": "meta-llama/Llama-3.3-70B-Instruct-fast",
            "api_key": "test_api_key",
        }

    @property
    def init_from_env_params(self) -> tuple[dict, dict, dict]:
        return (
            {
                "NEBIUS_API_KEY": "api_key",
                "NEBIUS_API_BASE": "https://base.com",
            },
            {"model": "meta-llama/Llama-3.3-70B-Instruct-fast"},
            {
                "nebius_api_key": "api_key",
                "nebius_api_base": "https://base.com",
            },
        )
```

**New file** `libs/nebius/tests/integration_tests/test_standard.py`:
```python
"""Standard LangChain interface integration tests (live API)."""
import pytest
from langchain_core.language_models import BaseChatModel
from langchain_tests.integration_tests import ChatModelIntegrationTests

from langchain_nebius import ChatNebius


class TestNebiusStandardIntegration(ChatModelIntegrationTests):
    @property
    def chat_model_class(self) -> type[BaseChatModel]:
        return ChatNebius

    @property
    def chat_model_params(self) -> dict:
        return {"model": "meta-llama/Llama-3.3-70B-Instruct-fast"}

    @property
    def supports_image_inputs(self) -> bool:
        return False  # Token Factory text-only for the default model

    @property
    def has_tool_calling(self) -> bool:
        return True
```

Add to `pyproject.toml` under `[tool.poetry.dev-dependencies]` (already present
per the existing file — verify the extras come through with the dev install).

**Verify:**
```bash
pytest tests/unit_tests/test_standard.py -v   # ~30 standard checks
NEBIUS_API_KEY=$NEBIUS_API_KEY pytest tests/integration_tests/test_standard.py -v
```

Document any failures in the PR description — they're the actual gap surface
that `BaseChatOpenAI` inheritance hides today. **This PR is also how we
resolve U4.**

**Commit msg:** `tests: add ChatModelUnitTests + ChatModelIntegrationTests conformance suites`

---

### PR-3: Token Factory rebrand (G8) — ~1 hour

**Branch:** `rebrand/token-factory`

**Bulk renames** in user-facing strings. Use `sd` or careful sed:

```bash
# In libs/nebius/, replace "Nebius AI Studio" with "Nebius Token Factory" in:
sd "Nebius AI Studio" "Nebius Token Factory" \
   langchain_nebius/chat_models.py \
   langchain_nebius/embeddings.py \
   langchain_nebius/retrievers.py \
   langchain_nebius/tools.py \
   README.md \
   pyproject.toml
```

Specific manual edits:

`libs/nebius/pyproject.toml`
```diff
-description = "LangChain integration for Nebius AI Studio"
+description = "LangChain integration for Nebius Token Factory (formerly Nebius AI Studio)"
```

`libs/nebius/README.md` — replace lines 1–3 with:
```markdown
# LangChain Nebius Integration

This package provides LangChain integration for **Nebius Token Factory**
(formerly Nebius AI Studio), enabling chat, embeddings, retrieval, and
tool-calling against Nebius's open-model inference platform.
```

And line 217:
```diff
-For more details, refer to the [Nebius AI Studio API Documentation](https://studio.nebius.ai/docs/api-reference).
+For more details, refer to the [Nebius Token Factory documentation](https://docs.tokenfactory.nebius.com/).
```

**Verify:**
```bash
grep -ri "AI Studio" libs/nebius/  # should return only the "(formerly Nebius AI Studio)" notes
pytest tests/unit_tests
```

**Commit msg:** `rebrand: align package naming with Nebius Token Factory`

---

### PR-4: Default base_url to Token Factory (G1) — ~30 min, **blocked by U1**

**Branch:** `feat/default-token-factory-url`

**Pre-req:** Phase 0.4 smoke test confirms `api.studio.nebius.ai` aliases to
the new domain. If not, hold this PR and file an issue with Nebius first.

`libs/nebius/langchain_nebius/chat_models.py`
```diff
     nebius_api_base: str = Field(
         default_factory=from_env(
-            "NEBIUS_API_BASE", default="https://api.studio.nebius.ai/v1/"
+            "NEBIUS_API_BASE", default="https://api.tokenfactory.nebius.com/v1/"
         ),
         alias="base_url",
     )
```

Same change in `libs/nebius/langchain_nebius/embeddings.py:125` and
`libs/nebius/langchain_nebius/retrievers.py:114`.

**Add a CHANGELOG entry** (create `libs/nebius/CHANGELOG.md` if absent):
```markdown
## 0.2.0

### Changed
- **Default `base_url` updated to `https://api.tokenfactory.nebius.com/v1/`**
  (Token Factory canonical domain). The legacy `https://api.studio.nebius.ai/v1/`
  remains accessible via `base_url=` or the `NEBIUS_API_BASE` environment variable.
- Package metadata rebranded from "Nebius AI Studio" to "Nebius Token Factory."
```

Bump version in `pyproject.toml` to `0.2.0`.

**Verify:**
```bash
NEBIUS_API_KEY=$NEBIUS_API_KEY python -c "
from langchain_nebius import ChatNebius
chat = ChatNebius(model='meta-llama/Llama-3.3-70B-Instruct-fast')
print(chat.nebius_api_base)
print(chat.invoke('Say hi in three words.').content)
"
```

**Commit msg:** `feat!: default base_url to Token Factory canonical domain`

---

### PR-5: Feature parity (G4, G5, G6, G9) — ~3 hours

**Branch:** `feat/parity-with-fireworks`

`libs/nebius/langchain_nebius/chat_models.py` — add after `_get_ls_params`:

```python
@classmethod
def is_lc_serializable(cls) -> bool:
    """Return whether this model can be serialized by LangChain."""
    return True

service_tier: Optional[str] = None
"""Service tier for the request. Pass model variants like
'meta-llama/Llama-3.3-70B-Instruct-fast' for fast routing; this field is
reserved for future Token Factory tier support."""
```

For G4 (response_metadata model_provider) and G5 (streaming usage), the
inherited `BaseChatOpenAI` behavior may already be correct — **verify with
PR-2's integration tests first.** If they fail:

```python
def _create_chat_result(self, response, generation_info=None):
    result = super()._create_chat_result(response, generation_info)
    for gen in result.generations:
        if hasattr(gen.message, "response_metadata"):
            gen.message.response_metadata["model_provider"] = "nebius"
    return result
```

**Verify:**
```bash
pytest tests/unit_tests
NEBIUS_API_KEY=$NEBIUS_API_KEY pytest tests/integration_tests
```

**Commit msg:** `feat: add serializable flag, service_tier param, fix response metadata provider tag`

---

### PR-6: Model capability profiles (G2) — ~4 hours

**Branch:** `feat/model-profiles`

**New file** `libs/nebius/langchain_nebius/data/__init__.py` (empty).

**New file** `libs/nebius/langchain_nebius/data/_profiles.py`:

Generate via `langchain-profiles` CLI (see `https://docs.langchain.com/oss/python/langchain/models#updating-or-overwriting-profile-data`).
If the CLI doesn't have Nebius source data on models.dev yet, **submit a PR
to https://github.com/sst/models.dev first** with Token Factory's model
catalog — this is a prerequisite, not part of PR-6.

Manual fallback if models.dev integration is blocked: hand-write profiles for
the top 5 models (Llama 3.3 70B Fast, DeepSeek R1 0528, DeepSeek R1 Distill,
Qwen 2.5 72B, Mistral Nemo 2407) using the `_PROFILES` dict shape from
fireworks's `_profiles.py:18` as a template.

`libs/nebius/langchain_nebius/chat_models.py` — add at module level:
```python
from langchain_core.language_models import ModelProfile, ModelProfileRegistry
from langchain_nebius.data._profiles import _PROFILES

_MODEL_PROFILES = cast("ModelProfileRegistry", _PROFILES)


def _get_default_model_profile(model_name: str) -> ModelProfile:
    return (_MODEL_PROFILES.get(model_name) or {}).copy()
```

And on the class:
```python
def _resolve_model_profile(self) -> ModelProfile | None:
    return _get_default_model_profile(self.model_name) or None
```

**Verify:**
```python
# Should match fireworks's tests/unit_tests/test_standard.py:test_profile pattern
from langchain_nebius import ChatNebius
m = ChatNebius(model="deepseek-ai/DeepSeek-R1-0528", api_key="test")
assert m.profile  # populated
```

**Commit msg:** `feat: add model capability profiles via ModelProfileRegistry`

---

## Phase 2: Fleet model config (parallel with Phase 1)

Does not require any of the PRs above to land — Fleet config uses the existing
`langchain-nebius` 0.1.3 with a custom `base_url` override.

### 2.1 Email LangSmith team (resolves U2 + U3)

Single email to the LangSmith eng partner contact (or post in the relevant
partnership thread):

> We're seeding Nebius Token Factory model configurations in LangSmith for a
> Fleet integration demo. Two questions:
>
> 1. The Provider field in Model Configurations — does it accept arbitrary
>    strings, or is it a fixed enum? If enum, what's the process to add "Nebius"
>    (or "Token Factory") so it gets a logo + dropdown entry like Fireworks?
> 2. The cost column on Fleet runs — where does that data live? If we need to
>    publish per-token rates somewhere (models.dev, an internal table, a PR
>    against a langchain-ai repo), point us at the right place.
>
> We'll ship initially as `OpenAI-compatible` with a custom `base_url` if
> that's the path of least resistance.

### 2.2 Seed Model Configurations

In LangSmith UI → Settings → Model Configurations → + Create, create five
entries with the table from `INTEGRATION_PLAN.md` § 2.1. Workspace secret:
`NEBIUS_API_KEY` (one secret, used by all five).

Once seeded, screenshot the Model Configurations panel for the partnership
update — this is the visual artifact for the status update.

### 2.3 Feature Access

Toggle each Nebius config to **enabled** for Fleet. Leave disabled for
Evaluators / Polly until product confirmation.

---

## Phase 3: Demo notebook (parallel with Phase 1)

Does not depend on any PR landing.

### 3.1 Notebook outline

`/Users/colin/Code/langchain-nebius/demo/fleet_nebius_tavily.ipynb`

| Cell | Purpose |
|---|---|
| 1 (md) | Title, summary, "what this demonstrates" |
| 2 (md) | Setup: pip install, env vars |
| 3 (code) | Imports + env var checks |
| 4 (code) | Build the Tavily tool (`TavilySearchResults(max_results=5)`) |
| 5 (code) | Build the Nebius LLM with explicit Token Factory `base_url` (works against current 0.1.3 — no PR-4 dependency) |
| 6 (code) | Build the Fireworks LLM (Llama 3.3 70B analogue) for comparison |
| 7 (code) | Compose the agent (LangGraph `create_react_agent`) — one factory function used for both LLMs |
| 8 (md) | "News brief" prompt template + 20-topic seed list (loaded from `topics.txt`) |
| 9 (code) | Benchmark loop: for each (provider, topic, run_idx), invoke the agent, capture trace_url, cost, latency, tool_call_count |
| 10 (code) | Aggregate to `results.csv` |
| 11 (code) | Render comparison table + plot (matplotlib, p50/p95 latency, cost-per-run) |
| 12 (md) | Summary: headline cost-per-run delta, links to 5 sample LangSmith traces |

### 3.2 Files committed alongside

- `demo/topics.txt` — 20 topic strings, one per line, fixed seed
- `demo/results.csv` — committed for reproducibility (regenerate by re-running cell 10)
- `demo/results.png` — exported plot
- A `demo/README.md` with the headline number (e.g., "Token Factory was
  Xx cheaper at parity tool-call success")

### 3.3 Cost capture

Two sources, captured per run:
- **Trace cost:** `langsmith.Client().read_run(run_id).total_cost` (relies on
  Workstream 2 cost telemetry — may be `None` until U2 is resolved; that's OK,
  capture it as a column)
- **Provider invoice cost:** computed locally from `usage_metadata` × per-token
  rates (hardcode the rates in a constant at the top of cell 5; document
  source URL in a comment)

If the two columns diverge significantly in the final results table, that's a
finding to flag in the demo write-up.

### 3.4 Tavily key handling

Tavily free tier is 1k searches/month. The benchmark runs ~120 searches × 5
results = ~600 search calls — well under the free tier. No need to provision
a paid key for this demo.

---

## Phase 4: PR-tracker checklist

Mark off as PRs land upstream.

- [ ] PR-1 cleanup merged (`cleanup/docstrings-and-python-floor`)
- [ ] PR-2 standard tests merged (`tests/standard-conformance`)
- [ ] PR-3 rebrand merged (`rebrand/token-factory`)
- [ ] PR-4 default URL merged (`feat/default-token-factory-url`) — gated on U1
- [ ] PR-5 parity merged (`feat/parity-with-fireworks`)
- [ ] PR-6 profiles merged (`feat/model-profiles`) — gated on models.dev PR
- [ ] LangSmith email sent; U2 + U3 answered
- [ ] 5 Model Configurations seeded in LangSmith
- [ ] Demo notebook runs end-to-end
- [ ] Cost benchmark shows Nebius < Fireworks at parity tool-call success
- [ ] Status update posted with screenshots + headline number

---

## Estimated total effort

| Phase | Effort |
|---|---|
| Phase 0 setup | ~1 hour |
| Phase 1 (PRs 1–6) | ~10 hours hands-on + 1–3 weeks PR review cycle |
| Phase 2 Fleet config | ~30 min hands-on + 2–5 days waiting on LangSmith reply |
| Phase 3 demo notebook | ~1 day (incl. benchmark runs) |
| **Total wall time to "done":** | **~3 weeks** assuming Akim merges at a normal cadence |
| **Total hands-on effort:** | **~2 days** of focused work |

The critical path is PR-2 (standard tests) — it both resolves U4 and surfaces
any latent BaseChatOpenAI inheritance bugs that would otherwise bite during
the Fleet demo. Start there.
