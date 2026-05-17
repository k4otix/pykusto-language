# pykusto-language

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/downloads/)
[![.NET 8.0+](https://img.shields.io/badge/.NET-8.0+-purple.svg)](https://dotnet.microsoft.com/download/dotnet/8.0)

`pykusto-language` is a two-tier Python library for Microsoft's KQL parser
(the `Microsoft.Azure.Kusto.Language` NuGet package), bridged via `pythonnet`.
It exposes the same parser used by Azure Data Explorer, Azure Monitor, and
Microsoft Sentinel.

> **Not affiliated with Microsoft.** This is an independent open-source project
> that wraps Microsoft's publicly distributed Apache 2.0–licensed library.

## Two tiers

| Tier | Install | Adds |
|---|---|---|
| 1 — thin .NET wrapper | `pip install pykusto-language` | `KustoCode`, `KustoQuery`, `parse`, `format_query`, `validate`. Full Microsoft API surface via `pykusto_language.bridge.*`. |
| 2 — semantic IR | `pip install 'pykusto-language[ir]'` | An **intermediate representation** (IR) of the parsed query as pydantic models — typed operators, expressions, and source spans — plus a schema binder for column-flow analysis. |

The two tiers compose: `KustoQuery.to_ir()` builds the IR from the same parsed
AST tier 1 produced — there is no double parse.

```python
from pykusto_language import parse

q = parse("StormEvents | where EventType == 'Tornado' | summarize count() by State")
ir = q.to_ir()                # tier 2: pydantic IR
ops = q.get_operator_chain()  # tier 1: raw .NET AST walk
```

## What's the IR?

**IR** stands for *intermediate representation*: a higher-level model of a
parsed KQL query than the raw .NET syntax tree exposed by tier 1.

Tier 1 hands you the unmodified Microsoft AST — useful for source-level
rewrites and direct interop, hostile to Python work (every traversal goes
through `pythonnet`, every node is a .NET object). Tier 2's IR is the same
query expressed as **pydantic models**: a `Pipeline` of typed operators
(`FilterOp`, `JoinOp`, `SummarizeOp`, …), each carrying typed expressions
(`BinOp`, `FuncCall`, `SetMembership`, …) and source spans.

The IR is the **semantic layer**:

- **Typed.** Every node is a pydantic model. Operator and expression families
  are explicit, so analyzers dispatch by `isinstance(op, FilterOp)` rather
  than by string matching on `node.Kind`.
- **Serializable.** `model_dump_json` round-trips losslessly. No .NET object
  references leak out — the IR can cross process boundaries.
- **Schema-aware.** `SchemaAttacher(schemas)` walks the pipeline and fills
  `result_type` and column→table provenance, including across joins, lookups,
  and unions.
- **Hash-stable.** Two queries that differ only in whitespace produce the same
  `structural_hash` — useful for de-duplication.
- **Explicit fallbacks.** Anything the builder doesn't model surfaces as
  `UnknownExpr` / `UnknownSource` rather than a silent gap. A coverage audit
  fails CI when those grow beyond a baseline.

Use the IR for analyzers (lint, lineage, anti-pattern detection), query
introspection (UI displays, JSON APIs), or anywhere you'd rather work with
`op.predicate.left.name` than `node.GetChild(0).GetChild(2).Name.ToString()`.

## Stability policy

**Tier 1** — `pykusto_language` top-level surface (`parse`, `format_query`,
`validate`, `KustoQuery`, the reflection helpers, the `pykusto` CLI) follows
[Semantic Versioning](https://semver.org/). What counts as a breaking change:

- Renaming or removing a public function, class, or method.
- Changing a public function's signature (parameter names, order, defaults).
- Removing a CLI subcommand, renaming a flag, or changing its default value.
- Changing the documented exit codes of any CLI subcommand.
- Changing the JSON output shape of `pykusto parse --json`, `pykusto
  validate --json`, or `IRBuilder.build(...).model_dump_json()`.
- Removing or renaming a re-export from `pykusto_language` or
  `pykusto_language.ir`.

**Tier 2** — `pykusto_language.ir.*` (the pydantic IR, the binder, the IR
walker utilities) is on a pre-1.0 track. Minor breaking changes are possible
at minor versions until the IR survives one Kusto.Language.dll upgrade cycle
without breaking. Each break is called out explicitly in `CHANGELOG.md`.
Specifically, the following may change in a tier-2 minor release:

- IR node field names, defaults, or types.
- The set of recognized `SyntaxKind` dispatch entries.
- The `Pipeline.source` union members (e.g. `TableRef`, `LetRef`, `ImplicitSource`).
- Binder enrichment fields (`result_type`, `nullable`, `canonical_form`).

When tier 2 lands a breaking change, the CHANGELOG names the affected
classes / fields and gives the migration. We do not silently shift shapes.

## Capabilities

### Tier 1 — thin wrapper (always available)

- **Parse and format** KQL via Microsoft's public `KustoCodeService` API.
- **Validate** queries with structured diagnostics (severity, code, offset).
- **Semantic binding** when called with a schema — resolves columns and tables
  through symbols. Falls back to a fast syntactic walk otherwise.
- **AST analyzers**: referenced tables (including `database()`/`cluster()`
  cross-cluster refs, joins, unions, lookups, facets), referenced columns,
  operator chain, operator counts, structural hash, time-range expressions.
- **Surgical rewrites**: rename tables across every reference position.
- **JSON interop**: export the AST to a Python dict or JSON.
- **Runtime reflection** (`pykusto_language.reflection`): self-updating lists
  of KQL functions — `time_functions()`, `aggregate_functions()`,
  `string_functions()`, `scalar_functions()` for categorized listings,
  `all_function_names()` for the full set, plus `syntax_kinds()` for the
  SyntaxKind enum.

### Tier 2 — semantic IR

- **`QueryIR`** — pydantic root model holding a `Pipeline` of typed operators
  (`FilterOp`, `SummarizeOp`, `JoinOp`, `LookupOp`, …) and typed expressions
  (`BinOp`, `FuncCall`, `SetMembership`, …).
- **`IRBuilder.build(query)`** — parse, bind, and build the IR in one call.
- **`IRBuilder.build_from_code(code)`** — build from a pre-parsed `KustoCode`.
  `KustoQuery.to_ir()` uses this so callers don't pay for two parses.
- **`SchemaAttacher(schemas)`** — annotates the IR with column types and
  owning-table provenance from a flat `{table: {col: type}}` dict.
- **`QueryIR.to_llm_dict()`** — lossy projection optimized for handing the
  IR to a language model: every node carries a `kind` discriminator, spans
  and defaulted fields are stripped, and `polarity` is collapsed into
  natural KQL operators (`!=`, `!in`, `!between`). Roughly 50% smaller than
  `model_dump_json()` on typical queries. See `examples/llm_view.py`.
- **`UnknownExpr` / `UnknownSource`** — explicit fallback nodes for shapes
  the builder doesn't model. The coverage audit (`scripts/audit_syntax_kinds.py`)
  fails CI when new shapes appear after a Kusto.Language DLL upgrade.

## Prerequisites

- **Python 3.10+**
- **[.NET 8.0+ runtime](https://dotnet.microsoft.com/download/dotnet/8.0)**

### macOS / Homebrew

If you installed `dotnet` via Homebrew, the runtime layout differs from
Microsoft's installer (`libhostfxr.dylib` lives under `libexec/`, not `bin/`).
The bridge auto-detects this. If detection fails, set `DOTNET_ROOT` explicitly:

```bash
export DOTNET_ROOT=/opt/homebrew/opt/dotnet/libexec   # Apple Silicon
export DOTNET_ROOT=/usr/local/opt/dotnet/libexec      # Intel
```

## Installation

```bash
pip install pykusto-language           # tier 1: thin .NET wrapper
pip install 'pykusto-language[ir]'     # tier 1 + tier 2: semantic IR (adds pydantic)
```

## Quick start

```python
from pykusto_language import parse, format_query, validate

query = "SecurityEvent | where EventID == 4624 | project TimeGenerated, Account"

print(format_query(query))

result = parse(query)
print(result.get_referenced_tables())          # {'SecurityEvent'}
print(result.get_referenced_columns())         # {'EventID', 'TimeGenerated', 'Account'}
print(result.get_structural_hash()[:16])

# Semantic binding via a schema enables column-aware analysis:
schema = {"SecurityEvent": {"EventID": "long", "TimeGenerated": "datetime", "Account": "string"}}
bound = parse(query, schema=schema)
assert bound.has_semantics
```

With the `[ir]` extra installed, the same `KustoQuery` builds a pydantic IR:

```python
from pykusto_language import parse
from pykusto_language.ir import FilterOp, SchemaAttacher

schema = {"SecurityEvent": {"EventID": "long", "TimeGenerated": "datetime", "Account": "string"}}
ir = parse("SecurityEvent | where EventID == 4624", schema=schema).to_ir()

SchemaAttacher(schema).enrich(ir)              # fills column types + table provenance

for op in ir.main_pipeline.operators:
    if isinstance(op, FilterOp):
        print(op.predicate.canonical_form)     # EventID == 4624
        print(op.predicate.left.table)         # SecurityEvent
        print(op.predicate.left.result_type)   # KustoType.LONG ('long')
```

## CLI

The `pykusto` console script ships with the base install:

```bash
pykusto version                          # print package version
pykusto format query.kql                 # reformat to canonical form
pykusto validate query.kql               # print parser diagnostics
pykusto validate --json query.kql        # diagnostics as JSON
pykusto parse query.kql                  # print the .NET AST
pykusto parse --ir query.kql             # print the pydantic IR (requires [ir])
pykusto parse --ir --json query.kql      # serializable IR
```

All subcommands also read from stdin when `file` is `-` or omitted. Exit codes:
`0` success, `1` input had Error-severity diagnostics or a runtime failure,
`2` usage error (bad flags, missing file, or missing `[ir]` extras for
`parse --ir`).

## Architecture

1. **Bridge** (`bridge.py`): initializes the .NET CLR via `pythonnet` with
   cross-platform `DOTNET_ROOT` auto-detection, exposing only Microsoft's
   public API surface (`KustoCode`, `KustoCodeService`, `GlobalState`,
   `TableSymbol`, …).
2. **Engine**: the unmodified Microsoft parser DLL — see
   [Verifying the bundled DLL](#verifying-the-bundled-dll).
3. **Tier-1 Python layer**: `services.py`, `core.py`, `utils/analysis.py` —
   `parse`/`format_query`/`validate` entry points and the `KustoQuery`
   analyzer. AST walks; no pydantic dependency.
4. **Tier-2 IR layer** (`pykusto_language.ir`, optional): pydantic `QueryIR`,
   `IRBuilder`, and `SchemaAttacher`. Activated by
   `pip install 'pykusto-language[ir]'`.

### When to use tier 1 vs tier 2

- **Tier 1**: format/lint pipelines, IDE integrations, simple analyzers that
  ask "what tables / columns / operators does this query touch?"
- **Tier 2**: structured analyzers (lineage, anti-pattern detection,
  contradiction checks), JSON-serializable query representations for APIs and
  UIs, schema-aware column-flow analysis. Anything that wants pydantic models
  with source-span provenance instead of raw .NET AST nodes.

## Verifying the bundled DLL

Every release pins the bundled DLL to a specific NuGet package version with a
SHA-256 hash. The pin lives in:

- `src/pykusto_language/bin/VERSION.txt` — package, version, sha256, refresh
  timestamp.
- `pyproject.toml` under `[tool.pykusto-language] kusto_language_version`.

### Offline hash check

```bash
shasum -a 256 src/pykusto_language/bin/Kusto.Language.dll
cat src/pykusto_language/bin/VERSION.txt
# The two sha256 values must match.
```

### Re-fetch from nuget.org and compare

`scripts/verify_dll.py` is pure Python (no `dotnet` required). It downloads
the pinned `Microsoft.Azure.Kusto.Language` package from nuget.org, hashes
every `Kusto.Language.dll` inside, and confirms one is byte-identical to the
bundled file.

```bash
python scripts/verify_dll.py
```

This is a **trust-on-first-use** model: it confirms the bundled DLL matches
what nuget.org currently serves over TLS. It does not validate NuGet's
signed-package signature. If your policy requires signature validation, run
`nuget verify -All <path-to-nupkg>` against a freshly fetched package.

### Refresh the bundled DLL

If you'd rather use a DLL you fetched yourself:

```bash
python scripts/refresh_dll.py            # uses the pinned version
python scripts/refresh_dll.py --version 12.3.2 --pin
```

This requires `dotnet` 8.0+, runs `dotnet publish` against a temporary
`.csproj`, copies the resulting `Kusto.Language.dll` into `bin/`, and updates
`VERSION.txt`.

## Development

```bash
git clone https://github.com/k4otix/pykusto-language.git
cd pykusto-language
pip install -e ".[dev]"

pytest
ruff check src tests scripts
mypy src
```

See [CONTRIBUTING.md](CONTRIBUTING.md) for the full workflow.

## License

Apache License 2.0. See [LICENSE](LICENSE), [NOTICE.md](NOTICE.md), and
[THIRD-PARTY-NOTICES.md](THIRD-PARTY-NOTICES.md). The bundled
`Kusto.Language.dll` is owned by Microsoft Corporation and redistributed
unmodified under Apache 2.0.

## Trademark notice

"Kusto", "KQL", "Microsoft", "Azure Data Explorer", "Azure Monitor", and
"Microsoft Sentinel" are trademarks of Microsoft Corporation. References to
those trademarks are nominative and used only to identify the upstream library
this package wraps. Apache License 2.0 §6 does not grant trademark rights;
nothing in this distribution should be construed as a trademark license.
