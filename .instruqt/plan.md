# Track: Hazelcast Community Edition — Deploy, Monitor & Query for Free

## Intent

After completing this track, learners can stand up a fully functional Hazelcast in-memory data grid, observe it live in the official Management Center dashboard, and drive it from the command line with the open-source CLC client — using **only** free, no-license, no-registration tooling. The track exists to prove to cost-conscious evaluators that Hazelcast's open-source path (Community Edition + Management Center's free tier + CLC) is a complete, real, hands-on experience, not a crippled demo.

## Products Covered

- Hazelcast Platform — Community Edition (Apache 2.0), v5.7.0
- Hazelcast Management Center (free tier, ≤3 members, no license key), v5.11.0
- Hazelcast CLC (Command Line Client), pinned to v5.3.3 (last Apache 2.0 source release — confirmed via the tagged LICENSE file at v5.3.3 and the project README, which states verbatim: "The latest open source release of CLC is v5.3.3 ... CLC v5.3.4 and up are binary only releases")

## Learning Objectives

1. **Deploy** a Hazelcast Community Edition member from the plain JAR distribution with zero configuration and no license key.
2. **Connect** the native Management Center to the running cluster and **interpret** its live cluster/member view, confirming the free tier requires no license entry for clusters of up to 3 members.
3. **Install** a version-pinned, source-available build of CLC and **use** it to run SQL statements and direct map operations (`clc sql`, `clc map set/get`) against the live cluster.
4. **Validate** that CLI-driven data changes are reflected in real time in Management Center's map and SQL browser — confirming the three free tools work together as one coherent, license-free workflow.

## Target Audience

- **Role:** Developers, architects, and platform engineers evaluating Hazelcast — particularly teams wanting to confirm what is achievable *without* Enterprise licensing before committing budget.
- **Experience level:** Beginner to intermediate with Hazelcast specifically. No prior Hazelcast knowledge assumed — cluster/member terminology, Management Center, and CLC are all taught from scratch.
- **Prerequisites (assumed, not taught):**
  - Comfortable navigating a Linux terminal (running commands, reading logs, backgrounding a process).
  - Has run `docker run` before and understands basic container concepts (image, port mapping, environment variables). Docker installation/administration is *not* taught — Docker is pre-installed in the sandbox.
  - Basic SQL literacy: can read/write `SELECT`, `INSERT`, `WHERE`.
- **Explicitly not assumed:** Java programming, prior Hazelcast experience, familiarity with distributed systems concepts (these are introduced as needed), prior exposure to CLC or Management Center.

## Sandbox Requirements

- **Infrastructure pattern:** single-vm-docker (one VM running Docker containers alongside natively-installed Java).
- **Hosts:** one VM, `workstation`.
  - Base image: `instruqt/docker-28-3` (Ubuntu + Docker CE pre-installed, avoids slow Docker install/DNS setup at track start).
  - `machine_type: n1-standard-2` — sufficient for one Hazelcast JVM member + one Management Center container + the Docker daemon; no heavy workload.
  - `nested_virtualization: true` — required for the Docker daemon to start.
  - `allow_external_ingress: [http, https, high-ports]` — required for the Management Center service tab (port 8080) to be reachable.
  - `provision_ssl_certificate: true` — avoids certificate warnings on the Management Center service tab.
- **Cloud accounts:** none. The entire track runs on a single self-contained VM with no external cloud dependency.
- **Estimated startup time:** ~2 minutes for VM boot + Docker daemon readiness, plus track-level setup time for downloads (see below) — target under 90 seconds of setup script time if the Hazelcast JAR, CLC binary, and Management Center image are fetched during track-level setup rather than per-challenge.
- **Special requirements:**
  - Outbound internet access from the VM to: GitHub (CLC v5.3.3 release asset), Docker Hub (`hazelcast/management-center:5.11.0`), and the Hazelcast download server (JAR). Flagged as an assumption below.
  - Java 17+ runtime installed on the VM (via `apt-get install openjdk-17-jre-headless`) — required to run the Hazelcast Community Edition JAR directly (chosen over running Hazelcast itself in Docker, so learners see two different free deployment shapes: plain JAR for Hazelcast, container for Management Center).
- **Process lifecycle / networking (resolved after track-plan review):**
  - The Hazelcast member (Challenge 1) is started as a **backgrounded** process: `nohup java -jar ~/hazelcast/hazelcast-5.7.0.jar > ~/hazelcast/hazelcast.log 2>&1 &`, so the single terminal tab remains free for Docker and CLC commands in later challenges. Learners verify it's alive via the log file and/or `jps`/`ss -tlnp`, not by watching a foreground process.
  - The Management Center container (Challenge 2) is run **detached** (`docker run -d ...`) and with **`--network host`**, so `MC_DEFAULT_CLUSTER_MEMBERS=localhost:5701` (or `127.0.0.1:5701`) correctly reaches the host-native Hazelcast member — avoiding the default bridge network, under which `localhost` inside the container would not resolve to the host's member. This also means the container's own port 8080 is exposed directly on the VM's network namespace (no explicit `-p` mapping needed alongside `--network host`).

## Track-Level Prerequisites

| Capability (functional after track setup) | Host | Verify command |
|--------------------------------------------|------|-----------------|
| Docker daemon operational (needed for the Management Center container) | workstation | `docker info` |
| OpenJDK 17+ installed (needed to run the Hazelcast Community Edition JAR) | workstation | `java -version 2>&1 \| grep -q '"17'` |
| Hazelcast Community Edition 5.7.0 JAR downloaded | workstation | `test -f ~/hazelcast/hazelcast-5.7.0.jar` |
| Hazelcast Management Center 5.11.0 image pre-pulled | workstation | `docker image inspect hazelcast/management-center:5.11.0` |
| CLC v5.3.3 binary installed and on PATH | workstation | `clc version \| grep -q 5.3.3` |

Pre-fetching all four artifacts once, at track-level setup, keeps every individual challenge's setup script fast (start a JVM, run a container, write a config file) rather than re-downloading anything mid-track.

## Track Configuration

- **Duration:** ~38-40 min
- **Difficulty:** Basic → Intermediate (progressive)
- **Pausable:** No (default — not requested)
- **Show timer:** Yes (default)
- **Hot start:** Recommended — track-level setup does real downloads (JAR, Docker image, CLC binary); a pre-warmed pool masks that latency for learners.

## Challenge Roadmap

### Challenge 1: Run Hazelcast Community Edition — No License Required

| Field | Value |
|-------|-------|
| **Slug** | 01-run-hazelcast-community-edition |
| **Goal** | Learner starts a single Hazelcast Community Edition member from the pre-downloaded JAR, backgrounded so the terminal stays free for later challenges (`nohup java -jar hazelcast-5.7.0.jar > hazelcast.log 2>&1 &`), and confirms from the log file that a member joined a cluster named `dev` and is listening on port 5701 — establishing that a fully functional Hazelcast cluster needs nothing more than a JVM and a JAR: no license key, no sign-up, no account. |
| **Concepts** | Hazelcast member vs. cluster; Community Edition vs. Enterprise licensing model; default cluster name; port 5701; backgrounding a long-running process with `nohup`/`&`; reading member startup/join logs. |
| **Time** | 8 min (timelimit: 12 min) |
| **Difficulty** | basic |

### Challenge 2: Monitor the Cluster with the Native Management Center

| Field | Value |
|-------|-------|
| **Slug** | 02-monitor-with-management-center |
| **Goal** | Learner runs the official Hazelcast Management Center as a single, detached, host-networked Docker container (`docker run -d --network host --env MC_DEFAULT_CLUSTER=dev --env MC_DEFAULT_CLUSTER_MEMBERS=localhost:5701 hazelcast/management-center:5.11.0`), pointing it at the host-native member from Challenge 1. They open the embedded dashboard (service tab) and confirm the member appears under Cluster ▸ Members with zero license prompts — proving the free, trial-less monitoring tier for clusters up to 3 members. |
| **Concepts** | Management Center as a separate monitoring process (not a cluster member); why `--network host` is needed to reach a host-native member from a container; `MC_DEFAULT_CLUSTER` / `MC_DEFAULT_CLUSTER_MEMBERS` env vars; the free-tier ≤3-member limit; reading the cluster overview and members view. |
| **Time** | 8 min (timelimit: 12 min) |
| **Difficulty** | basic |

### Challenge 3: Query and Modify Data with CLC

| Field | Value |
|-------|-------|
| **Slug** | 03-query-and-modify-with-clc |
| **Goal** | Learner first confirms the pre-installed CLC binary is pinned to v5.3.3 (`clc version`) and reads a short callout explaining *why*: v5.3.3 is the last CLC release Hazelcast published under Apache 2.0 with source available — from v5.3.4 onward CLC is free to use but binary-only, so this track deliberately pins to the fully open-source build to stay consistent with its "free and open source, no exceptions" thesis. Learner then configures CLC's connection to the running cluster (`clc config add dev cluster.name=dev cluster.address=localhost:5701`), and uses `clc sql` to `CREATE MAPPING` for a map, run `INSERT`/`SELECT` statements against it, and separately uses `clc map set`/`clc map get` for direct key-value operations — populating real data in the cluster entirely from the command line. |
| **Concepts** | CLC as an Apache-2.0 open-source CLI client (and why this track pins v5.3.3 specifically); named cluster connections (`clc config add`); one-shot (`clc sql "..."`) vs. interactive shell (`clc -c dev`) usage; mapping SQL to an `IMap`; `clc map set/get/remove`. |
| **Time** | 12 min (timelimit: 18 min) |
| **Difficulty** | intermediate |

### Challenge 4: Verify Live Updates in Management Center

| Field | Value |
|-------|-------|
| **Slug** | 04-verify-live-updates-in-management-center |
| **Goal** | With Management Center still open from Challenge 2, the learner returns to CLC and inserts, updates, and removes additional map entries (and/or runs a further SQL statement), then switches to Management Center's map browser and SQL browser to watch entry counts and values change in near real time — closing the loop across all three free tools (Hazelcast, Management Center, CLC) as a single workflow, with no license step anywhere in the chain. |
| **Concepts** | Management Center's map/SQL browser as a live observability surface; correlating CLI actions with dashboard state; end-to-end validation of a free, open-source workflow. |
| **Time** | 10 min (timelimit: 15 min) |
| **Difficulty** | intermediate |

## Assumptions Flagged for the Sandbox (no prior environment config exists)

1. **Single VM, not a container.** Docker is required (for Management Center), so per the container-vs-VM rule this must be a VM, not an Instruqt container (no Docker socket in containers). Hazelcast itself deliberately runs as a plain JAR, not in Docker, so learners see two distinct free deployment shapes back to back (JAR vs. container) — no Hazelcast-in-Docker alternative path is planned.
2. **`instruqt/docker-28-3` base image** is assumed available and used to avoid the DNS-fix dance and slow Docker install associated with bare Ubuntu images.
3. **Outbound internet access** from the VM is assumed for: downloading the Hazelcast JAR, pulling `hazelcast/management-center:5.11.0` from Docker Hub, and downloading the CLC v5.3.3 release asset from GitHub. If the delivery environment restricts egress, these three artifacts would need to be pre-baked into a custom Packer image instead.
4. **CLC release asset architecture** is assumed to be linux/amd64 (matching the Instruqt VM architecture) when downloading from the `v5.3.3` GitHub release tag — installed by fetching the tagged release asset directly (not the generic `hazelcast.com/clc/install.sh`, which always fetches latest and would silently pull past v5.3.3).
5. **Single-member cluster only.** No multi-node cluster is planned — this both matches "no cluster complexity" from the brief and keeps the track comfortably under Management Center's free 3-member ceiling.
6. **No editor tab.** All configuration in this track happens via CLI flags, env vars, and `clc config` commands — no file editing is required, so no editor tab is planned. Only a terminal tab (`workstation`) and a service tab (Management Center, port 8080) are needed.
7. **Hot start recommended but not mandatory** — flagged as a recommendation given the track's setup does real downloads; can be dropped if the delivery platform doesn't support it for this catalog entry.
