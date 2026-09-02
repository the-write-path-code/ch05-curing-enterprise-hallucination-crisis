# Chapter 5: Curing the Enterprise Hallucination Crisis

Companion code for *Building Safe Agentic AI for Enterprise Systems* by Mohit Aggarwal.

This repository implements corrective retrieval for cases where a standard retrieval-augmented generation (RAG) pipeline has found evidence that is relevant but incomplete, stale, or otherwise unfit to answer the full question. It combines a three-state Corrective RAG (CRAG) router, a self-reflective retrieval loop, query decoupling, and a bounded best-effort response path.

The repository does not treat a grounded disclaimer as an adequate endpoint when more targeted retrieval could resolve the missing part of a request. It also does not let the system retry forever. Retrieval, critique, rewriting, and fallback occur within stated limits.

## What You Will Run

| Chapter section | Demonstration | What it shows |
| --- | --- | --- |
| 5.1 | Corrective routing | Why one similarity search should not pass directly into generation without a check on relevance and completeness. |
| 5.2 | Three-state CRAG gatekeeper | `CORRECT`, `AMBIGUOUS`, and `INCORRECT` retrieval outcomes and the retrieval path each one selects. |
| 5.3 | SR-RAG critique loop | A secondary evaluator checks a draft for grounding and utility, then routes it to delivery, refinement, or a new retrieval cycle. |
| 5.4 | Query decoupling | The mutable search query changes across retries while the original user intent remains fixed for final generation. |
| 5.5 | Best-effort delivery and tracing | A bounded loop returns supported partial information when full resolution is unavailable, rather than discarding verified results or fabricating the missing part. |

## Production Warning

The fallback path can leave the enterprise data boundary. In this implementation, a `CORRECT` result stays with the local Qdrant context, an `AMBIGUOUS` result combines local context with web retrieval, and an `INCORRECT` result replaces local context with the web fallback. Do not enable external retrieval in a system containing sensitive, regulated, or proprietary requests until you have an explicit egress policy.

The critique loop is bounded. Do not remove the loop and iteration limits to chase a complete answer. A system that repeatedly re-retrieves and rewrites without a stop condition turns uncertainty into latency and cost without creating new evidence.

## Prerequisites

- Git
- [uv](https://docs.astral.sh/uv/)
- Python 3.9, which is the version pinned by `.python-version`
- A Google Gemini API key for generation and evaluation calls
- A Serper API key for the web-fallback path
- Network access for the web-fallback path

The repository uses Qdrant and Docling as part of the local retrieval and document-ingestion path. Check the repository configuration before choosing local or remote Qdrant deployment.

## Quick Start

### 1. Install uv

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### 2. Clone and synchronize the repository

```bash
git clone https://github.com/the-write-path-code/ch05-curing-enterprise-hallucination-crisis.git
cd ch05-curing-enterprise-hallucination-crisis
uv sync
```

The repository includes `uv.lock` and `.python-version`. Use `uv sync` after pulling changes so the environment matches the dependency set used by the companion code.

### 3. Create local configuration

Copy `.example.env` to create a local `.env` file from the variables read by the application configuration, and do not commit it:

```bash
cp .example.env .env
```

At minimum, configure the Gemini and Serper credentials required by your implementation. The variable names below match the names the repository reads:

```dotenv
GEMINI_API_KEY=your-gemini-api-key
SERPER_API_KEY=your-serper-api-key
```

A `.example.env` file is provided with placeholder values and comments describing each setting.

## Configuration

The pipeline has two bounded loops:

- `max_full_loops` limits full retrieval cycles. The Chapter 5 implementation defaults to two.
- `max_iterations` limits draft-refinement cycles. The Chapter 5 implementation defaults to three.

Keep both limits explicit and configurable. They define the maximum amount of retrieval and model work one request can trigger.

The pipeline uses three context sources:

| Source | Role | When used |
| --- | --- | --- |
| Qdrant | Local retrieval context | The initial search and the `CORRECT` route |
| Serper | External web fallback | The `INCORRECT` route and the supplementary path for `AMBIGUOUS` evidence |
| Generated draft and critique | Transient working state | Evaluated before delivery; not a substitute for retrieval evidence |

> **Tip**
>
> Test the `CORRECT` path first. It is the lowest-cost route and lets you validate the local corpus, Qdrant connection, and basic generation path before introducing external search or recursive critique.

## Run the Chapter Demonstrations

### 1. Run a Corrective Retrieval Query, Sections 5.1 and 5.2

Run the main entry point with a question:

```bash
uv run python src/main.py "Your question here"
```

The pipeline begins with local Qdrant retrieval, then evaluates the returned context against the search query.

- `CORRECT`: local context proceeds to generation.
- `AMBIGUOUS`: local context is retained and supplemented with web retrieval.
- `INCORRECT`: local context is removed from the active context and the system falls back to web retrieval.

The three states are not cosmetic labels. They control what evidence the generator is allowed to see.

### 2. Inspect the Self-Reflective Loop, Section 5.3

The generator produces a draft against the selected active context. A secondary evaluator checks two distinct conditions:

- **Grounding:** whether the draft's claims are supported by retrieved evidence.
- **Utility:** whether the draft satisfies the original user request.

A grounded disclaimer may fail utility. In that case, the system does not deliver it as success. It rewrites the search query and begins another bounded retrieval cycle.

The chapter's routing thresholds are:

| Critique outcome | Pipeline action |
| --- | --- |
| Grounding score at least 0.8 and utility is positive | Deliver the answer |
| Grounding score from 0.4 to below 0.8 and utility is positive | Refine the draft, subject to the iteration limit |
| Grounding score below 0.4, or utility is negative | Reset the retrieval path and rewrite the search query |

These values are chapter-specific configuration, not universal production thresholds.

### 3. Inspect Query Decoupling, Section 5.4

The pipeline maintains two values:

```text
search_query     Mutable retrieval string
original_query   Immutable user intent
```

The query rewriter may change `search_query` to locate a missing fact. Final generation still receives `original_query`, so a targeted rewrite does not replace the full multi-part request.

Review the orchestrator before changing this behavior. Passing the rewritten search string directly to final generation reintroduces the query-drift failure the chapter describes.

### 4. Inspect Best-Effort Delivery, Section 5.5

When the full-loop limit is reached, the pipeline can return a partial result if its grounding score meets the stated best-effort threshold. It does not fill in the missing material. If grounding remains low, it returns a failure response instead.

This is not a license to return partial answers silently. A production caller should be able to distinguish a complete answer from one delivered under the best-effort path.

## Expected Results

The most useful output is the route taken by a query, not the wording of one response. A run should show:

1. The local retrieval result.
2. The CRAG decision: `CORRECT`, `AMBIGUOUS`, or `INCORRECT`.
3. The context sources selected for generation.
4. The draft's grounding and utility evaluation.
5. Any refinement or rewritten query.
6. The final delivery decision, including whether it is a best-effort result.

The chapter uses a query comparing legacy Orion specifications with a newer Starship V3 reference to show the failure mode. Local retrieval can find Orion context but miss the newer material. A grounded statement that the context lacks the answer is safe in one narrow sense and still unhelpful. The corrective path seeks the missing evidence without allowing the mutable retrieval query to replace the original request.

## Run the Tests

Run the repository test suite:

```bash
uv run pytest
```

Run tests before changing route conditions, loop caps, query rewriting, fallback behavior, or the best-effort threshold. The tests should establish that the three CRAG paths, query preservation, and bounded-loop behavior remain intact.

## Repository Layout

```text
.
├── README.md
├── pyproject.toml
├── uv.lock
├── .python-version
├── .example.env                       # Environment configuration template
├── ISSUES.md                          # Design notes and resolved failure modes
├── TESTING_QUERIES.md                 # Multi-part and edge-case test prompts
├── src/
│   └── main.py                        # Pipeline entry point
├── data/                              # Local corpus for document ingestion
├── workflows/                         # CRAG and SR-RAG workflow materials and diagrams
├── tests/
└── .env                               # Local credentials only; never commit
```

## Architecture Diagrams and Supporting Documents

The repository documents the CRAG and SR-RAG paths with Mermaid workflows. Use those diagrams with the Chapter 5 figures to trace the following sequence:

```text
Original user query
    -> local Qdrant retrieval
    -> CRAG relevance evaluation
    -> local, hybrid, or web-only context
    -> draft generation
    -> grounding and utility critique
    -> delivery, refinement, or query rewrite
```

The design documents should remain the authoritative source for exact prompt templates, model configuration, retrieval collection names, and observability settings.

## Safety and Operational Limits

- The CRAG evaluator and critique call use model output. Temperature settings and structured instructions can constrain them; they do not turn them into mathematical proofs.
- Web fallback introduces external egress. Do not send protected health information, confidential policy text, credentials, or sensitive identifiers through it.
- The `AMBIGUOUS` route combines local and web context. Preserve source provenance so a reader can distinguish proprietary evidence from externally retrieved material.
- Best-effort delivery returns only material that clears the configured grounding threshold. It must not invent the unresolved part.
- Query decoupling preserves original intent, but it does not guarantee that retrieval finds all required evidence.
- Context caching can reduce repeated input cost in recursive loops. It does not remove the need for loop caps, tracing, or budget limits.

## Troubleshooting

### The application cannot find Gemini credentials

Confirm that the local `.env` file exists and that its variable names match the application configuration. Do not place keys in source files or commit the `.env` file.

### The Serper fallback fails

Confirm the Serper key, network access, and any configured endpoint. A local-only deployment should treat fallback failure as a defined result, not as permission to generate from insufficient context.

### The pipeline keeps refining a response

Check `max_full_loops`, `max_iterations`, the utility decision, and the query-rewrite output. Do not remove the bounds. If the system repeatedly seeks unavailable evidence, the correct result may be a transparent partial response or a defined failure state.

### The final answer ignores part of a multi-part question

Inspect the handoff to final generation. It must receive the immutable original query, not only the most recent rewritten search query.

### A grounded answer is still unhelpful

That is the truthful-ignorance failure mode. Review the utility evaluation and the missing-evidence route before relaxing grounding requirements.

## Related Chapters

- Chapter 3 establishes the dense, sparse, hybrid, and Qdrant retrieval patterns that supply the initial evidence.
- Chapter 4 measures retrieval, grounding, sufficiency, and final policy decisions.
- Chapter 6 extends controlled retrieval and handoff patterns to multimodal inputs.
- Chapter 15 turns grounding and regression checks into CI merge gates and uses recorded traces for model migration testing.

## License and Errata

See `LICENSE` for licensing terms. Report documentation or code issues through this repository's GitHub issue tracker.
