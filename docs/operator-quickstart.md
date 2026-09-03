# vin-actor — operator quickstart

**Every step below was executed on 2026-09-03 against commit `4efacd9`**, on a
macOS workstation at load average ~13–22. Measured results are recorded inline.
If a step **prints** something other than what is shown, that is a finding — the
outputs here are observations, not aspirations. Durations are a different kind
of number: they are indicative only and will move with machine load.

There is nothing to deploy. This repo is a pure `.cljc` boundary: no runtime, no
network, no credentials. You can complete this entire quickstart offline once
dependencies are cached.

---

## 1. Get the repo

It is a west project named `vin-actor`, checked out at
`orgs/cloud-itonami/vin-actor`:

```bash
cd ~/github/com-junkawasaki
west update --fetch smart vin-actor
```

> Do not run `west update` with no arguments — it walks all 4,300+ projects.

`west` is installed at `~/.local/bin/west` on this workstation, which is **not**
on a non-interactive shell's `PATH` — under `bash -c` / `ssh` the command fails
with `No such file or directory` (exit 127) rather than anything about west.
Use the absolute path there.

The checkout is detached at the manifest pin, and its git remote is named
`cloud-itonami`, **not `origin`** (west names remotes after the org). Commands
that assume `origin` fail here — `git fetch origin` returns
`fatal: Needed a single revision`, which reads like a corrupt repository but is
only a missing remote name:

```bash
cd orgs/cloud-itonami/vin-actor
git rev-parse --short HEAD
git rev-parse --short cloud-itonami/main
```

**Measured — both printed `4efacd9`**, the commit this quickstart documents.
Both values move as the repo advances; what the check is for is that they
*agree*. If they differ, the checkout is not on the manifest pin and every
measurement below is being taken against a different tree than this document
describes.

---

## 2. Exercise the boundary without a JVM  ← start here

This is the fastest way to see what the repo does. `nbb` loads the `.cljc`
directly; no dependency resolution, no JVM.

```bash
nbb --classpath src -e '
(require (quote [vin.murakumo :as m]))
(println "cells:" (count m/cell-specs))
(let [blocked (m/cell-plan :vehicle {})]
  (println "no attestations   ->" (:status blocked)
           "| effects" (count (:effects blocked))
           "| missing" (count (:missing-gates blocked))))
(let [att   (into {} (map (fn [g] [g true])) m/common-gates)
      ready (m/cell-plan :vehicle {:attestations att :request-id "req-1"})]
  (println "all gates attested ->" (:status ready)
           "| effects" (count (:effects ready)))
  (println "collection:" (:collection (first (:records ready)))))'
```

**Measured — 0.35 s wall:**

```
cells: 15
no attestations   -> :blocked | effects 0 | missing 7
all gates attested -> :ready | effects 1
collection: com.etzhayyim.vin.vehicle
```

Both directions matter. The first line is the gate actually closing: `:blocked`
with an **empty** effect vector, not a plan that merely says "blocked" while
still emitting records. The second is it opening once all 7 baseline gates carry
a truthy attestation.

Note the collection on the last line — see §5.

---

## 3. Run the test suite (JVM)

```bash
clojure -M:test
```

**Measured — exit 0, 4.0 s wall with dependencies already cached (load ~13):**

```
Ran 9 tests containing 213 assertions.
0 failures, 0 errors.
```

The first run on a clean machine is slower: it resolves the
`io.github.cognitect-labs/test-runner` git dependency, which needs network.

The suite introspects `cell-specs` instead of hardcoding cell names, so it keeps
holding if the manifest's cell list changes. It covers `gate-value` (map and set
attestations, keyword and string keys), `missing-gates`, `put-record-effect`
shape, `records-for` (one record per declared collection, plus explicit record
override), `cell-plan` in both directions, the throw on an unknown cell, and
`all-cell-plans` coverage.

---

## 4. Lint

```bash
clojure -M:lint
```

**Measured — exit 0. The one line that must match is the warning:**

```
src/vin/murakumo.cljc:159:14: warning: unused binding input
linting took <varies>, errors: 0, warnings: 1
```

`errors: 0, warnings: 1` is the assertion; the elapsed figure is not. Two runs
on the same tree on 2026-09-03 reported 1115 ms and 593 ms of linting, inside
18.3 s of wall clock — nearly all of which is JVM start and dependency
resolution, not analysis.

The alias fails only on `error` level, so the one warning does not fail it. The
warning is real: `records-for` destructures `:as input` and never uses it.

---

## 5. Re-measure the three-plane divergence

The README states that the collection names this repo emits and the ones its
manifest declares **do not intersect**. Verify it rather than trusting it:

```bash
nbb --classpath src -e '
(require (quote [vin.murakumo :as m])
         (quote [clojure.set :as set])
         (quote ["fs" :as fs]))
(let [emitted  (set (mapcat :collections (vals m/cell-specs)))
      manifest (js->clj (js/JSON.parse (fs/readFileSync "actor-manifest.jsonld" "utf8")))
      declared (set (concat (get-in manifest ["triggers" "subscribeRepos" "collections"])
                            (get manifest "requiredCollections")))]
  (println "emitted" (count emitted)
           "declared" (count declared)
           "intersection" (count (set/intersection emitted declared))))'
```

**Measured:**

```
emitted 15 declared 10 intersection 0
```

This reads the `:collections` the cells actually declare, by loading the
namespace — it does **not** re-derive the names from the cell keys. That
distinction is what makes the check able to fail at all: a version that rebuilt
`com.etzhayyim.vin.<cell-key>` with a regex would print `intersection 0`
no matter what the source said, including after the divergence was fixed.

**Verified in both directions on 2026-09-03.** Against a copy of `murakumo.cljc`
whose `:collections` are derived from `:legacy-cell` (dashes to dots) instead of
from the cell key, the same command prints:

```
PATCHED COPY -> emitted 15 declared 10 intersection 10
```

So `intersection 0` is a measurement, not a constant.

And the actor's own DID, read from the three places that state it:

```bash
node -e '
const fs=require("fs");
const did=JSON.parse(fs.readFileSync(".well-known/did.json","utf8")).id;
const mid=JSON.parse(fs.readFileSync("actor-manifest.jsonld","utf8"))["@id"];
const src=fs.readFileSync("src/vin/murakumo.cljc","utf8")
            .match(/\(def actor-did\s+"([^"]+)"/)[1];
console.log("did.json  id      :", did);
console.log("manifest  @id     :", mid);
console.log("source    actor-did:", src);
console.log("distinct          :", new Set([did,mid,src]).size);'
```

**Measured — the actor's own identity resolves to two different DIDs:**

```
did.json  id      : did:web:etzhayyim.com:actor:vin
manifest  @id     : did:web:vin.etzhayyim.com
source    actor-did: did:web:vin.etzhayyim.com
distinct          : 2
```

The cause is visible in the history — the migration touched one file:

```bash
git show --stat ba1f9ac | tail -3
#  .well-known/did.json | 8 ++++----
#  1 file changed, 4 insertions(+), 4 deletions(-)
```

A non-zero intersection, or a single DID, means someone reconciled the planes —
update `docs/adr/0001-three-planes-disagree.edn` and this section together.

---

## 6. What this quickstart deliberately does not cover

`CLAUDE.md` documents a deployed vehicle-intelligence platform: ~40,000 WMI
codes, a 13-label SQL graph, 12 MCP commands, seed cron schedules, WIT exports
and a max-flow path planner. **None of that is in this repository** and none of
it is reachable from here. This quickstart stops at the boundary because the
boundary is all that is here.

Do not treat a green run of §2–§4 as evidence that the platform works.
