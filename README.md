# LLM Wiki

A practical, source-backed wiki for understanding large language models and
the adjacent technical systems they depend on.

The project is inspired by clear, first-principles LLM notes such as Andrej
Karpathy's educational material. It is intended to grow as a set of focused
pages rather than as one long tutorial.

## Principles

- Explain the smallest useful idea first, then add implementation detail.
- Link claims to primary sources, papers, documentation, or runnable examples.
- Keep theory, engineering practice, and empirical results clearly separated.
- Prefer small diagrams, equations, and examples over vague summaries.
- Record dates and version-sensitive assumptions when they matter.

## Structure

Content lives in [`docs/`](docs/). Reusable knowledge is curated here; active
repository contracts and operational procedures stay with their owning code.

The main sections are:

- `foundations/` - mathematics, representations, optimisation, and training
- `systems/` - inference, serving, data, evaluation, and hardware
- `techniques/` - prompting, fine-tuning, retrieval, and agents
- `papers/` - short paper notes and links to original sources
- `glossary/` - concise definitions and cross-links

Adjacent systems knowledge may live under `systems/` when it is useful for
understanding model-driven technical work. Each migrated page should retain a
source note and should not pretend to be the live contract for another repo.

Start with [`docs/index.md`](docs/index.md). New pages should use Markdown,
have one clear topic, and link to related pages and sources.

## Status

This repository is at the initial scaffold stage. The wiki structure is
deliberately tooling-agnostic so a publishing workflow can be added once the
content model is established.
