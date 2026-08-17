# Vibe Detox

> **Vibe responsibly. Know what your code is doing.**

Vibe Detox is a community-driven project for rebuilding and preserving the fundamentals behind modern software development.

AI can generate code faster than ever, but understanding still matters.

This repository collects clear, practical explanations of the concepts that developers increasingly rely on without thinking about — from algorithms, networking, databases, and operating systems to machine learning and AI engineering.

The goal is not to reject AI-assisted development.

**The goal is to make sure abstraction never becomes ignorance.**

---

## Ways to contribute

- **Add a missing fundamental concept** — pick something developers use without understanding, explain it clearly.
- **Improve an existing explanation** — make it clearer, more accurate, or more practical.
- **Add diagrams or examples** — a good example is worth a thousand words.
- **Correct technical inaccuracies** — precision matters.
- **Add references** — link to papers, books, talks, RFCs.
- **Suggest topics developers are forgetting** — open an issue with the `forgotten-fundamental` label.

See [`CONTRIBUTING.md`](CONTRIBUTING.md) to get started.

---

## Repository structure

```
vibe-detox/
├── README.md
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
├── ROADMAP.md
│
├── docs/                  # Knowledge base, organized by domain
│   ├── computer-science/
│   ├── mathematics/
│   ├── programming/
│   ├── backend/
│   ├── databases/
│   ├── networking/
│   ├── operating-systems/
│   ├── software-engineering/
│   ├── distributed-systems/
│   ├── system-design/
│   ├── machine-learning/
│   └── ai-engineering/
│
├── templates/             # Standard templates for contributions
│   ├── concept-template.md
│   └── topic-proposal.md
│
└── .github/
    ├── ISSUE_TEMPLATE/
    │   ├── new-topic.md
    │   ├── correction.md
    │   └── discussion.md
    └── PULL_REQUEST_TEMPLATE.md
```

Every concept doc follows the same [standard template](templates/concept-template.md) — so the knowledge base stays consistent no matter who writes it.

---

## The standard

Not a tutorial dump. A knowledge base with a single, consistent format.

Every entry answers:

1. **What is it?**
2. **Why does it matter?**
3. **Mental model**
4. **How does it actually work?**
5. **Example**
6. **Common misconceptions**
7. **What abstractions / AI tools often hide**
8. **Practical engineering implications**
9. **Related topics**
10. **References**

---

## License

MIT — see [`LICENSE`](LICENSE).

---

**Detox the vibe. Rebuild the fundamentals.**
