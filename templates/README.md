# Templates

Templates are grouped by the document structure they create:

```text
templates/
├── README.md
├── iteration/
│   ├── README.md
│   ├── design.md
│   └── delivery.md
└── decision.md
```

## Start an iteration

Copy the complete [iteration/](iteration/README.md) directory to
`iterations/NNNN-descriptive-slug/`, using the next available iteration ID.
The directory contains the overview, technical design, and delivery plan with
relative links that work unchanged after copying.

Replace placeholders, remove sections that do not apply, and add the iteration
to [the iteration index](../iterations/README.md). Create an `assets/` directory
only when diagrams or other supporting files are needed.

## Record a decision

Copy [decision.md](decision.md) to `decisions/NNNN-descriptive-slug.md`, using
the next available decision ID. Decision numbering is independent of iteration
numbering; one iteration may produce several decisions.

Add the record to [the decision index](../decisions/README.md) and link it from
the relevant iteration. New decisions are appended. To replace an accepted
decision, create a new record and mark the previous one Superseded with a link
to its replacement. Proposed decisions and editorial corrections may be edited
in place.
