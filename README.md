# vin-actor

**The pure `.cljc` actor boundary for the VIN / vehicle-registration actor —
a deny-by-default planner that turns a cell invocation into a set of
`:mst/put-record` effects, and refuses to emit any effect until every required
attestation gate is present.**

This repository is **not** the VIN platform. It holds the boundary only: ten
tracked files, one namespace, no network, no runtime, no host. The name
`vin-actor` names the subject (VIN — the ISO 3779 vehicle identification
number) and the role, but not the shape: nothing here decodes a VIN.

## What is actually here

`git ls-files`, measured 2026-09-03 at commit `4efacd9` — all ten files:

| path | what it is |
|---|---|
| `src/vin/murakumo.cljc` | the only implementation — one namespace, `vin.murakumo` |
| `test/vin/murakumo_test.cljc` | its contract tests (9 tests / 213 assertions) |
| `deps.edn` | `:test` (cognitect test-runner) / `:lint` (clj-kondo) |
| `actor-manifest.jsonld` | actor declaration — capabilities, triggers, runtime, pipelines |
| `.well-known/did.json` | the published DID document |
| `storage-profile.edn` | `:kotoba/local-agent-kagi-chunks-v1` |
| `CLAUDE.md` | **describes the deployed platform, which does not live here** |
| `NOTICE` | Apache-2.0 + etzhayyim Charter Compliance Rider |
| `.gitignore`, `.nojekyll` | housekeeping |

`vin.murakumo` is one data table and eight functions:

- **`cell-specs`** — 15 cells, each declaring `:legacy-cell`, `:phase`,
  `:murakumo-node`, `:collections` and `:required-gates`.
- **`gate-value` / `missing-gates`** — attestation lookup that accepts either a
  map or a set, keyed by keyword or by string.
- **`records-for` / `put-record-effect`** — one record per declared collection,
  stamped with `:actorDid`, `:legacyCell`, `:scaffold true`.
- **`cell-plan` / `all-cell-plans`** — the boundary itself.
- **`collection` / `safe-rkey`** — name derivation. `collection` is the subject
  of §"Three planes" below; `safe-rkey` sanitises a record key to
  `[A-Za-z0-9._~-]` and falls back to `"unknown"` rather than emitting a blank.

The only dependency is `clojure.string`. There is no function here that opens a
socket, reads a file, or starts a process.

### The gate is the point

`cell-plan` is deny-by-default. With no attestations it returns `:blocked` and
**an empty effect vector** — it does not emit a single record:

```
no attestations    -> :blocked  effects 0  missing 7
all gates attested -> :ready    effects 1
```

All 15 cells require the same 7 baseline gates (`common-gates`): council
charter attestation, no platform-held key, no probing, Murakumo-only inference,
DID-primary, append-only, Kotoba-only substrate. An unknown cell key throws
rather than quietly planning nothing.

Reproduce both directions in §2 of the quickstart — it takes about a second and
needs no JVM.

## Three planes describe this actor, and they do not agree

Measured 2026-09-03 at `4efacd9`. **Nothing here has been reconciled**: which
plane is canonical is an owner decision, not something a docs change may settle.
See [`docs/adr/0001-three-planes-disagree.edn`](docs/adr/0001-three-planes-disagree.edn)
for the decision to record rather than reconcile, and §5 of the quickstart for
the commands that re-measure each row.

| Question | `src/vin/murakumo.cljc` | `actor-manifest.jsonld` | `.well-known/did.json` | `CLAUDE.md` |
|---|---|---|---|---|
| actor DID | `did:web:vin.etzhayyim.com` | same | **`did:web:etzhayyim.com:actor:vin`** | same as source |
| runtime | (none) | `k8s-langserver` | — | **`Single Worker`** |
| UI | — | `yoro` | — | **appview (Protocol Canvas)** |
| collections | `com.etzhayyim.vin.*` (15) | `com.etzhayyim.apps.*` (10) | — | — |

**The collection-name sets intersect in zero places.** Every name the planner
emits is `com.etzhayyim.vin.<lowercased-cell-key>`; every name the manifest
declares is `com.etzhayyim.apps.<app>.<camelCase>`. For the `vehicle` cell the
whole difference is one missing `apps.` segment — emitted
`com.etzhayyim.vin.vehicle` against declared `com.etzhayyim.apps.vin.vehicle`.
The correct name is present in the data — `:legacy-cell` preserves it — but
`collection` re-derives a different one from the cell key instead of reading it.

So a `:ready` plan from this repo currently targets collections that no declared
trigger subscribes to. That is a real finding, not a lint nit, and it is why
this README states it instead of describing the actor as wired.

The DID divergence has a traceable cause: commit `ba1f9ac`
(*chore(identity): migrate did:web to etzhayyim.com scheme*, 2026-07-02)
changed `.well-known/did.json` **and nothing else** — four lines in one file.
The source, the manifest and `CLAUDE.md` were never migrated with it. The old
value survives in `alsoKnownAs` as the handle `at://vin.etzhayyim.com`.

## What CLAUDE.md describes that is not here

`CLAUDE.md` documents a deployed vehicle-intelligence platform: a role-based DID
hierarchy of ~40,000 WMI codes, a SQL graph with 13 node labels, 12 MCP
commands (`decode_vin`, `lookup_plate`, `collect_recall`, …), seed cron
schedules, WIT exports, a production-topology extension and a max-flow path
planner. **None of it is in this repository** and none of it is reachable from
here. Read it as a description of the upstream deployment, not of this tree.

## Running it

See **[`docs/operator-quickstart.md`](docs/operator-quickstart.md)**. Every step
there was executed on 2026-09-03 and carries its measured result.

## Provenance

Apache-2.0 with the etzhayyim Charter Compliance Rider (`NOTICE`).
Storage profile `:kotoba/local-agent-kagi-chunks-v1` (`storage-profile.edn`):
local query, append-only transactions, kagi-chunked EDN payloads, kotobase head.
