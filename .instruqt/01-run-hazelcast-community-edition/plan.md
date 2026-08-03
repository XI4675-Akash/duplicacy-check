# Challenge 1: Run Hazelcast Community Edition — No License Required

## Context

This is the first challenge in the track. It has no prior challenge to build on — the learner arrives fresh from track-level sandbox setup. Its job is to deliver the track's core thesis in miniature: a real, functional Hazelcast cluster comes up from nothing but a JVM and a plain JAR — no license key, no account, no sign-up. Everything the learner does here (the running member, the `dev` cluster name, port 5701) becomes the foundation Challenges 2–4 build on: Challenge 2 points Management Center at this exact member, and Challenges 3–4 reuse this exact terminal tab and this exact running process to run CLC commands against this exact cluster.

## Prior State

None — this is the first challenge. The learner arrives with only track-level setup completed:
- Services running: none (no Hazelcast member, no Docker containers).
- Files present: `~/hazelcast/hazelcast-5.7.0.jar` (pre-downloaded), Java 17 on `PATH`, Docker daemon running (unused until Challenge 2), CLC v5.3.3 on `PATH` (unused until Challenge 3), `hazelcast/management-center:5.11.0` image pre-pulled (unused until Challenge 2).
- State from previous challenges: none.
- Terminology known to the learner: none — cluster, member, and licensing model are all introduced fresh in this challenge.

## Prerequisites

| Capability (functional at challenge start) | Host | Verify command | Provided by |
|--------------------------------------------|------|-----------------|-------------|
| OpenJDK 17+ resolves via `java` on `PATH` and reports version 17 | workstation | `java -version 2>&1 \| grep -q '"17'` | track setup |
| Hazelcast Community Edition 5.7.0 JAR present at `~/hazelcast/hazelcast-5.7.0.jar` | workstation | `test -f ~/hazelcast/hazelcast-5.7.0.jar` | track setup |
| `~/hazelcast` directory exists and is writable (needed for `> ~/hazelcast/hazelcast.log` redirection) | workstation | `test -w ~/hazelcast` | track setup |
| Port 5701 is free and no stale Hazelcast process is already running (guards against a hot-started or re-run sandbox leaving a member bound to the port before the learner starts theirs) | workstation | `! ss -tln \| grep -q ':5701'` | this challenge |

Note: this challenge's own setup script does not, and must not, start the Hazelcast member itself — that action is the learner's graded task. Setup only guarantees a clean slate (Java/JAR present, port free) so the learner's own `nohup java -jar ...` command is the first thing to bind port 5701.

## Assignment Outline

1. **Member vs. cluster.** One Hazelcast *member* is a single JVM process running the Hazelcast JAR. One or more members that discover each other form a *cluster*. Confirm the JAR is already in place: `ls -lh ~/hazelcast/hazelcast-5.7.0.jar`.

2. **Community Edition vs. Enterprise licensing.** Everything in this track — Community Edition, Management Center's free tier, CLC — runs against this member with zero license key and zero sign-up. No action here; this sets up why the next step matters.

3. **Start the member, backgrounded.** Introduce `nohup ... &` as the mechanism for running a long-lived process without occupying the terminal — explain that this same terminal tab is reused for `docker run` and `clc` commands in later challenges, so the member must not run in the foreground. Command:
   ```bash,run
   nohup java -jar ~/hazelcast/hazelcast-5.7.0.jar > ~/hazelcast/hazelcast.log 2>&1 &
   ```
   Result feedback: the shell immediately returns a job-control line like `[1] 12345` and the prompt is free again — this is the confirmation that the process is backgrounded, not that it has joined a cluster yet.

4. **Read the cluster view in the log.** Introduce the default cluster name and the member-count view as one paired concept: `grep -E "Cluster name|Members \{size" ~/hazelcast/hazelcast.log` should show `Cluster name: dev` (the default cluster name, assigned with zero configuration) and `Members {size:1, ver:1} [ ... ]` (confirms exactly one member — itself — is in the cluster view).

5. **Confirm startup completed.** A separate, single check for a separate concept: `grep "is STARTED" ~/hazelcast/hazelcast.log` should show a line ending `... 5701 is STARTED` — confirms the member finished booting and is ready to accept connections.

6. **Confirm the network port.** Introduce port 5701 as Hazelcast's default member port. Command: `ss -tln | grep 5701`. Result feedback: a `LISTEN` line for `5701` confirms the member is accepting connections — this is the same port Management Center will connect to in the next challenge.

7. **Completion / closing.** Reinforce the thesis: no license file was created, no email was entered, no account was signed into — this is Hazelcast Community Edition, Apache 2.0, running for free. Preview Challenge 2: "Next, you'll connect the official Management Center dashboard to this exact member and watch it appear live." Completion marker: "Once your log shows `Members {size:1, ver:1}` and `ss -tln` shows something listening on 5701, click Check."

## Tabs

| Title | Type | Target | Notes |
|-------|------|--------|-------|
| Terminal | terminal | hostname: workstation | VM host → use `shell: /bin/bash`, `workdir: /root`. This exact tab is reused unmodified by Challenges 2–4 for `docker run` and `clc` commands — no per-challenge tab changes needed. |

No editor or service tab in this challenge (per track plan: no file editing occurs, and Management Center's service tab does not appear until Challenge 2).

## Scripts

### Setup (`setup-workstation`)

```bash
#!/bin/bash
set -euxo pipefail

# Defensive: guarantee a clean slate even on a hot-started or re-run sandbox —
# clear any stale member from a previous attempt before the learner starts theirs.
pkill -f "hazelcast-5.7.0.jar" 2>/dev/null || true

# Bounded poll (not a blind sleep) for the port to actually release.
for i in $(seq 1 15); do
  ss -tln 2>/dev/null | grep -q ':5701' || break
  sleep 1
done

mkdir -p ~/hazelcast
rm -f ~/hazelcast/hazelcast.log

# --- Verification tail: assert every capability this challenge depends on ---
verify() {
  local desc="$1"; shift
  if ! "$@" >/dev/null 2>&1; then
    echo "SETUP VERIFICATION FAILED: ${desc}" >&2
    echo "  command: $*" >&2
    exit 1
  fi
}

verify "Java 17 resolves via 'java' on PATH"          bash -c 'java -version 2>&1 | grep -q "\"17"'
verify "Hazelcast 5.7.0 JAR present"                   test -f ~/hazelcast/hazelcast-5.7.0.jar
verify "~/hazelcast is writable for log redirection"   test -w ~/hazelcast
verify "port 5701 is free at challenge start"          bash -c '! ss -tln 2>/dev/null | grep -q ":5701"'
```

### Check (`check-workstation`)

No `set -euo pipefail` (check scripts must catch failures and report via `fail-message`, not exit silently). All assertions are pure reads (`pgrep`, `grep`, `ss`) — safe to re-run any number of times.

```bash
#!/bin/bash

TOTAL_CHECKS=5

# Check 1: a Java process is running the Hazelcast JAR
if ! pgrep -f "hazelcast-5.7.0.jar" >/dev/null 2>&1; then
  fail-message "[Check 1/${TOTAL_CHECKS}] No running Java process for hazelcast-5.7.0.jar was found. Start the member with: nohup java -jar ~/hazelcast/hazelcast-5.7.0.jar > ~/hazelcast/hazelcast.log 2>&1 &"
  exit 1
fi

# Check 2: the log shows no fatal startup error (e.g. two members fighting over port 5701)
if grep -qiE "already in use|BindException|Exception in thread \"main\"" ~/hazelcast/hazelcast.log 2>/dev/null; then
  fail-message "[Check 2/${TOTAL_CHECKS}] ~/hazelcast/hazelcast.log shows a startup error (often 'Address already in use'). Run: pkill -f hazelcast-5.7.0.jar, then restart the member with the nohup command from Step 2."
  exit 1
fi

# Check 3: the log confirms the member joined the default cluster "dev"
if ! grep -q "Cluster name: dev" ~/hazelcast/hazelcast.log 2>/dev/null; then
  fail-message "[Check 3/${TOTAL_CHECKS}] ~/hazelcast/hazelcast.log does not show 'Cluster name: dev' yet. Give the member a few more seconds to start, then click Check again."
  exit 1
fi

# Check 4: the log shows a single-member cluster view
if ! grep -q "Members {size:1" ~/hazelcast/hazelcast.log 2>/dev/null; then
  fail-message "[Check 4/${TOTAL_CHECKS}] ~/hazelcast/hazelcast.log does not show 'Members {size:1, ver:1}' yet. The member may still be starting -- wait a moment and click Check again."
  exit 1
fi

# Check 5: the member is listening on the default port
if ! ss -tln 2>/dev/null | grep -q ':5701'; then
  fail-message "[Check 5/${TOTAL_CHECKS}] Nothing is listening on port 5701. Confirm the process is still running with: pgrep -fa hazelcast-5.7.0.jar"
  exit 1
fi

exit 0
```

### Solve (`solve-workstation`)

```bash
#!/bin/bash
set -euxo pipefail

# Idempotent: only start the member if it is not already running
# (e.g. the learner started it manually before clicking Solve).
if ! pgrep -f "hazelcast-5.7.0.jar" >/dev/null 2>&1; then
  nohup java -jar ~/hazelcast/hazelcast-5.7.0.jar > ~/hazelcast/hazelcast.log 2>&1 &
  disown
fi

# Bounded poll for the exact state check-workstation validates (checks 3-5).
for i in $(seq 1 30); do
  if grep -q "Members {size:1" ~/hazelcast/hazelcast.log 2>/dev/null \
     && ss -tln 2>/dev/null | grep -q ':5701'; then
    exit 0
  fi
  sleep 1
done

echo "ERROR: Hazelcast member did not reach 'Members {size:1...}' / port 5701 within 30s" >&2
exit 1
```

### Cleanup (`cleanup-workstation`)

None. The running Hazelcast member is intentionally left in place — Challenges 2–4 depend on it staying up. Full teardown happens at track/VM destruction, not at challenge boundaries.

## Infrastructure Changes

No `config.yml` exists yet in this track (this is the first challenge planned). This plan establishes the baseline host configuration required by the track plan's Sandbox Requirements — nothing here is specific to Challenge 1's own needs beyond the `workstation` host itself; the ingress/SSL settings exist to serve Challenge 2's Management Center service tab, not this challenge (Challenge 1 uses only `localhost:5701`, never exposed externally).

- New host: `workstation` (VM), image `instruqt/docker-28-3`, `machine_type: n1-standard-2`, `nested_virtualization: true`, `allow_external_ingress: [http, https, high-ports]`, `provision_ssl_certificate: true` — per track plan Sandbox Requirements.
- New ports to expose: none for this challenge. Port 5701 is bound only to `localhost` and consumed entirely by processes on the same VM (Management Center via `--network host` in Challenge 2, `clc` in Challenge 3) — it is never reached from the learner's browser, so no external ingress or service tab is needed for it.
- New services: none beyond the Hazelcast member itself, which is started by the learner's own terminal command (the assignment's graded action), not by a setup script or a config.yml service definition.
- Environment variables needed: none.

Suggested `config.yml` (to be created since none exists):

```yaml
virtualmachines:
- name: workstation
  machine_type: n1-standard-2
  image: instruqt/docker-28-3
  nested_virtualization: true
  allow_external_ingress:
  - http
  - https
  - high-ports
  provision_ssl_certificate: true
```

## Concepts

All concepts in this challenge are new — full scaffolding required (no prior challenge exists to reference confidently):

- **Member vs. cluster** — one JVM process (member) vs. one or more members that have discovered each other (cluster). New; explained in Step 1 before the learner starts anything.
- **Community Edition vs. Enterprise licensing model** — CE requires no license key, no account, no sign-up; introduced in Step 2 as context, reinforced in the closing summary once the learner has seen it work.
- **Default cluster name (`dev`)** — new; explained in Step 4 exactly where the learner first sees it in the log.
- **Port 5701 (default member port)** — new; explained in Step 6 immediately before the learner checks it, and flagged as the same port Management Center will use in Challenge 2 (forward reference, not a re-explanation).
- **Backgrounding a process with `nohup ... &`** — new; explained in Step 3 immediately before the command that uses it, with the *why* (terminal reuse across Challenges 2–4) stated explicitly since that's easy to miss.
- **Reading a service's startup/join log** — new; taught as a transferable skill across Steps 4-5 via three named log-line patterns, each with its own single grep, rather than one combined grep covering three concepts at once.

## Caveats

- **Cross-challenge process persistence — resolved via a committed downstream contract (not just a flag).** The plugin's own best-practice reference (`anti-patterns/nohup-across-challenges.md`, severity: blocking) states that Instruqt's runner does not guarantee `nohup` background processes survive challenge boundaries. The track plan deliberately keeps `nohup ... &` in *this* challenge because typing that command is the graded learning objective (backgrounding a process) — it is not replaced with a systemd unit, which would remove that objective. Within this challenge alone the risk is not observable (check runs in the same session the process was started in), so no change is needed here.
  **Committed fix, to be implemented verbatim in every downstream challenge (2, 3, 4) that depends on this member being up:** each of those challenges' `setup-workstation` scripts MUST start with this defensive, idempotent restart-if-not-running block, before any of that challenge's own setup logic, and must NOT unconditionally truncate `~/hazelcast/hazelcast.log` (only clear/recreate it inside the `if` branch, i.e. only when actually restarting):
  ```bash
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
  ```
  This guarantees the precondition Challenges 2–4 need regardless of whether the runner preserved the process across the boundary, without touching the learner's graded action in Challenge 1. Highest priority for Challenge 2, since Management Center is configured with `MC_DEFAULT_CLUSTER_MEMBERS=localhost:5701` and would otherwise show zero members with no explicit error. Validate empirically with `instruqt track test` once Challenges 1–2 both exist.
