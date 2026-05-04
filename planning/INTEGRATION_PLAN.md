# Nebius Token Factory × LangSmith Fleet — Integration Plan

Audit and three-workstream plan to make Nebius Token Factory a first-class
inference provider in LangSmith Fleet, mirroring the `langchain-fireworks` /
Fleet integration. **Plan only — no implementation in this pass.**

---

## Naming context (load-bearing)

**AI Studio has been rebranded to Token Factory.** They are the same product —
Token Factory is the current name for what shipped as Nebius AI Studio. Evidence:
`docs.nebius.com/studio/inference/quickstart` 307-redirects to
`docs.tokenfactory.nebius.com/`. The current docs site references neither the
old name nor the rebrand explicitly (no migration notice).

What this means for the audit:
- The `langchain-nebius` v0.1.3 package is internally branded "Nebius AI Studio"
  throughout (README, class docstrings, examples). That's now stale naming, not
  a separate product.
- The default base_url `api.studio.nebius.ai/v1/` is the **legacy domain** for
  the same service now hosted at `api.tokenfactory.nebius.com/v1/`. Whether
  the old domain still works (alias / 30x) or is being phased out is open
  (U1 below); either way, the package should target the new canonical URL.

## Blocking unknowns (resolve before workstream commits)

| # | Unknown | Why it blocks | Owner to ask |
|---|---------|---------------|--------------|
| U1 | Whether `api.studio.nebius.ai/v1/` is permanently aliased to `api.tokenfactory.nebius.com/v1/`, or scheduled for deprecation. | Determines whether G1 is "switch the default for hygiene" (low-risk) or "switch the default before existing users break" (urgent). Also drives whether to keep a legacy fallback. | Akim Tsvigun (`aktsvigun@nebius.com`, langchain-nebius maintainer) |
| U2 | Where LangSmith Fleet sources per-token cost data. Not in `langchain-fireworks/pyproject.toml`, not in `langchain_fireworks/data/_profiles.py` (those are capability profiles, not pricing). Likely a closed-source LangSmith pricing table or a models.dev join. | Workstream 2 cost telemetry can't be planned without knowing whether Nebius needs to (a) submit a PR somewhere, (b) hand a CSV to LangSmith, or (c) do nothing because Fleet pulls from models.dev automatically. | LangSmith eng partner contact |
| U3 | Whether LangSmith's "Provider" dropdown is a fixed enum or accepts a custom string. If enum, adding "Nebius" / "Token Factory" requires a LangSmith product change. If freeform, ship today as `OpenAI-compatible`. | Workstream 2 deliverable wording. | LangSmith product |
| U4 | Whether Nebius's `bind_tools`, `with_structured_output`, and streaming actually work end-to-end via the inherited `BaseChatOpenAI` path against the Token Factory endpoint. The package's existing integration tests likely run against the legacy `studio` domain. | Workstream 1 effort estimate flips from S→M if any of these break under the new domain. | Live test (Workstream 1 task 1.1) |

---

## Audit: `langchain-nebius` v0.1.3 vs `langchain-fireworks` 1.3.0

Side-by-side review of `/tmp/langchain-nebius-ref/libs/nebius/langchain_nebius/`
vs `/tmp/langchain-fireworks-ref/libs/partners/fireworks/langchain_fireworks/`.

**Inheritance fork.** ChatNebius extends `BaseChatOpenAI` (wraps the OpenAI
Python client); ChatFireworks subclasses `BaseChatModel` directly and uses the
`fireworks-ai` SDK. The OpenAI-compatible path means ChatNebius gets a lot for
free (tool calling, structured output, streaming, JSON mode) — but it also
means anywhere Fireworks customizes the OpenAI path (provider tagging, usage
metadata in streaming, model profiles), ChatNebius silently inherits OpenAI's
behavior.

### Gap table

| # | Gap | Severity | File path in `nebius/langchain-nebius` | Fix sketch | Verify |
|---|-----|----------|----------------------------------------|------------|--------|
| G1 | **Default `base_url` is the legacy AI Studio domain.** Default is `https://api.studio.nebius.ai/v1/`; current canonical Token Factory URL is `https://api.tokenfactory.nebius.com/v1/`. Same product, renamed; old domain status pending U1. | High — default points at deprecated naming; risk of future breakage if legacy alias is sunset | `libs/nebius/langchain_nebius/chat_models.py:323` | Flip default to the Token Factory URL. Keep `NEBIUS_API_BASE` env override + `base_url` constructor arg as the escape hatch for anyone still pinned to the legacy domain. | `ChatNebius()` (no args) hits the Token Factory `/chat/completions` endpoint successfully with valid creds |
| G2 | **No model capability profiles.** Fireworks ships `langchain_fireworks/data/_profiles.py` (~400 lines, auto-generated from models.dev) wired through `langchain_core.language_models.ModelProfileRegistry`. ChatFireworks calls `_resolve_model_profile()`. ChatNebius has no `data/` directory. | High — Fleet UI capability surfacing (max_tokens, tool_calling, structured_output) won't populate for Nebius models | New file: `libs/nebius/langchain_nebius/data/_profiles.py`; new method on ChatNebius: `_resolve_model_profile()` | Generate via the `langchain-profiles` CLI referenced in fireworks `_profiles.py` header (see `https://docs.langchain.com/oss/python/langchain/models#updating-or-overwriting-profile-data`). Cover the 60+ Token Factory models. | `ChatNebius(model="deepseek-ai/DeepSeek-R1-0528").profile` returns a populated `ModelProfile` |
| G3 | **No `langchain_tests.ChatModelUnitTests` conformance test.** Fireworks ships `tests/unit_tests/test_standard.py` subclassing `ChatModelUnitTests`. Nebius has only ad-hoc unit tests; the `langchain-tests` dep is declared in `pyproject.toml` but unused. | High — without standard conformance, Fleet integration risks silent breakage on LangChain core upgrades | New file: `libs/nebius/tests/unit_tests/test_standard.py` mirroring `tests/unit_tests/test_standard.py` in fireworks | Subclass `ChatModelUnitTests`, return `ChatNebius` from `chat_model_class`, set `init_from_env_params` for `NEBIUS_API_KEY` / `NEBIUS_API_BASE`. Add an analogous `test_integration.py`. | `pytest tests/unit_tests/test_standard.py` passes; the test exercises ~30 standard checks |
| G4 | **`response_metadata["model_provider"]` not set.** Fireworks sets `message.response_metadata["model_provider"] = "fireworks"` in `_create_chat_result` (chat_models.py:852). ChatNebius inherits BaseChatOpenAI which sets `model_provider` from the OpenAI client (likely `"openai"` or absent). Fleet's per-provider analytics depend on this. | Medium — `_get_ls_params` is correct (`ls_provider="nebius"`), but the message-level metadata may mis-attribute. Verify before fixing. | `libs/nebius/langchain_nebius/chat_models.py` — override `_create_chat_result` or inject in a post-processor | Verify what BaseChatOpenAI currently writes; if wrong, override. | Live call: assert `result.response_metadata["model_provider"] == "nebius"` |
| G5 | **Streaming `usage_metadata` not guaranteed.** Fireworks 1.2.0 explicitly opts into `stream_options.include_usage` and surfaces the final empty-`choices` chunk as `AIMessageChunk` with `usage_metadata`. ChatNebius inherits BaseChatOpenAI's behavior which is similar but version-coupled to `langchain-openai`. | Medium — Fleet cost dashboards rely on per-call usage metadata even for streaming runs | Verify with a live streaming test against Token Factory; if usage metadata is missing on the final chunk, override `_stream`/`_astream` to inject `stream_options={"include_usage": True}` | Override or pin `langchain-openai` floor | Live streaming test asserts the last chunk has `usage_metadata` populated |
| G6 | **No `service_tier` / `fast` mode parameter.** Fireworks 1.3.0 added `service_tier: str | None`. Token Factory exposes "fast" model variants (e.g. `Llama-3.3-70B-Instruct-fast`) — currently surfaced only via the `model` string. | Low — works today via model string, but no first-class param | `chat_models.py` — optional `service_tier` field | Pass-through to `extra_body`; document the fast-model naming convention | Init test |
| G7 | **Docstring bugs.** `chat_models.py:48` says "from env var OPENAI_API_KEY" (should be `NEBIUS_API_KEY`). `chat_models.py:53` typos "langhcain_nebius". | Trivial | `libs/nebius/langchain_nebius/chat_models.py:48,53` | One-line fix each | Visual review |
| G8 | **Stale "AI Studio" branding throughout.** Package description (`pyproject.toml:8`: "LangChain integration for Nebius AI Studio"), README, and class docstrings reference "Nebius AI Studio" — the old name. Token Factory is the current name for the same product. | Medium — wrong-named package will confuse anyone landing from current Nebius docs / marketing | `libs/nebius/pyproject.toml:8`, `libs/nebius/README.md`, all class docstrings in `chat_models.py`, `embeddings.py`, `retrievers.py`, `tools.py` | Bulk rename "AI Studio" → "Token Factory" in user-facing strings. Bump minor version. Optionally add a one-line note: "formerly Nebius AI Studio." | `grep -ri "AI Studio" libs/nebius/` returns only intentional references (e.g., a single migration note) |
| G9 | **No `is_lc_serializable() -> True`.** ChatFireworks declares this (chat_models.py:561). ChatNebius does not. Affects checkpointing / runnable serialization. | Low | `chat_models.py` | Add classmethod | Unit test imports + serializes |
| G10 | **Python floor too low.** Pyproject declares `python = ">=3.8.1"` and classifies 3.8/3.9. Fireworks requires `>=3.10.0`; LangChain core 1.x dropped 3.8/3.9. | Low (but blocks adopting recent langchain-core releases) | `libs/nebius/pyproject.toml:27` | Bump to `>=3.10` | `uv sync` resolves cleanly with current langchain-core |

### Recommendation: PR upstream, do not fork

Defaulting to (a) per the prompt: submit PRs to `nebius/langchain-nebius`. The
inheritance choice (BaseChatOpenAI) is sound — Token Factory is genuinely
OpenAI-compatible, and Akim is the maintainer of record at Nebius. Forking to
subclass BaseChatModel would (i) duplicate ~1300 lines of fireworks-style
plumbing, (ii) lose Akim's maintenance investment, and (iii) split the package
ecosystem. The gaps above are all bolt-on changes, not architectural rewrites.

**Exception:** if G1's resolution requires a behavior-breaking default change
(flipping base_url to Token Factory), discuss with Akim before submitting —
that's a v0.2 / 1.0 question, not a patch release.

---

## Workstream 1: ChatNebius package gaps — **effort M**

**Scope.** Land G1–G10 as upstream PRs to `nebius/langchain-nebius`. Order by
severity. Cluster G1+G8 as one "Token Factory positioning" PR (default URL +
docs). G2 as its own PR (model profiles is a 400-line generated file). G3 as
its own PR (standard tests). G4–G6 + G9 as one "feature parity" PR. G7+G10 as
trivial cleanup.

**Files to touch (in nebius/langchain-nebius repo, not this directory):**
- `libs/nebius/langchain_nebius/chat_models.py` — G1, G4, G5, G6, G7, G9
- `libs/nebius/langchain_nebius/data/__init__.py` (new), `data/_profiles.py` (new) — G2
- `libs/nebius/tests/unit_tests/test_standard.py` (new) — G3
- `libs/nebius/tests/integration_tests/test_standard.py` (new) — G3
- `libs/nebius/README.md` — G8
- `libs/nebius/pyproject.toml` — G10

**Dependencies.** Resolve U1 (legacy `studio` domain status) before sizing G1
— if it's permanently aliased, G1 is a low-stakes default flip; if it's
sunsetting, G1 is urgent and we should also publish a deprecation note.
Resolve U4 (do inherited methods work against Token Factory?) before deciding
whether G4–G6 are "verify and close as no-op" or "actual code changes."

**Verification checklist.**
- [ ] `pytest tests/unit_tests` passes including new `test_standard.py`
- [ ] `pytest tests/integration_tests` passes against live Token Factory creds
- [ ] `ChatNebius(model="...")` with no `base_url` calls the intended endpoint (Studio or Token Factory per U3)
- [ ] `ChatNebius(...).bind_tools([SomeSchema]).invoke("...")` returns valid tool_calls
- [ ] `ChatNebius(...).with_structured_output(SomeModel).invoke("...")` returns a Pydantic instance
- [ ] Streaming a 1k-token response yields a final chunk with populated `usage_metadata`
- [ ] LangSmith trace for an invoke shows `ls_provider="nebius"` AND `response_metadata.model_provider="nebius"`
- [ ] `ChatNebius(model="deepseek-ai/DeepSeek-R1-0528").profile` returns a non-empty `ModelProfile`

---

## Workstream 2: LangSmith Fleet model config — **effort S–M (depends on U2)**

**Scope.** Add Nebius Token Factory as a usable provider in the Fleet Model
Configurations library. Mostly LangSmith-side configuration, no production
code changes in `langchain-nebius`.

### 2.1 Model Configurations to seed

Five entries in the LangSmith Model Configurations library (literal values, not
placeholders):

| Display name | Provider field | base_url | Model string | Default secret |
|---|---|---|---|---|
| Llama 3.3 70B (Token Factory, Fast) | `OpenAI-compatible` (until U3) → `Nebius` | `https://api.tokenfactory.nebius.com/v1/` | `meta-llama/Llama-3.3-70B-Instruct-fast` | `NEBIUS_API_KEY` |
| DeepSeek R1 0528 (Token Factory) | same | same | `deepseek-ai/DeepSeek-R1-0528` | `NEBIUS_API_KEY` |
| DeepSeek R1 Distill Llama 70B | same | same | `deepseek-ai/DeepSeek-R1-Distill-Llama-70B` | `NEBIUS_API_KEY` |
| Qwen 2.5 72B Instruct | same | same | `Qwen/Qwen2.5-72B-Instruct` | `NEBIUS_API_KEY` |
| Mistral Nemo Instruct 2407 | same | same | `mistralai/Mistral-Nemo-Instruct-2407` | `NEBIUS_API_KEY` |

> **Verify exact model IDs against the Token Factory docs before seeding.** The
> list above is a starter set — Token Factory advertises 60+ models. Authoritative
> list source: `https://docs.tokenfactory.nebius.com/`. Production seed should
> include Kimi K2.x once Nebius confirms availability (referenced in the
> benchmark tweet but not visible on the marketing page model list).

### 2.2 First-class provider in UI

Pending U3. Two paths:

- **If LangSmith Provider field accepts arbitrary strings:** ship today, set
  `Provider="Nebius"` on each config row, file a follow-up with LangSmith to
  add a logo + dropdown entry.
- **If LangSmith Provider is a fixed enum:** ship as `Provider="OpenAI-compatible"`
  with `base_url` override. File a LangSmith product request to add Nebius to
  the enum (same pattern Fireworks went through — find the PR/Linear issue and
  reference it).

### 2.3 Feature Access table

Default-on for: **Fleet** (chat agents), **Playground**. Default-off pending
ownership decision: **Evaluators** (does it make sense to evaluate agent runs
with the same provider you're benchmarking? Probably not), **Polly**
(unfamiliar — confirm with LangSmith product), **Insights** (read-only by
nature of the model config; no toggle needed).

### 2.4 Cost / pricing telemetry — pending U2

Without resolving U2, this section is conditional:

- **If pricing comes from models.dev:** Token Factory must be added to
  `https://github.com/sst/models.dev`. PR Nebius pricing entries with the same
  schema fireworks uses. Then re-run `langchain-profiles` regen for nebius
  (Workstream 1 G2 already covers this).
- **If pricing is in a closed-source LangSmith table:** file an internal LangSmith
  ticket with the per-1M-token rates (input + output) for the five seed models.
  Authoritative source: Nebius self-serve dashboard or sales-confirmed sheet.
- **If neither:** Fleet's cost-per-run column will show `N/A` for Nebius runs.
  Acceptable for v0; flag as a follow-up.

### 2.5 Workspace secret naming

Use `NEBIUS_API_KEY` (matches the package's env var default). Do not
introduce `NEBIUS_TOKEN_FACTORY_KEY` — same product, same credential.

**Verification checklist.**
- [ ] All five model configs visible in LangSmith → Settings → Model Configurations
- [ ] Each row in Feature Access shows Fleet enabled, Evaluators disabled
- [ ] Smoke test: a Fleet agent invoking the Llama 3.3 70B Fast config returns a response and the trace shows `ls_provider="nebius"`
- [ ] Cost column populated for at least one model (or U2 documented as known gap)

---

## Workstream 3: End-to-end demo (Tavily + Nebius + Fleet) — **effort S**

**Scope.** A runnable Jupyter notebook demonstrating Token Factory in a Fleet
agent with Tavily search, plus a side-by-side cost benchmark against the
analogous Fireworks model.

### 3.1 Demo task

**"News brief agent."** Given a topic (e.g., `"vertical integration in cloud
inference providers"`), the agent (a) searches with Tavily for the top recent
articles, (b) fetches and summarizes each, (c) returns a 200-word brief with
inline citations. Exercises tool calling (Tavily), multi-turn reasoning, and
output formatting.

### 3.2 Stack wiring

```
LangSmith Fleet agent
├── Model: ChatNebius(model="meta-llama/Llama-3.3-70B-Instruct-fast",
│                     base_url="https://api.tokenfactory.nebius.com/v1/")
├── Tools: TavilySearchResults(max_results=5)
└── Prompt: ReAct-style with citation requirement
```

**Model choice.** `meta-llama/Llama-3.3-70B-Instruct-fast` because: (a) it's
the package default, (b) "fast" tier optimizes for tool-calling latency, (c)
Llama 3.3 has the strongest open-weight tool-calling track record. Fireworks
analogue for the benchmark: `accounts/fireworks/models/llama-v3p3-70b-instruct`.

### 3.3 Cost benchmark methodology

| Param | Value |
|---|---|
| Prompt set | 20 topics drawn from a fixed seed list (commit the file with the notebook) |
| Runs per prompt | 3 (jitter for cache effects); take median |
| N total runs per provider | 60 |
| Providers compared | (a) ChatFireworks Llama 3.3 70B, (b) ChatNebius Llama 3.3 70B Fast |
| Metrics | per-run cost ($), p50/p95 wall latency (s), tool-calling success rate (%), output token count |
| Cost source | **Provider invoice authoritative**, but report LangSmith trace metadata cost alongside and flag any divergence |
| Latency measurement | Wall time in the agent runner, not just inference (so tool calls + agent overhead count — that's what users actually see) |

**Why both cost sources.** LangSmith traces reflect what Fleet *thinks* the
run cost; the provider invoice is what the user actually pays. If they
diverge, that's a Workstream 2 / U1 bug to fix before declaring victory.

### 3.4 Where the demo lives

- Notebook: `/Users/colin/Code/langchain-nebius/demo/fleet_nebius_tavily.ipynb`
- Seed prompts: `/Users/colin/Code/langchain-nebius/demo/topics.txt`
- Benchmark output (committed for reproducibility): `/Users/colin/Code/langchain-nebius/demo/results.csv`
- Short README section in `/Users/colin/Code/langchain-nebius/README.md` with the headline cost-per-run number and a link to the notebook

Do **not** push to `nebius/langchain-nebius` upstream until the demo has been
reviewed; that's a follow-up after Workstream 1 lands.

**Verification checklist.**
- [ ] Notebook runs end-to-end on a fresh kernel with only `NEBIUS_API_KEY`, `FIREWORKS_API_KEY`, `TAVILY_API_KEY`, `LANGSMITH_API_KEY` env vars set
- [ ] `results.csv` contains 120 rows (60 per provider) with non-null cost + latency
- [ ] Per-run cost for Nebius is lower than Fireworks (otherwise the whole exercise is invalidated — re-check model selection)
- [ ] Tool-calling success rate ≥ 90% for both providers
- [ ] LangSmith trace links for at least 5 sampled runs included in the notebook

---

## Critical files referenced

- Existing package source: `https://github.com/nebius/langchain-nebius` (cloned to `/tmp/langchain-nebius-ref`)
- Reference integration: `https://github.com/langchain-ai/langchain/tree/master/libs/partners/fireworks` (cloned to `/tmp/langchain-fireworks-ref`)
- Token Factory docs: `https://docs.tokenfactory.nebius.com/`
- Token Factory marketing / model list: `https://nebius.com/services/token-factory`
- Source brief: `/Users/colin/Downloads/fireworks_to_nebius_token_factory.md`
- Maintainer contact: Akim Tsvigun, `aktsvigun@nebius.com`

## Effort summary

| Workstream | Effort | Blocked by |
|---|---|---|
| W1: Package gaps | M (1–2 weeks PR cycle with upstream review) | U1, U4 |
| W2: Fleet model config | S–M | U2, U3 |
| W3: E2E demo | S (2–3 days incl. benchmark runs) | None — can start in parallel using current package + custom base_url |

W3 can start immediately and surface real bugs that retroactively scope W1 +
W2. Recommend kicking it off first.
