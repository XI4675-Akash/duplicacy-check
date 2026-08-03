# Challenge 3: Query and Modify Data with CLC

## Context

Challenges 1 and 2 left two things running in the background: the Hazelcast member (cluster `dev`, port 5701) and the Management Center container (connected, dashboard on port 8080) — both in the one reused terminal tab's environment. This challenge introduces the third and final free tool, CLC, and uses it to actually put data into the cluster: a SQL-mapped `currency` table and a schemaless `greeting` map, written and read entirely from the command line. It deliberately does not touch Management Center at all — no `docker` commands, no container restarts — because Challenge 4 is where the learner switches back to the MC UI to watch this exact data appear live. This challenge's setup must still defensively confirm MC is still up (per the same "verify, don't assume" contract Challenge 2 established for Hazelcast), purely so Challenge 4 doesn't inherit a dead container; the learner never sees or interacts with MC in this challenge itself, and the Management Center tab is kept present (not removed) so the environment feels continuous across Challenges 2-3-4.

## Prior State

- **Services running:** the Hazelcast member (cluster `dev`, 1 member, port 5701, from Challenge 1) and the Management Center container (detached, `--network host`, connected to cluster `dev`, serving port 8080, from Challenge 2). Neither is guaranteed to have survived the challenge boundary — see Prerequisites and Setup below.
- **Files present:** `~/hazelcast/hazelcast-5.7.0.jar`, `~/hazelcast/hazelcast.log`; `hazelcast/management-center:5.11.0` image and container; CLC v5.3.3 on `PATH` (track setup, unused until now). No CLC configuration or map data exists yet anywhere — this challenge is the first to create either.
- **State from previous challenges:** the learner has started a backgrounded JVM process and a detached container, read two different services' logs to confirm state, and seen that neither Community Edition nor Management Center's free tier requires a license key. They know: member, cluster, cluster name `dev`, port 5701, and that Management Center is a separate observer process, not a member.
- **Terminal tab:** the same `Terminal` tab from Challenges 1-2, unchanged. **Management Center tab:** also carried forward unchanged and left visible/reachable, even though nothing in this challenge is graded through it — preserving the continuity Challenge 2's closing text promised ("leave Management Center open").
- **Terminology already known (referenced, not re-taught):** member, cluster, cluster name `dev`, port 5701, backgrounding processes so the terminal stays free (`nohup ... &`, `docker run -d`), reading a service's log/state to confirm it's alive, basic SQL (`SELECT`/`INSERT`/`WHERE` — assumed track prerequisite, applied here for the first time against Hazelcast specifically, not re-taught as SQL).

## Prerequisites

| Capability (functional at challenge start) | Host | Verify command | Provided by |
|--------------------------------------------|------|-----------------|-------------|
| Hazelcast member alive: cluster `dev`, one member, listening on 5701 | workstation | `pgrep -f "hazelcast-5.7.0.jar" >/dev/null && grep -q "Members {size:1" ~/hazelcast/hazelcast.log && ss -tln \| grep -q ':5701'` | 01-run-hazelcast-community-edition (defensively re-asserted by **this challenge's** setup, per the committed cross-challenge contract in Challenge 1's plan) |
| Management Center still running, connected to cluster `dev`, serving on 8080 | workstation | `CID=$(docker ps -q --filter ancestor=hazelcast/management-center:5.11.0 --filter status=running \| head -n1); [ -n "$CID" ] && docker logs "$CID" 2>&1 \| grep -qi "connected to cluster dev" && curl -sf -o /dev/null -m 5 http://localhost:8080/` | 02-monitor-with-management-center (defensively re-asserted by **this challenge's** setup — Challenge 4 depends on it, and this challenge itself never starts/stops/recreates it) |
| CLC v5.3.3 resolves via `clc` on `PATH` | workstation | `clc version \| grep -q 5.3.3` | track setup |
| `$HOME` is writable, so CLC can persist named configs under its own config store (`$(clc home)/configs/...`) | workstation | `test -w "$HOME"` | this challenge (first challenge to actually need this exercised) |
| No leftover CLC config named `dev`, and no leftover data in the `currency`/`greeting` maps, from a previous attempt at this challenge | workstation | `[ ! -d "$(clc home configs dev)" ]` (config); map-clear commands in setup succeed (data) | this challenge |

Note: this challenge's setup script does not run `clc config add dev ...`, does not create the `currency`/`greeting` maps' data, and does not touch the Management Center container beyond confirming it's alive. Adding the named connection and populating both maps are the learner's graded actions, exactly as starting the member (Ch1) and starting Management Center (Ch2) were.

## Assignment Outline

1. **Confirm which CLC you're running, and why it matters.**
   ```bash,run
   clc version
   ```
   You should see `5.3.3`. That's not the newest CLC release — and that's deliberate. Hazelcast still ships newer CLC builds for free, but starting with v5.3.4 those builds are binary-only: no source code is published alongside them. v5.3.3 is the last CLC release Hazelcast published under the Apache 2.0 license with source available. Since this whole track is about proving what you can do with genuinely open-source Hazelcast tooling — not just free-to-download tooling — every command below runs through this specific, fully open-source build.

2. **Tell CLC how to reach the cluster.**
   ```bash,run
   clc config add dev cluster.name=dev cluster.address=localhost:5701
   ```
   Two different things are both called `dev` here on purpose — don't let that trip you up. The `dev` right after `config add` is just a nickname you're choosing for this saved connection; you could call it anything (`myconn`, `local`) and nothing below would change. `cluster.name=dev` is the actual Hazelcast cluster name you read in Challenge 1's log — CLC will refuse to connect if this doesn't match the running cluster. They're spelled the same here only because it was convenient to name the connection after the cluster it points to.

3. **Two ways to talk to that connection.** Every command below is a **one-shot** command: `clc -c dev ...` runs once against the `dev` connection and returns you straight to the prompt, so each line here is safe to copy/paste on its own. CLC also offers an **interactive shell** — run `clc -c dev` by itself, and you land in a `>` prompt where you can type statement after statement until you type `\exit`. Both reach the identical cluster the identical way; the shell is just more convenient once you're running many statements in a row. Feel free to try it, but everything graded in this challenge is shown below in one-shot form.

4. **Put a SQL schema on top of a map.** A Hazelcast map (an `IMap`) is schemaless key-value storage by default — but `CREATE MAPPING` lets you describe columns for one, so you can then use ordinary `INSERT`/`SELECT` against it. Create a mapping for a `currency` map:
   ```bash,run
   clc -c dev sql "CREATE OR REPLACE MAPPING currency (__key INT, code VARCHAR, name VARCHAR) TYPE IMap OPTIONS ('keyFormat'='int','valueFormat'='json-flat');"
   ```
   `OR REPLACE` means this is always safe to re-run if you make a typo — it won't complain that the mapping already exists.

5. **Insert rows with ordinary SQL.**
   ```bash,run
   clc -c dev sql "INSERT INTO currency VALUES (1,'USD','US Dollar'),(2,'EUR','Euro'),(3,'GBP','British Pound'),(4,'JPY','Japanese Yen'),(5,'INR','Indian Rupee');"
   ```
   This is a real `INSERT INTO`, exactly like you'd write against any relational table — underneath, it's writing five entries into the `currency` `IMap`, keyed `1` through `5`. Unlike `CREATE OR REPLACE MAPPING` above, `INSERT INTO` is append-only: if you run this exact line a second time, it will error on a duplicate key instead of silently overwriting. That's expected — it means your data is already there; there's no need to re-run it.

6. **Read it back with `SELECT`.**
   ```bash,run
   clc -c dev sql -f table "SELECT * FROM currency ORDER BY code;"
   ```
   `-f table` formats the result as readable columns. You should see all five currencies you just inserted.

7. **Now skip SQL entirely — talk to a map directly.** CLC can also read and write a map's entries with no schema and no SQL at all, using `clc map set`/`clc map get`. This works against *any* map, including ones that have never had a `CREATE MAPPING` run against them — this is the same kind of storage as `currency`, just accessed a different way. Populate a small `greeting` map:
   ```bash,run
   clc -c dev map set --name greeting en Hello
   clc -c dev map get --name greeting en
   ```
   ```bash,run
   clc -c dev map set --name greeting fr Bonjour
   clc -c dev map get --name greeting fr
   ```
   Each `get` should print back exactly the value you just `set` for that key.

8. **Completion / closing.** In eight commands, you've configured a CLI connection, created a SQL-queryable table backed by a distributed map, inserted and queried real rows, and read/written a second map directly by key — all without a license key, a sign-up, or a GUI. Preview Challenge 4: "Management Center is still open in its tab from Challenge 2. Next, you'll come back to CLC to change this data further, then switch to that dashboard and watch your changes show up live." Completion marker: "Once `clc -c dev sql \"SELECT * FROM currency;\"` shows your five currencies and `clc -c dev map get --name greeting en`/`fr` return `Hello`/`Bonjour`, click Check."

## Tabs

| Title | Type | Target | Notes |
|-------|------|--------|-------|
| Terminal | terminal | hostname: workstation | Same tab as Challenges 1-2, unchanged (`shell: /bin/bash`, `workdir: /root`). All `clc` commands run here. |
| Management Center | service | hostname: workstation, port: 8080 | Same tab as Challenge 2, unchanged, kept present in this challenge's own config for UI continuity even though no step here is graded through it (Challenge 4 is where the learner returns to it). Setup still defensively confirms the container is alive (see Prerequisites), but this challenge never opens or checks the tab itself. |

No editor tab (per track plan — no file editing occurs anywhere in this track; CLC configuration happens entirely via CLI flags).

## Scripts

### Setup (`setup-workstation`)

```bash
#!/bin/bash
set -euxo pipefail

# --- Defensive precondition carried forward from Challenge 1 (committed contract) ---
if ! pgrep -f "hazelcast-5.7.0.jar" >/dev/null 2>&1; then
  rm -f ~/hazelcast/hazelcast.log
  nohup java -jar ~/hazelcast/hazelcast-5.7.0.jar > ~/hazelcast/hazelcast.log 2>&1 &
  disown
  for i in $(seq 1 30); do
    grep -q "Members {size:1" ~/hazelcast/hazelcast.log 2>/dev/null \
      && ss -tln 2>/dev/null | grep -q ':5701' && break
    sleep 1
  done
fi

# --- Defensive precondition adapted from Challenge 2's solve-script pattern (Ch2 itself
# never needed a "restart if not running" block in its own setup, since starting MC is
# Ch2's own graded action) -- Management Center must still be up here because Challenge 4
# depends on it. This challenge never recreates or otherwise touches the container if it
# is already running -- restart only if it's not.
if [ -z "$(docker ps -q --filter ancestor=hazelcast/management-center:5.11.0 --filter status=running)" ]; then
  docker rm -f $(docker ps -aq --filter ancestor=hazelcast/management-center:5.11.0 2>/dev/null) >/dev/null 2>&1 || true
  docker run -d --network host \
    --env MC_DEFAULT_CLUSTER=dev \
    --env MC_DEFAULT_CLUSTER_MEMBERS=localhost:5701 \
    hazelcast/management-center:5.11.0
  CID="$(docker ps -q --filter ancestor=hazelcast/management-center:5.11.0 --filter status=running | head -n1)"
  for i in $(seq 1 45); do
    docker logs "$CID" 2>&1 | grep -qi "connected to cluster dev" \
      && curl -sf -o /dev/null -m 2 http://localhost:8080/ && break
    sleep 2
  done
fi

# --- This challenge's own clean slate ---
# 1) Remove any stale "dev" CLC config from a previous attempt (hot start / re-run), so
#    the learner's own `clc config add dev ...` in Step 2 is the first time it exists.
STALE_DEV_DIR="$(clc home configs dev 2>/dev/null || true)"
[ -n "$STALE_DEV_DIR" ] && rm -rf "$STALE_DEV_DIR"

# 2) Clear any stale mapping/data left in the maps this challenge writes to, using a
#    private, throwaway CLC config -- deliberately never named "dev", so it can never
#    satisfy Step 2's own check on its own. Hazelcast map data lives in the long-running
#    member process, not in the terminal session, so it persists across challenge/hot-
#    start boundaries unless explicitly cleared here.
clc config add __ch3_setup cluster.name=dev cluster.address=localhost:5701 >/dev/null
clc -c __ch3_setup sql "DROP MAPPING IF EXISTS currency;" >/dev/null 2>&1 || true
clc -c __ch3_setup map clear --name currency >/dev/null
clc -c __ch3_setup map clear --name greeting >/dev/null
CURRENCY_SIZE="$(clc -c __ch3_setup map size --name currency 2>/dev/null | tr -dc '0-9')"
GREETING_SIZE="$(clc -c __ch3_setup map size --name greeting 2>/dev/null | tr -dc '0-9')"

# --- Verification tail: assert every capability this challenge depends on ---
verify() {
  local desc="$1"; shift
  if ! "$@" >/dev/null 2>&1; then
    echo "SETUP VERIFICATION FAILED: ${desc}" >&2
    echo "  command: $*" >&2
    exit 1
  fi
}

verify "Hazelcast member alive: cluster dev, 1 member, port 5701" \
  bash -c 'pgrep -f "hazelcast-5.7.0.jar" >/dev/null && grep -q "Members {size:1" ~/hazelcast/hazelcast.log && ss -tln | grep -q ":5701"'
verify "Management Center still running, connected to cluster dev, serving on 8080" \
  bash -c 'CID=$(docker ps -q --filter ancestor=hazelcast/management-center:5.11.0 --filter status=running | head -n1); [ -n "$CID" ] && docker logs "$CID" 2>&1 | grep -qi "connected to cluster dev" && curl -sf -o /dev/null -m 5 http://localhost:8080/'
verify "CLC v5.3.3 resolves via 'clc' on PATH"        bash -c 'clc version | grep -q 5.3.3'
verify "\$HOME is writable for CLC's config store"    test -w "$HOME"
verify "No leftover 'dev' CLC config from a previous attempt"      bash -c '[ ! -d "$(clc home configs dev)" ]'
verify "currency map is actually empty after clearing (not just that clear succeeded)" \
  bash -c '[ "'"$CURRENCY_SIZE"'" = "0" ]'
verify "greeting map is actually empty after clearing (not just that clear succeeded)" \
  bash -c '[ "'"$GREETING_SIZE"'" = "0" ]'

# Remove the private helper config now that verification is done -- it must never be
# left behind under a name that could be mistaken for the learner's own "dev" config.
rm -rf "$(clc home configs __ch3_setup 2>/dev/null)" 2>/dev/null || true
```

### Check (`check-workstation`)

No `set -euo pipefail` (check scripts report via `fail-message`, not silent exit). Every assertion below is a pure read (`sql SELECT`, `map get`) against already-committed cluster state — safe to click Check any number of times in a row, in any order relative to how much of the assignment is done.

```bash
#!/bin/bash

TOTAL_CHECKS=5

# Check 1: the "dev" CLC config exists and actually connects to the live cluster
# (a bad cluster.name or cluster.address would make this fail, not just "config missing")
if ! clc -c dev sql "SELECT 1;" >/dev/null 2>&1; then
  fail-message "[Check 1/${TOTAL_CHECKS}] 'clc -c dev sql \"SELECT 1;\"' failed -- no working 'dev' CLC connection was found. Run: clc config add dev cluster.name=dev cluster.address=localhost:5701"
  exit 1
fi

CURRENCY_OUT="$(clc -c dev sql -f csv "SELECT code, name FROM currency;" 2>/dev/null)"
CURRENCY_COUNT="$(echo "$CURRENCY_OUT" | grep -c . || true)"

# Check 2: the currency mapping exists and is queryable at all
if [ -z "$CURRENCY_OUT" ]; then
  fail-message "[Check 2/${TOTAL_CHECKS}] The 'currency' map isn't queryable yet (no mapping, or no rows). Run the CREATE OR REPLACE MAPPING and INSERT INTO commands from this challenge's SQL section."
  exit 1
fi

# Check 3: all five taught rows are present (not just one), matching the assignment's own
# completion marker ("shows your five currencies") -- a single spot-checked row would let
# a partial/incomplete INSERT pass.
if [ "$CURRENCY_COUNT" -lt 5 ] || ! echo "$CURRENCY_OUT" | grep -qi "USD" \
   || ! echo "$CURRENCY_OUT" | grep -qi "EUR" || ! echo "$CURRENCY_OUT" | grep -qi "GBP" \
   || ! echo "$CURRENCY_OUT" | grep -qi "JPY" || ! echo "$CURRENCY_OUT" | grep -qi "INR"; then
  fail-message "[Check 3/${TOTAL_CHECKS}] The 'currency' map doesn't contain all five rows yet (found ${CURRENCY_COUNT}/5). Re-run the INSERT INTO statement shown in this challenge -- it inserts all five currencies in one statement."
  exit 1
fi

# Check 4: greeting map holds the expected 'en' -> 'Hello' entry
if ! clc -c dev map get --name greeting en 2>/dev/null | grep -qi "Hello"; then
  fail-message "[Check 4/${TOTAL_CHECKS}] 'clc -c dev map get --name greeting en' didn't return 'Hello'. Run: clc -c dev map set --name greeting en Hello"
  exit 1
fi

# Check 5: greeting map holds the expected 'fr' -> 'Bonjour' entry
if ! clc -c dev map get --name greeting fr 2>/dev/null | grep -qi "Bonjour"; then
  fail-message "[Check 5/${TOTAL_CHECKS}] 'clc -c dev map get --name greeting fr' didn't return 'Bonjour'. Run: clc -c dev map set --name greeting fr Bonjour"
  exit 1
fi

exit 0
```

### Solve (`solve-workstation`)

```bash
#!/bin/bash
set -euxo pipefail

# Every command below is idempotent by construction: `config add` overwrites by name,
# `CREATE OR REPLACE MAPPING` replaces by name, `SINK INTO` upserts by key, and `map set`
# overwrites by key. So unlike Challenges 1-2's solve scripts (which need "if not already
# running" guards because starting a process/container twice is unsafe), nothing here
# needs a guard -- this script is safe to run whether the learner did nothing, some
# steps, or all of them manually first.
#
# One deliberate divergence from the assignment: this script uses SINK INTO instead of
# the taught INSERT INTO for the currency rows. INSERT INTO is append-only and would
# throw a duplicate-key error on a second run (e.g. the learner inserted the rows
# manually, then clicked Solve) -- SINK INTO is Hazelcast's upsert equivalent and
# reaches the identical end state check-workstation validates. The learner is never
# shown SINK INTO; it exists only so Solve is safely re-runnable.

clc config add dev cluster.name=dev cluster.address=localhost:5701 >/dev/null

clc -c dev sql "CREATE OR REPLACE MAPPING currency (__key INT, code VARCHAR, name VARCHAR) TYPE IMap OPTIONS ('keyFormat'='int','valueFormat'='json-flat');" >/dev/null

clc -c dev sql "SINK INTO currency VALUES (1,'USD','US Dollar'),(2,'EUR','Euro'),(3,'GBP','British Pound'),(4,'JPY','Japanese Yen'),(5,'INR','Indian Rupee');" >/dev/null

clc -c dev map set --name greeting en Hello >/dev/null
clc -c dev map set --name greeting fr Bonjour >/dev/null

# Bounded confirmation of the exact state check-workstation validates (all 5 rows, not just one).
for i in $(seq 1 15); do
  OUT="$(clc -c dev sql -f csv "SELECT code FROM currency;" 2>/dev/null)"
  if clc -c dev sql "SELECT 1;" >/dev/null 2>&1 \
     && echo "$OUT" | grep -qi USD && echo "$OUT" | grep -qi EUR \
     && echo "$OUT" | grep -qi GBP && echo "$OUT" | grep -qi JPY && echo "$OUT" | grep -qi INR \
     && clc -c dev map get --name greeting en 2>/dev/null | grep -qi Hello \
     && clc -c dev map get --name greeting fr 2>/dev/null | grep -qi Bonjour; then
    exit 0
  fi
  sleep 1
done

echo "ERROR: solve script ran but check-workstation's conditions were not met within 15s" >&2
exit 1
```

### Cleanup (`cleanup-workstation`)

None. The CLC config, the `currency` map's data, and the `greeting` map's data are intentionally left in place — Challenge 4 depends on this exact data being visible in Management Center. Full teardown happens at track/VM destruction, not at challenge boundaries. (Management Center itself is also left untouched, per this challenge's own scope — it was never started, stopped, or recreated here except defensively in Setup.)

## Infrastructure Changes

No changes to `config.yml`. This challenge introduces no new host, port, or service — it reuses the `workstation` VM, the existing `Terminal` tab, and the existing `Management Center` service tab exactly as Challenge 2 defined them. No new environment variables (CLC's connection parameters are supplied via `clc config add` flags at the CLI, not env vars or files). No `containers:`/port declarations change, since Docker/Management Center are not touched by this challenge beyond the defensive re-check in Setup.

## Concepts

**New — full scaffolding required:**
- **CLC as an Apache-2.0 open-source CLI client, and why this track pins v5.3.3 specifically** — new; Step 1, genuine learner-facing explanation (last source-available release; v5.3.4+ is free but binary-only) before any connection is made.
- **Named cluster connections (`clc config add`)** — new; Step 2, with an explicit callout distinguishing the CLC connection nickname from the Hazelcast cluster name (both happen to be `dev`).
- **One-shot vs. interactive shell usage** — new; Step 3, explained as two interchangeable ways to reach the same connection, with one-shot chosen for every graded command in this challenge.
- **Mapping SQL onto an `IMap` (`CREATE MAPPING`, `keyFormat`/`valueFormat`)** — new; Step 4, introduced as "why" immediately before the command that needs it.
- **`INSERT INTO` / `SELECT` against a mapped map** — new as applied to Hazelcast specifically (the SQL syntax itself is an assumed track prerequisite); Steps 5-6.
- **Direct key-value access with `clc map set`/`clc map get`, with no schema at all** — new; Step 7, explicitly contrasted with the SQL-mapped `currency` map from Steps 4-6 to show CLC supports both interaction styles against the same cluster.

**Confident reference (taught before, not re-explained):**
- **Cluster name `dev`, port 5701** — known from Challenge 1; used directly in Step 2's `cluster.address=localhost:5701` without re-explaining what the port or name mean.
- **Backgrounding processes so the terminal stays free** — taught via `nohup`/`docker run -d` in Challenges 1-2; not relevant to this challenge's own commands (CLC's one-shot commands are inherently short-lived), but the terminal being reused unmodified across all three challenges is a direct continuation of that setup, referenced in Context rather than re-taught.
- **Basic SQL (`SELECT`/`INSERT`/`WHERE`)** — assumed track prerequisite; applied against Hazelcast for the first time in Steps 5-6, not taught as SQL syntax.
- **Reading a service's state to confirm success** — taught across Challenges 1-2's log-reading steps; applied here in a lighter form (reading command output directly rather than a log file) without re-teaching the underlying skill.

## Caveats

- **This challenge re-commits, and extends, the Challenge 1/2 cross-challenge defensive-restart contract to a third dependency.** Setup here defensively re-verifies *both* the Hazelcast member (Ch1's verbatim block) *and* Management Center (Ch2's verbatim "start if not running" block) before doing anything of its own — even though this challenge's assignment never touches either directly. This is necessary specifically because Challenge 4 depends on Management Center still being connected, and this is the last chance to catch it having died before that challenge's own setup runs. Validate empirically with `instruqt track test` once Challenges 1-3 all exist, per Challenge 1's own caveat.
- **IMap data persistence across challenge/hot-start boundaries is a real risk this plan handles explicitly, not just flags.** Unlike a `nohup` process, Hazelcast map data lives inside the long-running member process itself and is fully unaffected by which challenge/terminal session is "current." On a hot-started or re-run sandbox, stale `currency`/`greeting` entries from a previous attempt could otherwise still be present when the learner arrives. Left unhandled, this would make the learner's own first `INSERT INTO currency ...` throw a duplicate-key error (`INSERT INTO` is append-only) through no fault of theirs — the single biggest new failure mode this challenge introduces relative to Challenges 1-2. Setup's clean-slate section clears both maps unconditionally via a throwaway, non-`dev`-named CLC config before the learner ever runs `clc config add dev ...`, specifically so that config-add remains genuinely the learner's first action and so their taught `INSERT INTO` command always succeeds on the first try.
- **Solve deliberately uses `SINK INTO` instead of the taught `INSERT INTO`.** This is the same underlying idempotency issue from the previous point, but from the Solve-script side: if a learner partially completes the assignment by hand (e.g., insert the rows, then click Solve) and Solve re-ran `INSERT INTO` verbatim, it would fail on the duplicate keys and abort under `set -e`. `SINK INTO` is Hazelcast's upsert-style equivalent and reaches the identical end state Check validates, so Solve needs no "did the learner already do this" guards anywhere — a deliberate, and intentionally undisclosed-to-the-learner, script-vs-assignment divergence.
- **Check is pure-read and safe to re-run without limit.** Every assertion (`SELECT`, `map get`) only reads already-committed cluster state; clicking Check repeatedly, in any order relative to assignment progress, cannot corrupt state or produce a false pass/fail based on how many times it's been run before.
- **Exact on-disk path for `$(clc home)` and its `configs/<name>` subdirectory was not independently re-confirmed for v5.3.3 specifically** (docs for adjacent CLC versions showed two different default roots for `$CLC_HOME` across different pages). The plan avoids hardcoding any path by always resolving it dynamically via `clc home configs <name>`, which should be stable across versions by design — but recommend validating empirically with `instruqt track test` once this challenge is generated, alongside Challenge 2's own flagged validation items.
- **`SELECT 1;` (a literal, table-less query) is assumed valid Hazelcast SQL syntax**, used in Check 1 purely as a connectivity/config-correctness probe unrelated to the `currency` mapping. If empirical testing shows this isn't supported, the fallback is `clc -c dev sql "SHOW MAPPINGS;"` for the same purpose (any successful response proves the `dev` connection resolves and authenticates against the right cluster name/address).
- **CLC commands are assumed to exit non-zero on SQL/connection errors** (standard CLI behavior), which every `if ! clc ...` check and the setup's clean-slate commands rely on. Recommend confirming empirically during track testing, same as the other assumptions above.
