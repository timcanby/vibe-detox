# Contributing to Vibe Detox

Thanks for helping rebuild the fundamentals. This guide will get you started.

## Ways to contribute

- **Add a missing fundamental concept** — something developers use without understanding.
- **Improve an existing explanation** — clarity, accuracy, or practical depth.
- **Add diagrams or examples** — concrete beats abstract.
- **Correct technical inaccuracies** — precision is the point.
- **Add references** — papers, books, talks, RFCs, specs.
- **Suggest topics developers are forgetting** — open an issue labeled `forgotten-fundamental`.

---

## Before you write

1. **Search existing issues and docs** to avoid duplicates.
2. **Open a topic proposal** (use [`templates/topic-proposal.md`](templates/topic-proposal.md)) if you're unsure whether your topic fits.
3. Wait for brief discussion — keep it short, we'd rather you write than wait.

---

## How to add a concept

1. Pick the right domain folder under `docs/` (e.g. `docs/networking/`).
2. Copy [`templates/concept-template.md`](templates/concept-template.md) to a new file.
3. Name the file in `kebab-case.md` (e.g. `tcp-congestion-control.md`).
4. Fill in every section. Skip a section only if it genuinely doesn't apply.
5. Commit with a message like: `docs(networking): add TCP congestion control`.
6. Open a pull request.

---

## Writing guidelines

- **Write to teach, not to impress.** Clear > clever.
- **Explain the why before the how.**
- **Examples should run or be copy-pasteable when possible.**
- **Name the abstraction that AI tools hide.** That's the detox.
- **Cite primary sources** (RFCs, papers, specs) over blog posts.
- **No "I just asked ChatGPT and..."** — this repo is the anti-decay project.

---

## Domain folders

| Folder | Covers |
|---|---|
| `computer-science/` | Algorithms, data structures, complexity, theory |
| `mathematics/` | Linear algebra, probability, discrete math, calculus |
| `programming/` | Language semantics, paradigms, memory models |
| `backend/` | Servers, APIs, concurrency, queues, caching |
| `databases/` | SQL/NoSQL, indexing, transactions, isolation |
| `networking/` | TCP/IP, HTTP, DNS, load balancing |
| `operating-systems/` | Processes, memory, scheduling, filesystems |
| `software-engineering/` | Testing, design patterns, architecture principles |
| `distributed-systems/` | Consensus, replication, partitioning, CAP |
| `system-design/` | Scalability, trade-offs, real-world architectures |
| `machine-learning/` | Training, evaluation, optimization, classic ML |
| `ai-engineering/` | LLMs, RAG, embeddings, inference, fine-tuning |

---

## Review criteria

Pull requests are reviewed for:

- **Correctness** — no hand-waving past the hard parts.
- **Clarity** — a motivated reader should walk away understanding it.
- **Consistency** — follows the template structure.
- **Honesty about abstractions** — names what AI tools hide.

---

## Code of conduct

By participating you agree to uphold the [Code of Conduct](CODE_OF_CONDUCT.md). Be kind, be precise, be patient with people relearning.
