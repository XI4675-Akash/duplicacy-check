---
slug: run-hazelcast-community-edition
id: b5ga5phbxaih
type: challenge
title: Run Hazelcast Community Edition — No License Required
teaser: Start a real Hazelcast cluster with nothing but a JVM and a plain JAR — no
  license, no sign-up.
notes:
- type: text
  contents: |-
    <div style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; max-width: 680px; color: #E6E8EB; background: #0B0D12; border: 1px solid #24272F; border-radius: 14px; padding: 40px;">

      <div style="font-family: 'SF Mono', Menlo, monospace; font-size: 14px; letter-spacing: 2px; color: #FF6D28; text-transform: uppercase; margin-bottom: 14px;">Instruqt Sandbox</div>

      <div style="display: flex; align-items: baseline; justify-content: space-between; flex-wrap: wrap; gap: 10px; margin-bottom: 8px;">
        <div style="font-size: 36px; font-weight: 700; color: #fff;">Free, Open-Source In-Memory Data Grid</div>
        <div style="font-family: 'SF Mono', Menlo, monospace; font-size: 13px; color: #A8AFBA; border: 1px solid #2A2E38; border-radius: 20px; padding: 6px 14px;">4 CHALLENGES</div>
      </div>
      <div style="font-size: 18px; color: #9AA2AF; margin-bottom: 28px;">Deploy, monitor, and query Hazelcast Community Edition with the CLC CLI — zero license, zero cost</div>

      <div style="height: 2px; background: linear-gradient(to right, #FF6D28, transparent); margin-bottom: 28px;"></div>

      <div style="background: #12151C; border: 1px solid #24272F; border-radius: 12px; padding: 22px 24px; margin-bottom: 32px; font-size: 16px; color: #C2C8D2; line-height: 1.7;">
        <strong style="color: #fff;">Hazelcast Community Edition's</strong> core clustering and map engine are Apache 2.0; its SQL module ships under the source-available Hazelcast Community License. This track runs pinned, frozen versions end to end — Hazelcast 5.7.0, Management Center 5.11.0, and CLC <strong style="color: #fff;">v5.3.3</strong> (the last CLC release with published open source) — no account, signup, or license key anywhere.
      </div>

      <div style="font-family: 'SF Mono', Menlo, monospace; font-size: 13px; letter-spacing: 2px; color: #FF6D28; text-transform: uppercase; margin-bottom: 16px;">What you'll build</div>
      <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 14px; margin-bottom: 32px;">
        <div style="background: #12151C; border: 1px solid #24272F; border-left: 3px solid #FF6D28; border-radius: 10px; padding: 20px;">
          <div style="font-family: 'SF Mono', Menlo, monospace; font-size: 14px; color: #FF6D28; margin-bottom: 10px;">01</div>
          <div style="font-size: 17px; font-weight: 600; color: #fff; margin-bottom: 8px;">Run Hazelcast Community Edition</div>
          <div style="font-size: 14px; color: #9AA2AF; line-height: 1.5;">Plain JAR · no license key · cluster 'dev'</div>
        </div>
        <div style="background: #12151C; border: 1px solid #24272F; border-left: 3px solid #FF6D28; border-radius: 10px; padding: 20px;">
          <div style="font-family: 'SF Mono', Menlo, monospace; font-size: 14px; color: #FF6D28; margin-bottom: 10px;">02</div>
          <div style="font-size: 17px; font-weight: 600; color: #fff; margin-bottom: 8px;">Monitor with Management Center</div>
          <div style="font-size: 14px; color: #9AA2AF; line-height: 1.5;">Official dashboard · free tier · zero license</div>
        </div>
        <div style="background: #12151C; border: 1px solid #24272F; border-left: 3px solid #FF6D28; border-radius: 10px; padding: 20px;">
          <div style="font-family: 'SF Mono', Menlo, monospace; font-size: 14px; color: #FF6D28; margin-bottom: 10px;">03</div>
          <div style="font-size: 17px; font-weight: 600; color: #fff; margin-bottom: 8px;">Query & Modify with CLC</div>
          <div style="font-size: 14px; color: #9AA2AF; line-height: 1.5;">Open-source CLI · SQL · direct map access</div>
        </div>
        <div style="background: #12151C; border: 1px solid #24272F; border-left: 3px solid #FF6D28; border-radius: 10px; padding: 20px;">
          <div style="font-family: 'SF Mono', Menlo, monospace; font-size: 14px; color: #FF6D28; margin-bottom: 10px;">04</div>
          <div style="font-size: 17px; font-weight: 600; color: #fff; margin-bottom: 8px;">Verify Live Updates</div>
          <div style="font-size: 14px; color: #9AA2AF; line-height: 1.5;">CLI changes, watched live in the dashboard</div>
        </div>
      </div>

      <div style="background: #12151C; border: 1px solid #24272F; border-radius: 12px; padding: 20px 24px; font-size: 15px; color: #C2C8D2; text-align: center; line-height: 1.6;">
        This challenge: <strong style="color: #FF6D28;">bring the cluster itself into existence</strong> — nothing Hazelcast-related is running yet on your workstation.
      </div>

    </div>
tabs:
- id: sortby3lfsfi
  title: Workstation Terminal
  type: terminal
  hostname: workstation
  workdir: /root
difficulty: basic
timelimit: 720
enhanced_loading: null
---

<style>
.step-header { color: #326CE5; font-weight: bold; }
</style>

Hazelcast Community Edition's core clustering and map engine are licensed under Apache 2.0; its SQL module ships under the source-available Hazelcast Community License. Both are free to use for exactly what this track does — no license key, no account, no sign-up, for either piece. In this challenge you'll prove that yourself: start a single Hazelcast member and confirm it forms a working cluster.

---

<h2 class="step-header">Step 1: One member, one cluster</h2>

A single Hazelcast **member** is one JVM process running the Hazelcast JAR. One or more members that discover each other form a **cluster**. Right now nothing is running yet, but the JARs are already sitting on disk, pre-downloaded for you:

```bash,run
ls -lh ~/hazelcast/hazelcast-5.7.0.jar ~/hazelcast/hazelcast-sql-5.7.0.jar
```

**Result:** you should see two files. `hazelcast-5.7.0.jar` is the core distribution — clustering and the key-value map engine. `hazelcast-sql-5.7.0.jar` is a separate module that adds SQL support (`CREATE MAPPING`, `INSERT INTO`, `SELECT` — you'll use these with CLC in Challenge 3). Both go on the classpath together in the next step.

---

<h2 class="step-header">Step 2: Community Edition vs. Enterprise licensing</h2>

> [!NOTE]
> Everything you do in this track — Community Edition, Management Center's free tier, and CLC — runs against the member you're about to start with **zero license key and zero sign-up**. Keep that in mind as you go: nothing below will ever prompt you for a license or an account.

There's nothing to run for this step — it's context for why the next step matters.

---

<h2 class="step-header">Step 3: Start the member, backgrounded</h2>

This same terminal tab is reused in every later challenge to run `docker` and `clc` commands, so the Hazelcast member can't run in the foreground and tie it up. `nohup ... &` starts a process, detaches it from the terminal, and immediately hands you back the prompt.

Start the member, with both JARs on the classpath. The `-Dhz.jet.enabled=true` flag turns on Hazelcast's SQL execution engine (it's off by default when running the plain JAR directly) — you'll need it for the `CREATE MAPPING`/`INSERT INTO`/`SELECT` commands in Challenge 3:

```bash,run
nohup java -Dhz.jet.enabled=true -cp "$HOME/hazelcast/hazelcast-5.7.0.jar:$HOME/hazelcast/hazelcast-sql-5.7.0.jar" com.hazelcast.core.server.HazelcastMemberStarter > ~/hazelcast/hazelcast.log 2>&1 &
```

**Result:** the shell immediately prints a job-control line like `[1] 12345` and gives you back the prompt. That confirms the process is running in the background — it does **not** yet mean the member has joined a cluster. Startup takes a few seconds; the next steps confirm it.

---

<h2 class="step-header">Step 4: Read the cluster view in the log</h2>

Hazelcast writes its startup and cluster-membership state straight to the log file you redirected output to. Two things to look for together: the cluster this member joined, and how many members are currently in it.

```bash,run
grep -E "Cluster name|Members \{size" ~/hazelcast/hazelcast.log
```

**Result:** you should see `Cluster name: dev` — Hazelcast's default cluster name, assigned automatically with zero configuration — followed by a line like `Members {size:1, ver:1} [ ... ]`, confirming exactly one member (this one) is in the cluster.

If nothing prints yet, the member is still starting — wait a few seconds and try again.

---

<h2 class="step-header">Step 5: Confirm startup completed</h2>

A separate line in the log confirms the member finished booting and is ready to accept connections:

```bash,run
grep "is STARTED" ~/hazelcast/hazelcast.log
```

**Result:** a line ending `... 5701 is STARTED`.

---

<h2 class="step-header">Step 6: Confirm the network port</h2>

`5701` is Hazelcast's default member port — the same port Management Center will connect to in the next challenge. Confirm something is actually listening on it:

```bash,run
ss -tln | grep 5701
```

**Result:** a `LISTEN` line mentioning `5701` confirms the member is accepting connections.

---

<h2 class="step-header">Completion</h2>

No license file was created. No email was entered. No account was signed into. That's Hazelcast Community Edition — free and open to run, clustering under Apache 2.0 and SQL under the Hazelcast Community License.

Next, you'll connect the official Management Center dashboard to this exact member and watch it appear live.

Once your log shows `Members {size:1, ver:1}` and `ss -tln` shows something listening on `5701`, click **Check**.
