<p align="center">
  <a href="https://github.com/blackwell-systems"><img src="https://raw.githubusercontent.com/blackwell-systems/blackwell-docs-theme/main/badge-trademark.svg" alt="Blackwell Systems"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="License"></a>
</p>

---

# code-context-benchmark

Reproducible evaluation for code context retrieval systems. 308 tasks, 16 repos, 8 languages.

---

## Why this exists

No code context retrieval system publishes reproducible precision metrics. codegraph (19K stars), GitNexus (40K stars), Gortex, codebase-memory, and Aider all ship without published P@10, ground truth corpora, or reproducibility tooling.

This benchmark is the first multi-system evaluation in the space. It is designed to be run by anyone, on any system, with pinned inputs and deterministic scoring.

## What it measures

**P@10 (Precision at 10):** Of the top 10 symbols returned for a task, what fraction are actually relevant? This is the headline metric because it directly measures whether the context an agent receives is useful.

Also measured: R@10, NDCG@10, MRR, token efficiency, latency.

## Corpus

| Stat | Value |
|------|-------|
| Tasks | 308 |
| Repos | 16 |
| Languages | Go, Python, TypeScript, Java, C#, Rust, Ruby, multi |
| Ground truth | Hand-curated, expert-verified, DB-validated |
| Matching | Dot-bounded containment (no mid-word substring inflation) |

Each repo is pinned to an exact commit. See `corpus/MANIFEST.yaml`.

### Repos

| Repo | Language | Tasks |
|------|----------|------:|
| Django | Python | 33 |
| Terraform | Go | 20 |
| Caddy | Go | 20 |
| Jekyll | Ruby | 20 |
| Rails | Ruby | 20 |
| Ocelot | C# | 20 |
| FastAPI | Python | 20 |
| Spark-Java | Java | 20 |
| Ripgrep | Rust | 20 |
| Kafka | Java | 19 |
| Kubernetes | Go | 19 |
| Flask | Python | 19 |
| Cargo | Rust | 19 |
| VS Code | TypeScript | 19 |
| Saleor | Python | 11 |
| Cross-cutting | Mixed | 9 |

## Quick start

```bash
# 1. Clone this repo
git clone https://github.com/blackwell-systems/code-context-benchmark.git
cd code-context-benchmark

# 2. Clone corpus repos at pinned commits
./corpus/corpus-setup.sh clone

# 3. Index (requires a code context system that produces graph DBs)
# Example with knowing:
KNOWING_BIN=knowing ./corpus/corpus-setup.sh index

# 4. Clear any stale state
for db in corpus/repos/*/.knowing/graph.db; do
  sqlite3 "$db" "DELETE FROM task_memory; DELETE FROM feedback;" 2>/dev/null
done

# 5. Run the benchmark
go test ./... -run '^TestCrossSystem$' -v -timeout 0
```

## Writing an adapter

Implement the `Adapter` interface:

```go
type Adapter interface {
    Name() string
    Index(repoPath string) (indexTimeMs int64, err error)
    Retrieve(repoPath string, task Task, tokenBudget int) (RetrievalResult, error)
    SupportsLearning() bool
    RecordFeedback(repoPath string, task Task, relevantSymbols []string) error
    Reset(repoPath string) error
}
```

Put your adapter in `adapters/` and register it in the harness. The benchmark runs every registered adapter on every task and reports comparative metrics.

## Methodology

- **Same input, different systems.** Every system receives identical task descriptions.
- **Cold start.** No pre-existing feedback, session history, or learned state.
- **Manual ground truth.** Hand-labeled by a developer who read the source code.
- **Paired statistical tests.** Wilcoxon signed-rank (non-parametric).
- **Effect size over p-values.** Cohen's d reported alongside significance.

Full methodology: [METHODOLOGY.md](METHODOLOGY.md)

## Task format

```yaml
id: django-easy-003
repo: django
tier: easy
description: "Write a management command that exports user data to CSV"
ground_truth:
  - BaseCommand
  - BaseCommand.handle
  - BaseCommand.add_arguments
  - OutputWrapper
  - call_command
tags: [management-commands, cli]
```

Tasks are in `corpus/tasks/<repo>/<tier>/`.

## Contributing

We welcome:
- New task fixtures (with ground truth validated against the graph DB)
- New corpus repos (with pinned commits and MANIFEST entry)
- New adapter implementations
- Methodology improvements

## License

MIT
