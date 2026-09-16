# tools/

Repository tooling. Nothing here ships to consumers.

## Rules

1. Tools are deterministic and offline by default. Anything that reaches the network says so in its
   name and its `--help`.
2. A tool that fetches vectors writes a provenance manifest (see [`../vectors/README.md`](../vectors/README.md)).
3. A tool that asserts a quality gate exits non-zero on failure. No tool may report success from a
   warning or a screenshot.

## Planned tools

| Tool | Purpose | Phase |
|------|---------|-------|
| `fetch-vectors` | Retrieve a pinned vector set and write its provenance manifest | P1 |
| `dependency-audit` | Assert that no crate in the shipping path pulls a C/C++ build requirement; detect `links = "…"`, `*-sys` dependencies, and `cc` / `cmake` / `bindgen` build dependencies | P1 |
| `differential` | Run the same canonical input through two backends and report per-vector agreement; archives mismatches instead of resolving them | P2 |
| `licence-inventory` | Generate the dependency licence inventory for a release | P2 |
| `unsafe-report` | Report `unsafe` occurrences per crate for review triage | P3 |

`dependency-audit` matters more than it looks: contract rule 9 requires that replacing a backend not
change the canonical input, and the practical way a violation first appears is a C dependency
silently reappearing in the graph.
