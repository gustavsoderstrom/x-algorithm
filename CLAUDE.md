# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is the X (Twitter) "For You" feed recommendation algorithm. It combines in-network posts (from followed accounts) with out-of-network posts (ML-discovered) and ranks them using a Grok-based transformer model.

## Project Structure

The codebase has four main components:

- **home-mixer/** (Rust) - Orchestration layer that assembles the For You feed. Implements the `PhoenixCandidatePipeline` which coordinates all stages of recommendation.

- **candidate-pipeline/** (Rust) - Reusable framework for building recommendation pipelines. Defines traits: `Source`, `Hydrator`, `Filter`, `Scorer`, `Selector`, `SideEffect`.

- **thunder/** (Rust) - In-memory post store with real-time Kafka ingestion. Serves in-network candidates (posts from followed accounts) with sub-millisecond latency.

- **phoenix/** (Python/JAX) - ML models for retrieval and ranking:
  - Two-tower model for retrieval (user/candidate embeddings + ANN search)
  - Grok-based transformer for ranking with candidate isolation attention masking

## Commands

### Phoenix (Python/JAX ML code)

Requires [uv](https://docs.astral.sh/uv/getting-started/installation/).

```bash
# Run from phoenix/ directory
uv run run_ranker.py              # Run ranking model
uv run run_retrieval.py           # Run retrieval model
uv run pytest test_recsys_model.py test_recsys_retrieval_model.py  # Run tests
```

### Rust Components

The Rust code (home-mixer, candidate-pipeline, thunder) uses standard Cargo commands. Note: Some modules are marked as excluded from open source release (clients, params, util in home-mixer).

## Architecture

### Pipeline Flow

1. **Query Hydration** - Fetch user context (engagement history via `UserActionSeqQueryHydrator`, following list via `UserFeaturesQueryHydrator`)
2. **Candidate Sourcing** - Parallel fetch from `ThunderSource` (in-network) and `PhoenixSource` (out-of-network retrieval)
3. **Hydration** - Enrich candidates with core data, author info, video duration, subscription status
4. **Pre-Scoring Filters** - Remove duplicates, old posts, self-posts, blocked/muted authors, muted keywords
5. **Scoring** - Sequential application: `PhoenixScorer` → `WeightedScorer` → `AuthorDiversityScorer` → `OONScorer`
6. **Selection** - Sort by score and select top K
7. **Post-Selection** - Visibility filtering, conversation deduplication

### Key Patterns

- **Candidate Isolation**: In Phoenix ranking, candidates cannot attend to each other during transformer inference - only to user context. This ensures scores are consistent and cacheable regardless of batch composition.

- **Multi-Action Prediction**: The model predicts probabilities for many engagement types (like, reply, repost, click, etc.) with positive/negative weights combined into final score.

- **Parallel Execution**: Sources and hydrators run in parallel where possible; filters and scorers run sequentially.

- **Trait-Based Pipeline**: The `CandidatePipeline` trait defines the contract. Each component (Source, Filter, Scorer, etc.) implements its respective trait with `enable()`, `name()`, and core logic methods.
