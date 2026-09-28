## Public Repository

This repository is public.

Never commit or include in generated artifacts:

- credentials, tokens, or secrets
- usernames or account identifiers
- machine names
- absolute user-specific filesystem paths
- raw Playnite databases
- private Playnite metadata
- local configuration containing identifying information

Use placeholders and environment variables in documentation and examples.


==================================================================================================================================================================================
# Project: Playnite Library Stats

## Objective

Build an extremely lightweight local exporter and static public website for displaying aggregated gaming playtime from my local Playnite library.

The priorities are:

1. Correctness
2. Extremely low resource usage
3. Minimal dependencies
4. Minimal architectural complexity
5. Privacy
6. Maintainability
7. Deterministic output

Do not over-engineer this project.

The local component should ideally be a tiny native executable that starts, performs its work, writes the result, and exits.

There should be NO permanently running background service unless there is a compelling technical reason.

---

# High-Level Architecture

```text
Playnite Library
      |
      v
Local Exporter
      |
      | Full recalculation
      v
library.json
      |
      | Scheduled once daily
      v
Git repository
      |
      v
GitHub Actions
      |
      v
Svelte + Vite
      |
      v
GitHub Pages
```

There are two separate concerns:

1. Local Playnite data extraction and aggregation.
2. Static website presentation.

Keep them loosely coupled through a small JSON contract.

---

# Local Exporter

The exporter runs on Windows 11.

It does NOT need to be a Playnite extension.

It must NOT modify Playnite.

It must treat Playnite's data as read-only.

Its job is simply:

```text
Read
→ Normalize
→ Apply rules
→ Group
→ Aggregate
→ Serialize
→ Atomic replace
→ Exit
```

It should not maintain its own database.

It should not maintain incremental state.

It should not run a web server.

It should not expose an API.

It should not remain resident after completing an export.

---

# Language Selection

Investigate using:

1. Rust
2. C#

Prefer Rust if Playnite's local database can be safely and cleanly read without requiring large or fragile dependencies.

Rust is attractive because this application should be:

- tiny
- fast
- native
- self-contained
- short-lived
- low-memory
- easy to invoke from Task Scheduler or PowerShell

Do NOT choose Rust merely because it is listed first.

Before implementation, determine:

- Playnite's current database/storage format
- safest method for read-only access
- whether accessing it directly from Rust is practical
- database locking/concurrency implications while Playnite is running
- whether Playnite-specific .NET libraries would materially simplify or make access safer

If C# provides substantially safer Playnite data access, use C#.

If both are reasonable, prefer Rust.

Document the decision briefly in an ADR or README section.

Do not introduce a large runtime/framework for convenience.

---

# Performance Requirements

This project should be extremely lightweight.

Optimize primarily for architectural simplicity rather than micro-optimizations.

Expected library size is small enough that the entire dataset can be processed in memory.

A normal export should:

- start quickly
- use minimal memory
- perform one full library scan
- write one output snapshot
- exit immediately

Avoid:

- Electron
- Python
- Node.js for the local exporter
- Docker
- databases
- ORMs unless genuinely necessary
- background services
- polling loops
- file watchers running permanently
- unnecessary async frameworks
- unnecessary dependency injection frameworks
- unnecessary abstraction layers

A few thousand game records should be considered trivial.

Prefer simple synchronous processing unless concurrency provides a concrete benefit.

---

# Playnite Access

Determine the current Playnite library/database location and storage format from authoritative Playnite documentation/source where necessary.

The exporter MUST be read-only with respect to Playnite.

Never:

- modify Playnite records
- modify Playnite configuration
- inject anything into Playnite
- install a Playnite extension
- depend on modifying Playnite's startup behavior

Handle the possibility that Playnite is currently using its database.

If safely reading while Playnite is running is problematic, document that and prefer triggering exports after Playnite closes.

Never risk corruption of the Playnite database.

---

# Export Semantics

Every exporter execution performs a COMPLETE recalculation.

`library.json` is a point-in-time snapshot.

Never incrementally mutate the previous snapshot.

Conceptually:

```text
current Playnite library
        +
current rules
        ↓
complete new dataset
        ↓
library.json.tmp
        ↓
validation
        ↓
atomic replacement
        ↓
library.json
```

This is important.

If a game:

- is removed from Playnite
- is excluded from tracking
- changes metadata
- changes genre
- changes playtime
- has its merge rule changed

the next export must naturally reflect the current state.

There should be no stale records retained from previous runs.

---

# Atomic Writes

Never write directly over the active `library.json`.

Generate something equivalent to:

```text
library.json.tmp
```

Validate serialization/output successfully.

Then atomically replace:

```text
library.json
```

If export fails, preserve the previous valid snapshot.

Return a meaningful non-zero exit code on failure.

---

# Game Data

Only extract information required by the public dataset.

Likely fields from Playnite include:

- game name
- playtime
- genres

Additional internal fields may be read if required for processing but must not automatically enter the public contract.

Never expose:

- Playnite internal IDs
- platform/store
- installation paths
- executable paths
- usernames
- account IDs
- local filesystem paths
- Playnite database paths
- private metadata

The website deliberately does NOT show which platform a game came from.

---

# Automatic Title Merging

Multiple Playnite records representing the same game should become one public record.

Default behavior:

Normalize the title conservatively and group records whose normalized titles are identical.

Safe normalization may include:

- trim leading/trailing whitespace
- case-insensitive comparison
- normalize repeated whitespace
- conservative punctuation normalization where demonstrably safe

Do NOT implement broad fuzzy matching.

Do NOT use Levenshtein similarity or AI/LLM matching.

False merges are worse than missed automatic merges.

Preserve an appropriate human-readable display title.

Example:

```text
Overwatch 2
OVERWATCH 2
Overwatch  2
```

should resolve to one game.

But similarly named sequels/editions must not be merged merely because their names look similar.

---

# Explicit Merge Rules

Special cases must be configuration-driven.

Do NOT hard-code game-specific rules into exporter source code.

Maintain a human-editable file such as:

```text
config/merge-rules.json
```

Support at minimum:

- exact aliases
- substring rules
- regex rules
- explicit exclusions
- preferred display names

Explicit rules take precedence over automatic normalized-name grouping.

Example concept:

```json
{
  "rules": [
    {
      "type": "regex",
      "pattern": "(?i)(call of duty|\\bcod\\b)",
      "target": "Call of Duty"
    }
  ]
}
```

This could allow:

```text
Call of Duty: Black Ops 6
Call of Duty: Modern Warfare III
COD HQ
```

to intentionally become:

```text
Call of Duty
```

with their playtime summed.

The rule engine itself must remain generic.

---

# Exclusion Rules

Allow specific records/titles to be excluded from public tracking.

This should use the same general configuration system where practical.

An excluded game simply does not exist in the generated snapshot.

Because every run is a full recalculation, newly excluded games must disappear automatically from the next snapshot.

---

# Merge Diagnostics

Make merge behavior inspectable without polluting public data.

Provide an optional verbose mode such as:

```text
playnite-stats export --verbose
```

Example output:

```text
Call of Duty: Black Ops 6
  -> Call of Duty
  reason: regex rule

COD HQ
  -> Call of Duty
  reason: regex rule

OVERWATCH 2
  -> Overwatch 2
  reason: normalized title
```

Normal execution should be quiet.

Do not introduce a heavyweight logging framework unless justified.

---

# Playtime

Use Playnite's stored playtime as the source of truth.

When records merge:

```text
merged playtime = sum(source record playtimes)
```

Use an integer duration internally and in the JSON contract.

Prefer seconds unless Playnite's native representation makes another integer unit substantially cleaner.

Do not store formatted durations in JSON.

Formatting belongs in the frontend.

---

# Genres

Preserve genre information.

When multiple records merge:

```text
merged genres = union(source genres)
```

Deduplicate genres.

Normalize genre values only conservatively.

The website must support:

- games by playtime
- playtime by genre
- number of games by genre

Define explicitly how multi-genre games affect genre aggregation.

Do NOT allow genre aggregation semantics to accidentally change overall total playtime.

The site's overall playtime must always equal the sum of unique merged game playtime.

---

# JSON Contract

Keep the public contract deliberately small.

Example:

```json
{
  "schemaVersion": 1,
  "generatedAt": "2026-09-28T00:00:00Z",
  "summary": {
    "gameCount": 123,
    "totalPlaytimeSeconds": 12345678
  },
  "games": [
    {
      "name": "Overwatch 2",
      "playtimeSeconds": 288000,
      "genres": [
        "Shooter"
      ]
    }
  ]
}
```

Only add fields that have an actual frontend use case.

Do not expose internal processing metadata.

Output must be deterministic.

Given identical Playnite data and identical configuration, game ordering and serialized data should be stable.

The timestamp is naturally allowed to differ.

---

# Local Snapshot Location

Do not require the continuously generated snapshot to live directly inside the Git working tree.

Prefer something similar to:

```text
%LOCALAPPDATA%\GameLog\library.json
```

This snapshot can be overwritten as frequently as necessary without creating Git activity.

Make paths configurable where appropriate.

---

# Triggering Exports

I want exports to occur when practical around Playnite usage.

Investigate the simplest reliable Windows-native mechanism.

Preferred options include:

- Windows Task Scheduler
- process exit event/triggers
- a tiny wrapper if absolutely necessary

Avoid a permanently running process solely to watch Playnite.

A scheduled periodic export is also acceptable if Windows cannot cleanly trigger on Playnite process exit without additional persistent software.

The exporter itself must always remain independently runnable:

```powershell
playnite-stats export
```

---

# Git Publication

Publishing is a SEPARATE concern from exporting.

Do not have every exporter invocation create a Git commit.

The local snapshot may be regenerated many times per day.

Once per day, a scheduled publishing script should:

1. obtain the latest valid local snapshot
2. copy it into the website's static data directory
3. compare it with the currently committed snapshot
4. commit only if meaningful data changed
5. push to GitHub

Conceptually:

```powershell
Copy-Item `
    "$env:LOCALAPPDATA\GameLog\library.json" `
    ".\web\static\data\library.json" `
    -Force

git add web/static/data/library.json

git diff --cached --quiet

if ($LASTEXITCODE -ne 0) {
    git commit -m "Update library stats $(Get-Date -Format 'yyyy-MM-dd')"
    git push
}
```

Improve this implementation as appropriate.

IMPORTANT:

If `generatedAt` is the only difference, do not create pointless commits.

Either:

- compare semantic game/stat data before publication, or
- otherwise ensure timestamp-only changes don't trigger commits.

Maximum desired automated data-update frequency:

```text
one commit per day
```

If nothing meaningful changed:

```text
zero commits
```

---

# GitHub Pages

The public website uses:

- Svelte
- Vite
- TypeScript
- GitHub Pages

Node.js is acceptable for frontend development/building.

Node.js must NOT become a dependency of the local exporter.

GitHub Actions should build and deploy the site whenever relevant website/data changes are pushed.

Also support:

```text
workflow_dispatch
```

for manual deployment.

A scheduled GitHub Pages rebuild is unnecessary if no repository data changed.

The local machine owns synchronization of new Playnite data to GitHub.

---

# Frontend

Initially keep the frontend small.

Minimum functionality:

## Summary

Display:

- total unique games
- total playtime

## Games

Display games ordered by descending playtime.

For each game:

- title
- playtime
- genres

## Genres

Display:

- genre
- game count
- aggregated playtime

Potential visualization can be added later.

Do not introduce a large UI/component framework initially.

Prefer native CSS/Svelte.

Avoid unnecessary frontend dependencies.

---

# Duration Formatting

JSON contains raw integer duration values.

Frontend handles formatting.

Examples:

```text
47 minutes
12.4 hours
1,284 hours
```

Use sensible presentation based on magnitude.

Do not compromise numerical precision in the underlying data.

---

# Repository Structure

Prefer a simple monorepo:

```text
/
├── exporter/
│   ├── src/
│   ├── tests/
│   └── ...
│
├── config/
│   └── merge-rules.json
│
├── web/
│   ├── src/
│   ├── static/
│   │   └── data/
│   │       └── library.json
│   ├── package.json
│   └── vite.config.*
│
├── scripts/
│   └── publish.ps1
│
├── .github/
│   └── workflows/
│
├── AGENTS.md
├── README.md
└── .gitignore
```

Adjust only where the chosen language/tooling has a strong conventional reason.

---

# Tests

Prioritize tests around data correctness rather than superficial coverage percentage.

At minimum test:

- identical title merging
- case-insensitive title matching
- whitespace normalization
- punctuation normalization if implemented
- titles that look similar but must remain separate
- exact aliases
- substring rules
- regex rules
- rule precedence
- explicit exclusions
- Call of Duty-style grouping
- playtime summation
- genre union
- genre deduplication
- multiple genres
- removed records disappearing on next full export
- deterministic ordering
- malformed configuration
- invalid/unavailable Playnite database
- atomic-write failure behavior

Tests must not require my real Playnite database.

Use fixtures.

---

# Dependency Policy

Dependencies need justification.

Before adding a dependency, ask:

> Does the standard library or an already-required dependency solve this cleanly?

Prefer mature, focused libraries where implementing functionality ourselves would be unsafe or wasteful.

Do not reinvent:

- database parsers
- JSON serialization
- regex engines
- filesystem atomicity primitives where suitable APIs exist

But equally, do not pull in large frameworks for trivial functionality.

Keep the exporter dependency tree intentionally small.

---

# Error Handling

Failures should be obvious and safe.

Examples:

```text
Playnite database unavailable
Configuration invalid
Database format unsupported
Snapshot serialization failed
Atomic replacement failed
```

On failure:

- return non-zero
- print a concise actionable error
- leave the previous valid snapshot untouched
- never modify Playnite

Normal successful scheduled execution should produce minimal/no output.

---

# Security and Privacy

Assume everything committed to the Git repository may become public.

Only `library.json` and intentional website assets should contain publishable information.

Never commit:

- Playnite database copies
- credentials
- GitHub tokens
- local paths
- account identifiers
- platform credentials
- debugging dumps containing private Playnite metadata

Add appropriate `.gitignore` rules immediately.

---

# Development Philosophy

Prefer:

```text
simple function
```

over:

```text
service + interface + factory + repository + dependency injection
```

unless abstraction has an actual demonstrated purpose.

This is a small deterministic ETL pipeline.

Treat it like one.

Conceptually:

```text
Extract
  Playnite records

Transform
  Filter
  Normalize
  Apply overrides
  Group
  Aggregate

Load
  JSON snapshot
```

The entire local pipeline should remain easy for one developer to understand.

---

# Initial Implementation Plan

Do NOT immediately build the entire feature set.

First:

1. Research current Playnite storage format and authoritative documentation/source.
2. Determine whether Rust or C# is the better exporter implementation.
3. Document that decision.
4. Identify any locking/concurrency concerns.
5. Scaffold the repository.
6. Implement Playnite read-only extraction.
7. Produce a minimal `library.json`.
8. Implement normalized same-title grouping.
9. Sum playtime.
10. Build the minimal Svelte page that displays it.

At that point, verify the complete path:

```text
Playnite
→ exporter
→ JSON
→ Svelte
→ Vite build
→ GitHub Pages
```

Only after that works should you add:

- configurable merge rules
- exclusions
- genre aggregation
- automatic export triggering
- scheduled Git publication
- richer frontend presentation

---

# Working Rules for Codex

Before making significant architectural decisions, inspect the existing repository first.

Do not replace working simple code with abstractions without a concrete reason.

When changing the JSON contract, update:

- exporter
- tests
- frontend types
- frontend consumption
- documentation

together.

Run relevant tests after changes.

Keep Windows as the primary local platform.

The static frontend must remain portable because it builds in GitHub Actions.

When unsure about Playnite behavior, verify against current official documentation or upstream source rather than guessing.

Do not modify Playnite upstream.

Do not clone Playnite into this repository.

If inspecting Playnite source is useful, treat upstream as external reference material.

The finished system should feel boring:

```text
tiny executable
+
small config
+
one JSON snapshot
+
small static website
+
one scheduled publishing script
```

That simplicity is intentional.
