# Challenge 4: Verify Live Updates in Management Center

## Context

This is the last challenge in the track. Challenges 1-3 built three independent pieces — a running Hazelcast member (Ch1), a connected Management Center dashboard (Ch2), and a CLC-driven `currency`/`greeting` dataset (Ch3) — but no challenge has yet shown all three working *together* in one motion. This challenge closes that loop: every action happens in CLC (the terminal tab, unchanged since Challenge 1), and every observation happens in Management Center (the service tab, unchanged since Challenge 2) — proving the CLI and the dashboard are two windows onto the same live, license-free cluster, not two disconnected demos. It is also the track's closing moment: the final step restates the whole arc (JAR → dashboard → CLI → live cross-tool verification) as one continuous, free, open-source workflow.

## Prior State

- **Services running:** the Hazelcast member (cluster `dev`, 1 member, port 5701, from Challenge 1) and the Management Center container (detached, `--network host`, connected to cluster `dev`, serving port 8080, from Challenge 2). Neither is guaranteed to have survived the challenge boundary — see Prerequisites and Setup below.
- **CLC state:** a named connection `dev` (`cluster.name=dev cluster.address=localhost:5701`, from Challenge 3); a `currency` map with a SQL mapping and 5 rows (USD, EUR, GBP, JPY, INR); a `greeting` map with 2 keys (`en`→`Hello`, `fr`→`Bonjour`). **This data is not guaranteed to have survived either — see the explicit decision below.**
- **State from previous challenges:** the learner has run one-shot `clc sql`/`clc map` commands against the `dev` connection, distinguishing SQL-mapped access from direct key-value access. They know: member, cluster, cluster name `dev`, port 5701, Management Center as a separate observer process with a free ≤3-member tier and no license prompts, `clc config add`, `CREATE MAPPING`, `INSERT INTO`/`SELECT`, and `clc map set`/`clc map get`.
- **Terminal tab:** the same `Terminal` tab from Challenges 1-3, unchanged. **Management Center tab:** the same service tab from Challenges 2-3, unchanged — this is the first challenge where the learner actually returns to look at it since Challenge 2.
- **Terminology already known (referenced, not re-taught):** everything listed above. No new vocabulary is introduced in this challenge beyond two small, sibling commands (`SINK INTO`, `clc map remove`) — this is a capstone challenge, not a new-concept-heavy one.

**Explicit decision on data persistence (do not assume Challenge 3's data survived):** Hazelcast `IMap` data lives only in the member process's memory — this track configures no persistence (no Hot Restart, no map store) anywhere. If the defensive Hazelcast-restart block below ever actually fires (i.e., the member process was not found running), every row written in Challenge 3 is gone outright, regardless of whether CLC's on-disk `dev` config file survived. Rather than try to detect "did the member just get restarted" and branch on it, this challenge's setup unconditionally re-establishes the exact Challenge 3 baseline (5 currency rows, 2 greeting keys) via idempotent commands before verifying it — safe whether Challenge 3's data fully survived, partially survived, or was wiped outright, and it also resets any stray partial-attempt state (e.g., a leftover `CAD` row or `es` key from an earlier run of *this* challenge) back to the correct starting point. See Setup below.

## Prerequisites

| Capability (functional at challenge start) | Host | Verify command | Provided by |
|--------------------------------------------|------|-----------------|-------------|
| Hazelcast member alive: cluster `dev`, one member, listening on 5701 | workstation | `pgrep -f "hazelcast-5.7.0.jar" >/dev/null && grep -q "Members {size:1" ~/hazelcast/hazelcast.log && ss -tln \| grep -q ':5701'` | 01-run-hazelcast-community-edition (defensively re-asserted by **this challenge's** setup, per the committed cross-challenge contract in Challenge 1's plan) |
| Management Center still running, connected to cluster `dev`, serving on 8080 | workstation | `CID=$(docker ps -q --filter ancestor=hazelcast/management-center:5.11.0 --filter status=running \| head -n1); [ -n "$CID" ] && docker logs "$CID" 2>&1 \| grep -qi "connected to cluster dev" && curl -sf -o /dev/null -m 5 http://localhost:8080/` | 02-monitor-with-management-center (defensively re-asserted by **this challenge's** setup) |
| CLC `dev` connection resolves and reaches the live cluster | workstation | `clc -c dev sql "SELECT 1;"` | 03-query-and-modify-with-clc (defensively re-asserted/recreated by **this challenge's** setup — CLC's on-disk config is not assumed to have survived independently of the processes above) |
| `currency` map restored to the exact 5-row Challenge 3 baseline (USD, EUR, GBP, JPY, INR) | workstation | `clc -c dev sql -f csv "SELECT code FROM currency;" \| grep -qi USD` (and EUR/GBP/JPY/INR) plus `clc -c dev map size --name currency` reports `5` | this challenge (unconditionally re-seeded — see "Explicit decision" above; **not** provided by Challenge 3, since Challenge 3's data is not assumed to survive) |
| `greeting` map restored to the exact 2-key Challenge 3 baseline (`en`→Hello, `fr`→Bonjour) | workstation | `clc -c dev map get --name greeting en` returns `Hello`, `clc -c dev map get --name greeting fr` returns `Bonjour`, `clc -c dev map size --name greeting` reports `2` | this challenge (unconditionally re-seeded, same reasoning) |

Note: this challenge's setup script establishes the *baseline* the learner will then modify (an unavoidable overlap with Challenge 3's own graded state, made necessary by the persistence decision above) — but it does not perform any of this challenge's own graded actions (adding `CAD`, updating row 5, removing `fr`, adding `es`). Those remain entirely the learner's task.

## Assignment Outline

1. **You're back where Challenge 2 left off.** Management Center is still open in its tab, still connected to cluster `dev`. Everything below happens in the terminal with CLC; Management Center is only ever the second half of each step — the mirror, not the tool doing the work. By the end you'll have added a currency, corrected one, deleted a greeting, and added a new one — and Management Center will show every change.

2. **Confirm the baseline before you touch anything.**
   ```bash,run
   clc -c dev sql -f table "SELECT * FROM currency ORDER BY code;"
   ```
   Recap of Challenge 3's `SELECT` — five rows. Hold this picture in mind; it's the "before" you'll compare Management Center against once you switch tabs.

3. **Add a sixth currency with the SQL you already know.**
   ```bash,run
   clc -c dev sql "INSERT INTO currency VALUES (6,'CAD','Canadian Dollar');"
   ```
   The exact same `INSERT INTO` from Challenge 3, one new row. As before, this is append-only — running it a second time will fail on the duplicate key `6`. That's expected, not a problem; it means the row is already there.

4. **Correct an existing row — with a new statement, `SINK INTO`.**
   ```bash,run
   clc -c dev sql "SINK INTO currency VALUES (5,'INR','Indian Rupee (updated)');"
   ```
   `INSERT INTO` refuses to touch a key that already exists — that's exactly what made it safe to call "append-only" a moment ago. `SINK INTO` is Hazelcast's own SQL statement for the opposite job: insert-or-replace by key. Row `5` already exists from Challenge 3 — `SINK INTO` overwrites it instead of erroring. Confirm both changes at once:
   ```bash,run
   clc -c dev sql -f table "SELECT * FROM currency ORDER BY code;"
   ```
   Six rows now, and row 5's name has changed.

5. **Delete a key outright with `clc map remove`.**
   ```bash,run
   clc -c dev map remove --name greeting fr
   clc -c dev map get --name greeting fr
   ```
   `map remove` is the direct-access sibling of the `map set`/`map get` you used in Challenge 3 — no SQL, no mapping, just delete a key. The `get` now returns nothing: the key is gone.

6. **Add a new key with the command you already know.**
   ```bash,run
   clc -c dev map set --name greeting es Hola
   clc -c dev map get --name greeting es
   ```

7. **Switch to Management Center and open the Maps browser — this step is not auto-graded.** Switch to the Management Center tab. Open **Data Structures ▸ Maps** and click into `currency`: six entries, including the new `CAD` row and the updated `INR` name. Click into `greeting`: two entries, `en` and `es` — `fr` is gone. Check cannot look at a rendered browser table, so it does not grade what you see here. What it verifies instead is the data underneath — the exact `currency`/`greeting` state you already confirmed yourself with CLC in Steps 4-6. If the Maps browser ever disagreed with what CLC showed you, that would point to a dashboard connection problem, not a mistake in your CLC commands — which is precisely why Check validates the data, not the screen.

8. **Switch to the SQL Browser and run the same query — also not auto-graded.** Open Management Center's **SQL Browser** and run:
   ```sql
   SELECT * FROM currency ORDER BY code;
   ```
   Same six rows, same updated name you just saw from the terminal — proof that CLC and Management Center's SQL Browser are two windows onto the identical live cluster data, not two separate copies of it. Same caveat as Step 7: this is yours to observe, not something Check inspects directly.

9. **Completion — for this challenge, and for the track.** In four challenges you took Hazelcast from a plain JAR (Challenge 1) to a member in a running cluster, to a fully monitored cluster in the official Management Center dashboard (Challenge 2), to a cluster you drove entirely from the command line with CLC — inserting, querying, updating, and deleting real data (Challenge 3, and again just now) — and finally watched those exact command-line changes appear live in the same dashboard you opened two challenges ago. At no point did any of it ask for a license key, a sign-up, or a trial that expires. Everything you just did is available today, for free: Hazelcast Community Edition, Management Center's free tier, and the last fully open-source release of CLC. Completion marker: "Once `clc -c dev sql \"SELECT * FROM currency;\"` shows six rows with an updated `INR` name, `clc -c dev map get --name greeting es` returns `Hola`, and `clc -c dev map get --name greeting fr` returns nothing, click Check."

## Tabs

| Title | Type | Target | Notes |
|-------|------|--------|-------|
| Terminal | terminal | hostname: workstation | Same tab as Challenges 1-3, unchanged (`shell: /bin/bash`, `workdir: /root`). All `clc` commands run here. |
| Management Center | service | hostname: workstation, port: 8080 | Same tab as Challenges 2-3, unchanged. This is the first challenge since Challenge 2 where the learner is actually asked to look at it (Steps 7-8). No path override, same reasoning as Challenge 2: the Maps/SQL Browser routes are dynamic, cluster-scoped paths that can't be hardcoded. |

No editor tab (per track plan — no file editing occurs anywhere in this track). No new tabs — this challenge introduces no new host, port, or service.

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

# --- Defensive precondition carried forward from Challenge 2/3 (adapted contract) ---
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

# --- This challenge's own baseline restore ---
# Decision, made explicit rather than assumed (see plan.md "Explicit decision on data
# persistence"): IMap data lives only in the member's memory, so if the block above ever
# actually had to restart the member, Challenge 3's data is gone regardless of whether
# CLC's on-disk "dev" config survived. Rather than detect "did we just restart it" and
# branch, unconditionally re-establish the exact Challenge 3 baseline via idempotent
# commands, using a private, throwaway CLC config so this never depends on "dev" already
# existing. Safe whether Ch3's data fully survived, partially survived, or was wiped
# outright -- and it also resets any stray partial-attempt state from a previous run of
# *this* challenge (e.g. a leftover CAD row or "es" key) back to the correct start point.
clc config add __ch4_setup cluster.name=dev cluster.address=localhost:5701 >/dev/null

clc -c __ch4_setup map clear --name currency >/dev/null
clc -c __ch4_setup map clear --name greeting >/dev/null

clc -c __ch4_setup sql "CREATE OR REPLACE MAPPING currency (__key INT, code VARCHAR, name VARCHAR) TYPE IMap OPTIONS ('keyFormat'='int','valueFormat'='json-flat');" >/dev/null
clc -c __ch4_setup sql "SINK INTO currency VALUES (1,'USD','US Dollar'),(2,'EUR','Euro'),(3,'GBP','British Pound'),(4,'JPY','Japanese Yen'),(5,'INR','Indian Rupee');" >/dev/null

clc -c __ch4_setup map set --name greeting en Hello >/dev/null
clc -c __ch4_setup map set --name greeting fr Bonjour >/dev/null

CURRENCY_SIZE="$(clc -c __ch4_setup map size --name currency 2>/dev/null | tr -dc '0-9')"
GREETING_SIZE="$(clc -c __ch4_setup map size --name greeting 2>/dev/null | tr -dc '0-9')"

# Ensure the learner's own named connection exists too -- idempotent (config add
# overwrites by name), safe to always run regardless of whether Ch3's on-disk config
# survived.
clc config add dev cluster.name=dev cluster.address=localhost:5701 >/dev/null

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
verify "'dev' CLC connection resolves and reaches the live cluster" \
  bash -c 'clc -c dev sql "SELECT 1;" >/dev/null 2>&1'
verify "currency map restored to the exact 5-row Challenge 3 baseline" \
  bash -c '[ "'"$CURRENCY_SIZE"'" = "5" ]'
verify "greeting map restored to the exact 2-key (en/fr) Challenge 3 baseline" \
  bash -c '[ "'"$GREETING_SIZE"'" = "2" ]'

# Remove the private helper config now that verification is done -- it must never be
# left behind under a name that could be mistaken for the learner's own "dev" config.
rm -rf "$(clc home configs __ch4_setup 2>/dev/null)" 2>/dev/null || true
```

### Check (`check-workstation`)

No `set -euo pipefail` (check scripts report via `fail-message`, not silent exit). Every assertion is a pure read against already-committed cluster state or the running Management Center container — safe to click Check any number of times.

Deliberately does **not** assert anything about what renders inside the Management Center browser UI (Steps 7-8) — that is a learner-observed outcome, exactly as Challenge 2 disclosed for its own "no license prompt" observation. What it does assert is the *underlying data* those two dashboard views are expected to show, plus that Management Center itself is actually up to render it.

```bash
#!/bin/bash

TOTAL_CHECKS=7

# Check 1: the "dev" CLC connection still works
if ! clc -c dev sql "SELECT 1;" >/dev/null 2>&1; then
  fail-message "[Check 1/${TOTAL_CHECKS}] 'clc -c dev sql \"SELECT 1;\"' failed -- the 'dev' CLC connection isn't working. Confirm it still exists: clc config add dev cluster.name=dev cluster.address=localhost:5701"
  exit 1
fi

CURRENCY_OUT="$(clc -c dev sql -f csv "SELECT code, name FROM currency;" 2>/dev/null)"

# Check 2: the four untouched Challenge 3 rows plus the new CAD row are all present
# (checked by content, not row count, since CSV output format/headers aren't assumed).
if ! echo "$CURRENCY_OUT" | grep -qi "USD" || ! echo "$CURRENCY_OUT" | grep -qi "EUR" \
   || ! echo "$CURRENCY_OUT" | grep -qi "GBP" || ! echo "$CURRENCY_OUT" | grep -qi "JPY" \
   || ! echo "$CURRENCY_OUT" | grep -qi "CAD"; then
  fail-message "[Check 2/${TOTAL_CHECKS}] The 'currency' map is missing one or more expected rows (USD/EUR/GBP/JPY/CAD). If CAD is the one missing, run: clc -c dev sql \"INSERT INTO currency VALUES (6,'CAD','Canadian Dollar');\""
  exit 1
fi

# Check 3: row 5 (INR) shows the Step 4 SINK INTO update, not its original Challenge 3 name
if ! echo "$CURRENCY_OUT" | grep -qi "Indian Rupee (updated)"; then
  fail-message "[Check 3/${TOTAL_CHECKS}] Row 5's name doesn't show the Step 4 update yet. Run: clc -c dev sql \"SINK INTO currency VALUES (5,'INR','Indian Rupee (updated)');\""
  exit 1
fi

# Check 4: 'en' (untouched by this challenge's steps) still survives -- guards against an
# over-broad fix (e.g. clearing the whole map) accidentally passing Checks 4/5 below.
if ! clc -c dev map get --name greeting en 2>/dev/null | grep -qi "Hello"; then
  fail-message "[Check 4/${TOTAL_CHECKS}] 'clc -c dev map get --name greeting en' no longer returns 'Hello'. This entry from Challenge 3 shouldn't have been touched -- restore it: clc -c dev map set --name greeting en Hello"
  exit 1
fi

# Check 5: 'fr' was removed from greeting
if clc -c dev map get --name greeting fr 2>/dev/null | grep -qi "Bonjour"; then
  fail-message "[Check 5/${TOTAL_CHECKS}] 'fr' is still in the 'greeting' map. Run: clc -c dev map remove --name greeting fr"
  exit 1
fi

# Check 6: 'es' was added to greeting
if ! clc -c dev map get --name greeting es 2>/dev/null | grep -qi "Hola"; then
  fail-message "[Check 6/${TOTAL_CHECKS}] 'clc -c dev map get --name greeting es' didn't return 'Hola'. Run: clc -c dev map set --name greeting es Hola"
  exit 1
fi

# Check 7: Management Center is still up and connected -- the underlying state Steps 7-8's
# dashboard views depend on. Check cannot assert what rendered on screen, but it can (and
# must) assert the dashboard has a live, connected cluster to render from at all.
CID="$(docker ps -q --filter ancestor=hazelcast/management-center:5.11.0 --filter status=running | head -n1)"
if [ -z "$CID" ] || ! docker logs "$CID" 2>&1 | grep -qi "connected to cluster dev" || ! curl -sf -o /dev/null -m 5 http://localhost:8080/; then
  fail-message "[Check 7/${TOTAL_CHECKS}] Management Center isn't running/connected right now, so its Maps and SQL Browser views from Steps 7-8 wouldn't show live data. Confirm the container is still up: docker ps --filter ancestor=hazelcast/management-center:5.11.0"
  exit 1
fi

exit 0
```

### Solve (`solve-workstation`)

```bash
#!/bin/bash
set -euxo pipefail

# Idempotent by construction, mirroring Challenge 3's solve script: SINK INTO (not the
# taught INSERT INTO) is used for the new CAD row too, so this script is safe to rerun
# even if the learner already ran the literal INSERT INTO themselves (which would
# otherwise error on a second run under set -e). The learner is never shown SINK INTO
# for the CAD row -- Step 3 teaches INSERT INTO deliberately, since that row is new and
# INSERT is the correct real-world tool for a genuinely new key; SINK INTO's use here is
# purely so Solve is safely re-runnable, exactly the same undisclosed divergence pattern
# Challenge 3's solve script used.
#
# `map remove` on an already-absent key is guarded with `|| true` since removing a key
# that isn't there should be a no-op outcome either way, not a script-aborting error.

clc config add dev cluster.name=dev cluster.address=localhost:5701 >/dev/null

clc -c dev sql "SINK INTO currency VALUES (6,'CAD','Canadian Dollar');" >/dev/null
clc -c dev sql "SINK INTO currency VALUES (5,'INR','Indian Rupee (updated)');" >/dev/null

clc -c dev map remove --name greeting fr >/dev/null 2>&1 || true
clc -c dev map set --name greeting es Hola >/dev/null

# Bounded confirmation of the exact state check-workstation validates.
for i in $(seq 1 15); do
  OUT="$(clc -c dev sql -f csv "SELECT code, name FROM currency;" 2>/dev/null)"
  if echo "$OUT" | grep -qi "USD" && echo "$OUT" | grep -qi "EUR" \
     && echo "$OUT" | grep -qi "GBP" && echo "$OUT" | grep -qi "JPY" \
     && echo "$OUT" | grep -qi "CAD" && echo "$OUT" | grep -qi "Indian Rupee (updated)" \
     && clc -c dev map get --name greeting en 2>/dev/null | grep -qi "Hello" \
     && ! clc -c dev map get --name greeting fr 2>/dev/null | grep -qi "Bonjour" \
     && clc -c dev map get --name greeting es 2>/dev/null | grep -qi "Hola"; then
    exit 0
  fi
  sleep 1
done

echo "ERROR: solve script ran but check-workstation's conditions were not met within 15s" >&2
exit 1
```

### Cleanup (`cleanup-workstation`)

None. This is the last challenge in the track — the Hazelcast member, Management Center container, and all CLC-populated data are left running/in place. Full teardown happens at track/VM destruction, not at challenge boundaries, consistent with Challenges 1-3.

## Infrastructure Changes

None. This challenge introduces no new host, port, service, or environment variable — it reuses the `workstation` VM, the existing `Terminal` tab, and the existing `Management Center` service tab exactly as Challenges 2-3 defined them. No changes to the root `config.yml`.

## Concepts

**New — small, low-scaffold (this is a capstone challenge; most of the learner's toolkit was already built in Challenges 1-3):**
- **`SINK INTO` as Hazelcast's SQL upsert statement** — new; Step 4, explicitly contrasted with `INSERT INTO` (which the learner just re-used in Step 3 and already knows is append-only from Challenge 3) so the *reason* a second statement exists is immediately clear.
- **`clc map remove`** — new; Step 5, explicitly introduced as the direct-access sibling of `map set`/`map get` from Challenge 3, not a new interaction paradigm.
- **Correlating CLI-driven changes with Management Center's Maps and SQL Browser as a live observability surface** — new synthesis and the actual point of this challenge; Steps 7-8. Builds directly on Challenge 2's framing of Management Center as "a separate process that observes, not a member" — this challenge is the first payoff of that framing.

**Confident reference (taught before, not re-explained):**
- **`INSERT INTO`, `SELECT`, `clc map set`/`clc map get`** — Challenge 3; reused directly in Steps 2, 3, and 6 with no re-explanation.
- **Named CLC connections (`clc config add dev ...`), one-shot command usage** — Challenge 3; used throughout without re-teaching the mechanics.
- **Member, cluster, cluster name `dev`, port 5701, backgrounding processes** — Challenges 1-2; referenced only implicitly via the already-running infrastructure, not restated.
- **Management Center as a separate, license-free monitoring process** — Challenge 2; assumed known, extended (not re-taught) into "and here's where you actually look at data, not just member counts."

## Caveats

- **Explicit decision on IMap data persistence across the challenge boundary (per the task's own instruction not to assume Challenge 3's data survived).** Documented in full under Prior State/Setup above: because Hazelcast IMap data lives only in the member process's memory with no persistence configured anywhere in this track, this challenge's setup unconditionally re-establishes the Challenge 3 baseline via idempotent commands (`map clear` + `CREATE OR REPLACE MAPPING` + `SINK INTO` + `map set`) rather than assuming it survived or trying to detect whether it did. This is a deliberate design choice, not an oversight — recommend validating empirically with `instruqt track test` once this challenge is generated, specifically by testing a hot-started run where the member process genuinely did not survive the boundary.
- **"Watching data appear live in the dashboard" is a learner-observed outcome, not independently machine-checked** — the same disclosed limitation Challenge 2 established for its own "no license prompt" observation. Check instead asserts the underlying cluster/map state Steps 7-8 are expected to show, plus that Management Center itself is up and connected (Check 6) — so a learner who sees a stale or disconnected dashboard has a machine-verified signal that something is wrong with Management Center specifically, not with their own CLC work.
- **Management Center 5.11's exact menu labels ("Data Structures ▸ Maps", "SQL Browser") are assumed from general familiarity with Hazelcast Management Center's UI structure, not independently re-confirmed for this specific version.** Same category of assumption Challenge 2 flagged for its own UI-navigation text ("Cluster ▸ Members"). Recommend confirming the exact labels empirically once this challenge is generated (e.g., via a screenshot during `instruqt track test`), and adjusting Steps 7-8's wording if the actual UI uses different labels.
- **`SINK INTO`'s upsert-by-key semantics are assumed consistent with Challenge 3's own already-approved usage of the same statement in its solve script** (Challenge 3 used `SINK INTO` internally, never shown to the learner, on the reasoning that it behaves as an idempotent insert-or-replace). This challenge is the first to teach `SINK INTO` to the learner directly rather than hide it in a solve script — the underlying behavioral assumption is carried forward, not re-verified independently here.
- **`clc map remove` on a key is assumed to exit 0 (or otherwise not abort under `set -e`) whether or not the key currently exists** — guarded defensively with `|| true` in the solve script for exactly this reason. Recommend confirming empirically during track testing, consistent with the other CLC-behavior assumptions flagged in Challenge 3's own caveats.
- **On a final "track completion" asset:** this plan deliberately does **not** add a separate track-level completion slide, video, or asset beyond Step 9's closing paragraph and Check 4 passing. Reasoning: (1) the track has no editor/notes-slide infrastructure planned anywhere per the track plan's own Assumption 6 ("no editor tab... all configuration happens via CLI"), so introducing a dedicated closing asset now would be new infrastructure with no precedent in Challenges 1-3; (2) Instruqt's own track-completion experience (a completion screen shown automatically once the last challenge's Check passes) already provides the mechanical "you're done" moment; what this plan adds on top of that is a *substantive* closing paragraph in Step 9 that explicitly re-states the track's thesis (free, open-source, no license, JAR → dashboard → CLI) one last time, which is the actual gap a bare "Check passing" would leave unaddressed. If the delivery platform's completion screen supports custom text/branding, Step 9's closing paragraph is written so it could be lifted into that surface directly — but this is presented as an optional enhancement, not a blocking requirement of this plan.
