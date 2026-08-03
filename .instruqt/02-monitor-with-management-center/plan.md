# Challenge 2: Monitor the Cluster with the Native Management Center

## Context

Challenge 1 left a single Hazelcast Community Edition member running (backgrounded, cluster `dev`, port 5701) in the one reused terminal tab. This challenge connects the official Management Center — a separate, free-tier monitoring tool, not another cluster member — to that exact member, and has the learner visually confirm the free tier needs no license entry for a cluster this small. It is the first challenge to use Docker and the first to introduce a browser/service tab. Management Center is left running (detached container) for the rest of the track: Challenge 3 populates data via CLC without touching MC, and Challenge 4 returns to this same MC session to watch that data appear live. Nothing here is torn down at the challenge boundary.

## Prior State

- **Services running:** the Hazelcast member from Challenge 1 — cluster `dev`, one member, listening on `5701`, backgrounded via `nohup ... &`. (Not guaranteed to have survived the challenge boundary — see Prerequisites and Setup below.)
- **Files present:** `~/hazelcast/hazelcast-5.7.0.jar`, `~/hazelcast/hazelcast.log`; `hazelcast/management-center:5.11.0` pre-pulled (track setup, unused until this challenge); CLC v5.3.3 on `PATH` (track setup, unused until Challenge 3).
- **State from previous challenges:** the learner has typed `nohup java -jar ...`, read three named log patterns (`Cluster name:`, `Members {size:...}`, `is STARTED`), and checked a listening port with `ss -tln`. They know what a member and a cluster are, and that Community Edition needs no license key.
- **Terminal tab:** the same `Terminal` tab from Challenge 1, unchanged (same host, same workdir) — this challenge adds commands to it, it does not replace it.
- **Terminology already known (referenced, not re-taught):** member, cluster, cluster name `dev`, port 5701, backgrounding a process so the terminal stays free, reading a service's log to confirm state.

## Prerequisites

| Capability (functional at challenge start) | Host | Verify command | Provided by |
|--------------------------------------------|------|-----------------|-------------|
| Hazelcast member alive: cluster `dev`, one member, listening on 5701 | workstation | `pgrep -f "hazelcast-5.7.0.jar" >/dev/null && grep -q "Members {size:1" ~/hazelcast/hazelcast.log && ss -tln \| grep -q ':5701'` | 01-run-hazelcast-community-edition (defensively re-asserted by **this challenge's** setup, per the committed cross-challenge contract in Challenge 1's plan — Instruqt does not guarantee `nohup` processes survive challenge boundaries) |
| Docker daemon operational | workstation | `docker info` | track setup |
| Management Center 5.11.0 image pre-pulled locally (so the learner's `docker run` never hits the network) | workstation | `docker image inspect hazelcast/management-center:5.11.0` | track setup |
| Port 8080 free and no stale Management Center container already running (guards against a hot-started or re-run sandbox) | workstation | `! ss -tln \| grep -q ':8080'` and `[ -z "$(docker ps -q --filter ancestor=hazelcast/management-center:5.11.0)" ]` | this challenge |

Note: this challenge's setup script must not run `docker run ... hazelcast/management-center` itself — starting Management Center is the learner's graded action, exactly as starting the Hazelcast member was in Challenge 1. Setup only guarantees the member is alive, Docker works, the image is local, and port 8080 is clear.

## Assignment Outline

1. **Management Center is a separate process, not a member.** One paragraph: MC observes the cluster over the client protocol; it never joins as a member and never counts toward the free tier's member ceiling. Contrasts directly with the "member" concept from Challenge 1 (referenced, not re-explained).

2. **Why the container needs `--network host`.** Hazelcast (Challenge 1) runs natively on this VM; Management Center runs in a container. On Docker's default bridge network, `localhost` inside the container means the container itself, not the VM — `localhost:5701` would never find the host-native member. `--network host` puts the container directly on the VM's network namespace, so `localhost:5701` correctly reaches the member. Callback: this is the same kind of "make one process reachable to another" problem `nohup` solved in Challenge 1, solved here with a different mechanism because a container is involved instead of a plain background process.

3. **The two env vars that pre-wire the connection.** `MC_DEFAULT_CLUSTER=dev` (must match the cluster name the learner already read in Challenge 1's log) and `MC_DEFAULT_CLUSTER_MEMBERS=localhost:5701` (a seed address) mean Management Center lands the learner straight on the `dev` cluster's dashboard at first load — no manual "Add Cluster" wizard, no address to type in by hand.

4. **Start Management Center, detached.** Command:
   ```bash,run
   docker run -d --network host --env MC_DEFAULT_CLUSTER=dev --env MC_DEFAULT_CLUSTER_MEMBERS=localhost:5701 hazelcast/management-center:5.11.0
   ```
   Note that `-d` plays the same role `nohup ... &` played in Challenge 1 — it keeps this terminal free for CLC in Challenge 3. Result feedback: the shell prints a long container ID and returns the prompt immediately — confirmation the container is running detached, not yet that Management Center has finished booting inside it.

5. **Confirm the container is up.** `docker ps --filter ancestor=hazelcast/management-center:5.11.0` — the `STATUS` column should show `Up ...`.

6. **Read the connection in the container's log.** Applies the log-reading skill from Challenge 1 to a new source:
   ```bash
   docker logs $(docker ps -q --filter ancestor=hazelcast/management-center:5.11.0) 2>&1 | grep -i "connected to cluster dev"
   ```
   A line containing `connected to cluster dev` confirms Management Center actually reached the Challenge 1 member — not merely that the container is running.

7. **Open the dashboard and take a look — this step is not auto-graded.** Switch to the "Management Center" tab. Because of the env vars from Step 3, it should load directly into the `dev` cluster's overview — no login wall, no "add cluster" form, no license key field anywhere. Click into **Cluster ▸ Members** (or the members panel on the overview) and see your one member listed. Unlike the earlier steps, Check does not (and cannot) verify what you see in the dashboard UI — the absence of a license prompt is a fixed property of the Community Edition image, not something that could regress at runtime, so it's called out here for you to see with your own eyes rather than machine-checked.

8. **Completion / closing.** Reinforce: no license key was entered anywhere in this challenge either. Preview Challenge 3: "Leave Management Center open — next you'll drive real data into this same cluster from the command line with CLC, and Challenge 4 brings you back to this exact dashboard to watch it update live." Completion marker: "Once `docker ps` shows the Management Center container `Up` and its logs show it connected to cluster `dev`, click Check."

## Tabs

| Title | Type | Target | Notes |
|-------|------|--------|-------|
| Terminal | terminal | hostname: workstation | Same tab as Challenge 1, unchanged (`shell: /bin/bash`, `workdir: /root`). Learner runs `docker run`, `docker ps`, `docker logs` here. |
| Management Center | service | hostname: workstation, port: 8080 | New this challenge. No `path` override — `MC_DEFAULT_CLUSTER`/`MC_DEFAULT_CLUSTER_MEMBERS` already land the learner on the `dev` cluster's dashboard at `/`, and the exact in-app route to the Members view is a dynamic, cluster-scoped path that can't be hardcoded. Stays open and is reused unmodified in Challenge 4. |

No editor tab (per track plan — no file editing occurs in this track).

## Scripts

### Setup (`setup-workstation`)

```bash
#!/bin/bash
set -euxo pipefail

# --- Defensive precondition carried forward from Challenge 1 (committed contract) ---
# Instruqt does not guarantee nohup processes survive challenge boundaries; restart
# only if actually not running, and only clear the log inside this branch.
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

# --- This challenge's own setup: clean slate for Management Center ---
# Remove any stale MC container from a previous/hot-started attempt (defensive, idempotent).
STALE_CIDS="$(docker ps -aq --filter ancestor=hazelcast/management-center:5.11.0 2>/dev/null || true)"
if [ -n "$STALE_CIDS" ]; then
  docker rm -f $STALE_CIDS >/dev/null 2>&1 || true
fi

# Bounded poll for port 8080 to actually release (mirrors Challenge 1's port-free wait pattern).
for i in $(seq 1 15); do
  ss -tln 2>/dev/null | grep -q ':8080' || break
  sleep 1
done

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
verify "Docker daemon operational"                              docker info
verify "Management Center 5.11.0 image pre-pulled"              docker image inspect hazelcast/management-center:5.11.0
verify "Port 8080 free at challenge start"                       bash -c '! ss -tln 2>/dev/null | grep -q ":8080"'
verify "No stale Management Center container running"           bash -c '[ -z "$(docker ps -q --filter ancestor=hazelcast/management-center:5.11.0)" ]'
```

### Check (`check-workstation`)

No `set -euo pipefail` (check scripts must report via `fail-message`, not exit silently). All assertions are pure reads — safe to re-run.

```bash
#!/bin/bash

TOTAL_CHECKS=6

CID="$(docker ps -q --filter ancestor=hazelcast/management-center:5.11.0 --filter status=running | head -n1)"

# Check 1: a Management Center container is running
if [ -z "$CID" ]; then
  fail-message "[Check 1/${TOTAL_CHECKS}] No running container from hazelcast/management-center:5.11.0 was found. Run the command from Step 4: docker run -d --network host --env MC_DEFAULT_CLUSTER=dev --env MC_DEFAULT_CLUSTER_MEMBERS=localhost:5701 hazelcast/management-center:5.11.0"
  exit 1
fi

# Check 2: it is using host networking
NETMODE="$(docker inspect --format '{{.HostConfig.NetworkMode}}' "$CID" 2>/dev/null)"
if [ "$NETMODE" != "host" ]; then
  fail-message "[Check 2/${TOTAL_CHECKS}] The Management Center container is running but not on host networking (found: '${NETMODE}'). Without --network host it cannot reach localhost:5701 on the VM. Remove it (docker rm -f ${CID}) and rerun the Step 4 command with --network host."
  exit 1
fi

# Check 3: MC_DEFAULT_CLUSTER=dev is set
if ! docker inspect --format '{{range .Config.Env}}{{println .}}{{end}}' "$CID" 2>/dev/null | grep -qx 'MC_DEFAULT_CLUSTER=dev'; then
  fail-message "[Check 3/${TOTAL_CHECKS}] The container is missing --env MC_DEFAULT_CLUSTER=dev (or it's set to something else). Recreate it with the exact env flags from Step 4."
  exit 1
fi

# Check 4: MC_DEFAULT_CLUSTER_MEMBERS=localhost:5701 is set
if ! docker inspect --format '{{range .Config.Env}}{{println .}}{{end}}' "$CID" 2>/dev/null | grep -qx 'MC_DEFAULT_CLUSTER_MEMBERS=localhost:5701'; then
  fail-message "[Check 4/${TOTAL_CHECKS}] The container is missing --env MC_DEFAULT_CLUSTER_MEMBERS=localhost:5701 (or it points elsewhere). This is what tells Management Center where the Challenge 1 Hazelcast member lives."
  exit 1
fi

# Check 5: Management Center's own log confirms it reached the cluster
if ! docker logs "$CID" 2>&1 | grep -qi "connected to cluster dev"; then
  fail-message "[Check 5/${TOTAL_CHECKS}] Management Center's logs don't show it connected to cluster 'dev' yet. It can take up to 30 seconds after the container starts — wait a moment and click Check again. If this persists, confirm the Challenge 1 member is still running: pgrep -fa hazelcast-5.7.0.jar"
  exit 1
fi

# Check 6: the web dashboard is actually serving on 8080
if ! curl -sf -o /dev/null -m 5 http://localhost:8080/; then
  fail-message "[Check 6/${TOTAL_CHECKS}] Management Center's web dashboard isn't responding on port 8080 yet. Give it a few more seconds to finish starting, then click Check again."
  exit 1
fi

exit 0
```

### Solve (`solve-workstation`)

```bash
#!/bin/bash
set -euxo pipefail

# Idempotent: only start Management Center if no running container from that image exists
# (e.g. the learner already ran the command manually before clicking Solve).
if [ -z "$(docker ps -q --filter ancestor=hazelcast/management-center:5.11.0 --filter status=running)" ]; then
  docker run -d --network host \
    --env MC_DEFAULT_CLUSTER=dev \
    --env MC_DEFAULT_CLUSTER_MEMBERS=localhost:5701 \
    hazelcast/management-center:5.11.0
fi

CID="$(docker ps -q --filter ancestor=hazelcast/management-center:5.11.0 --filter status=running | head -n1)"

# Bounded poll for the exact state check-workstation validates (checks 5-6).
for i in $(seq 1 45); do
  if docker logs "$CID" 2>&1 | grep -qi "connected to cluster dev" \
     && curl -sf -o /dev/null -m 2 http://localhost:8080/; then
    exit 0
  fi
  sleep 2
done

echo "ERROR: Management Center did not report connection to cluster 'dev' / serve port 8080 within 90s" >&2
exit 1
```

### Cleanup (`cleanup-workstation`)

None. The Management Center container is intentionally left running — Challenge 4 depends on this exact session still being connected (and Challenge 3 must not disturb it either). Full teardown happens at track/VM destruction, not at challenge boundaries.

## Infrastructure Changes

No changes to the root `config.yml`. It already carries everything this challenge needs, established in Challenge 1's plan specifically in anticipation of this challenge:

- `allow_external_ingress: [http, https, high-ports]` — port 8080 falls inside the `high-ports` range (1024-65535), so it's already open at the VM firewall level. No new ingress category is needed.
- `provision_ssl_certificate: true` — already present, so the Instruqt proxy can terminate TLS in front of the service tab without certificate warnings.
- `machine_type: n1-standard-2` — already sized for one JVM member + one Management Center container + the Docker daemon (per track plan's Sandbox Requirements); no resize needed.

**Why no `-p` mapping and no extra port declaration:** because the container runs with `--network host`, its port 8080 is exposed directly on the VM's own network interface — identical, from the Instruqt proxy's point of view, to any native process listening on 8080. Instruqt's service-tab routing for VM hosts targets `hostname:port` on the VM directly; it does not require a per-container port declaration the way container-pattern tracks do. `-p 8080:8080` would in fact be rejected/ignored by Docker when combined with `--network host` (host networking supersedes explicit publish mappings), so omitting it is correct, not an oversight.

New for this challenge (both established via the learner's own command and this challenge's own per-challenge `config.yml`, not the root sandbox `config.yml`):
- **New port in use:** 8080 (Management Center HTTP UI), bound directly to the VM by `--network host`.
- **New service:** the Management Center container itself, started by the learner (graded action) via `docker run -d --network host ...` — not pre-started by a setup script or declared as a `containers:` resource in the root `config.yml`, since it must run *inside* the `workstation` VM alongside the native Hazelcast member, not as a sibling Instruqt-managed container resource.
- **New service tab:** "Management Center" (service, `workstation`, port 8080) — added to this challenge's own `config.yml` at generation time (see Tabs section above).
- **Environment variables:** `MC_DEFAULT_CLUSTER=dev`, `MC_DEFAULT_CLUSTER_MEMBERS=localhost:5701` — passed via `--env` flags on the learner's `docker run` command itself, not baked into any config file.

## Concepts

**New — full scaffolding required:**
- **Management Center as a separate monitoring process, not a member** — new; Step 1, stated before any command runs.
- **Container networking gap + `--network host` fix** — new; Step 2, explained immediately before the command that needs it, with an explicit callback to Challenge 1's `nohup` as a parallel "make one process reachable" problem (different mechanism, same category of concern).
- **`MC_DEFAULT_CLUSTER` / `MC_DEFAULT_CLUSTER_MEMBERS` env vars** — new; Step 3, explained immediately before the command that sets them.
- **Free-tier ≤3-member ceiling, zero license entry** — new as a concrete demonstration (the concept "CE needs no license" was introduced in Ch1; here the learner sees it hold for the *monitoring* tool too); reinforced in Step 7 and the Step 8 closing.
- **Reading Management Center's own connection log line** — new specific log content ("connected to cluster dev"), but see below — the underlying skill is a confident reference.

**Confident reference (taught before, not re-explained):**
- **Backgrounding a process to keep the terminal free** — taught via `nohup ... &` in Challenge 1; Step 4 references it directly ("`-d` plays the same role") rather than re-explaining why backgrounding matters.
- **Cluster name `dev`** — already known from Challenge 1's log; Step 3 uses it directly as the expected `MC_DEFAULT_CLUSTER` value without re-defining what a cluster name is.
- **Port 5701** — already known as the Hazelcast member's port; Step 2 references it as "the port Management Center connects to," not as a new fact.
- **Reading a service's log to confirm state** — the transferable skill (not the specific line) was taught across three greps in Challenge 1; Step 6 applies the same skill to a new source without re-teaching the technique itself.
- **`docker run` basics (image, env vars, running a container)** — assumed track prerequisite ("has run `docker run` before"); this challenge introduces only the two things genuinely new to this specific track (`--network host`, `-d` in a multi-challenge-terminal context), not `docker run` itself.

## Caveats

- **Iframe embeddability of Management Center 5.11 is not independently confirmed.** Research turned up no documented `X-Frame-Options`/CSP behavior for the Management Center web UI (Vaadin/Spring-based), and no evidence it blocks framing either. Recommend validating empirically with `instruqt track test` once this challenge is generated. If the service tab renders blank: add `custom_response_headers: ["Content-Security-Policy: ", "X-Frame-Options: "]` to the tab definition (the standard, targeted fix — see `service-tabs.md`), falling back to `new_window: true` only if header stripping doesn't resolve it. Not added preemptively, since stripping headers without evidence of a real block is unnecessary and contrary to the "minimal, targeted" guidance for service tab configuration.
- **"Zero license prompts" is a learner-observed outcome, not independently machine-checked.** Check scripts can confirm the container is up, host-networked, correctly configured, connected to the cluster, and serving HTTP — but cannot easily assert the absence of a UI element inside the rendered dashboard. This is acceptable here because the free-tier, no-license behavior is a fixed property of the OSS image (no license key is ever supplied anywhere in this challenge), not a runtime state that could silently regress.
- **MC boot time after "container running" is non-trivial** (Vaadin-based UI, typically ~15-30s after `docker run -d` returns). Check Step 5/6 and the solve script's poll loop are written to tolerate this (bounded retry, not a blind sleep); Step 6's assignment text sets the same expectation for the learner ("give it a moment").
- **This challenge inherits, and re-commits, Challenge 1's `nohup`-across-challenge-boundary contract** (see Prerequisites and Setup) — validated empirically once Challenges 1 and 2 both exist, per Challenge 1's own caveat.
