# Planning Docs

Initial research and planning for positioning Nebius Token Factory as a
first-class inference provider in LangSmith Fleet, mirroring the existing
`langchain-fireworks` integration pattern.

## Files

- **[INTEGRATION_PLAN.md](INTEGRATION_PLAN.md)** — Audit of `langchain-nebius`
  v0.1.3 against `langchain-fireworks` 1.3.0. Identifies 10 concrete gaps
  (G1–G10), four open questions (U1–U4), and three workstreams (package
  changes, Fleet model config, end-to-end demo). Reads as a planning artifact:
  *what* to do and *why*.

- **[IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md)** — Companion to the
  integration plan. Six sequenced upstream PRs with concrete diffs, branch
  names, test commands, and commit messages. Plus the demo notebook outline
  and a PR-tracker checklist. Reads as an execution artifact: *how* to do it.

## How these were produced

Drafted iteratively across an exploratory session that audited both
`nebius/langchain-nebius` and `langchain-ai/langchain` (the Fireworks
integration), compared inheritance patterns, identified the AI Studio →
Token Factory rebrand, and resolved the major framing decisions before
codifying the implementation.

## Status

These are planning artifacts only — no code changes have been made to the
package. Sequencing recommendation: start with PR-2 (standard conformance
tests) since it both resolves the largest open question (U4) and surfaces
any latent `BaseChatOpenAI` inheritance bugs early.
