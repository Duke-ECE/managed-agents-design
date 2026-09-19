# Managed Agents Design

Cross-repository technical documentation for the Duke-ECE managed agents platform.

## Organization

One requirement has one document directory, even when implementation spans several
code repositories. Keep its design, decision log, and delivery record together.

```text
managed-agents-design/
├── template/
│   ├── README.md
│   ├── design.md
│   ├── decisions.md
│   └── delivery.md
└── requirements/
    ├── README.md
    └── 0001-durable-context-compaction/
        ├── README.md
        ├── design.md
        ├── decisions.md
        └── delivery.md
```

Copy the root-level template/ for each new requirement. Append decisions to that
requirement's decisions.md. Other requirements may link to those decisions
without copying them. Current implemented architecture lives in architecture/.

## Documentation boundaries

- [standards](https://github.com/Duke-ECE/standards): shared engineering rules.
- [managed-agents-docs](https://github.com/Duke-ECE/managed-agents-docs): product documentation.
- This repository: requirements, designs, decisions, and delivery evidence.
- Service repositories: authoritative code, migrations, contracts, and runbooks.

## Navigation

- [Requirements and creation instructions](requirements/README.md)
- [Complete requirement template](template/README.md)
- [Current architecture](architecture/README.md)
- [Agent instructions](AGENTS.md)

All documents are English. This Markdown-only repository has no build or deployment.
