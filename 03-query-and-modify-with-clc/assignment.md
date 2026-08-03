---
slug: query-and-modify-with-clc
id: e4dqlzrnxeyy
type: challenge
title: Query and Modify Data with CLC
teaser: Drive real data into the cluster from the command line with Hazelcast's open-source
  CLI client — still zero license, zero cost.
notes:
- type: text
  contents: |-
    <div style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; max-width: 680px; color: #E6E8EB; background: #0B0D12; border: 1px solid #24272F; border-radius: 14px; padding: 40px;">

      <div style="margin-bottom: 28px;">
        <div style="font-family: 'SF Mono', Menlo, monospace; font-size: 13px; color: #FF6D28; letter-spacing: 2px; text-transform: uppercase; margin-bottom: 10px;">Challenge 3 · Drive Real Data</div>
        <div style="font-size: 28px; font-weight: 700; color: #fff; margin-bottom: 12px; line-height: 1.3;">Talk to the Cluster the Way Any App Would</div>
        <div style="font-size: 16px; color: #9AA2AF; line-height: 1.65;">So far you've only <em>observed</em> the cluster — grepping a log, then watching a dashboard. This challenge connects to it as a real client, with CLC: Hazelcast's command-line client, pinned here to its last fully open-source release.</div>
      </div>

      <div style="display: flex; flex-direction: column; gap: 14px; margin-bottom: 28px;">
        <div style="background: #12151C; border: 1px solid #24272F; border-left: 3px solid #FF6D28; border-radius: 10px; padding: 20px; display: flex; gap: 18px; align-items: flex-start;">
          <div style="font-family: 'SF Mono', Menlo, monospace; color: #FF6D28; font-size: 16px; min-width: 30px; padding-top: 2px;">01</div>
          <div>
            <div style="color: #fff; font-weight: 600; font-size: 17px; margin-bottom: 6px;">v5.3.3 — the last CLC release with published source</div>
            <div style="font-size: 14px; color: #9AA2AF; line-height: 1.5;">Newer CLC builds are free, but binary-only. This track pins <code style="color:#FF6D28; background:#1a1512; padding:2px 6px; border-radius:4px;">v5.3.3</code> — Apache 2.0, source available — to stay consistent with a genuinely open-source workflow.</div>
          </div>
        </div>
        <div style="background: #12151C; border: 1px solid #24272F; border-left: 3px solid #FF6D28; border-radius: 10px; padding: 20px; display: flex; gap: 18px; align-items: flex-start;">
          <div style="font-family: 'SF Mono', Menlo, monospace; color: #FF6D28; font-size: 16px; min-width: 30px; padding-top: 2px;">02</div>
          <div>
            <div style="color: #fff; font-weight: 600; font-size: 17px; margin-bottom: 6px;">SQL on top of a map</div>
            <div style="font-size: 14px; color: #9AA2AF; line-height: 1.5;"><code style="color:#FF6D28; background:#1a1512; padding:2px 6px; border-radius:4px;">CREATE MAPPING</code>, <code style="color:#FF6D28; background:#1a1512; padding:2px 6px; border-radius:4px;">INSERT INTO</code>, and <code style="color:#FF6D28; background:#1a1512; padding:2px 6px; border-radius:4px;">SELECT</code> put a real `currency` table into a distributed map.</div>
          </div>
        </div>
        <div style="background: #12151C; border: 1px solid #24272F; border-left: 3px solid #FF6D28; border-radius: 10px; padding: 20px; display: flex; gap: 18px; align-items: flex-start;">
          <div style="font-family: 'SF Mono', Menlo, monospace; color: #FF6D28; font-size: 16px; min-width: 30px; padding-top: 2px;">03</div>
          <div>
            <div style="color: #fff; font-weight: 600; font-size: 17px; margin-bottom: 6px;">Direct key-value access, no SQL at all</div>
            <div style="font-size: 14px; color: #9AA2AF; line-height: 1.5;"><code style="color:#FF6D28; background:#1a1512; padding:2px 6px; border-radius:4px;">clc map set</code>/<code style="color:#FF6D28; background:#1a1512; padding:2px 6px; border-radius:4px;">get</code> write and read a second map, `greeting`, with no schema needed.</div>
          </div>
        </div>
      </div>

      <div style="background: #12151C; border: 1px solid #24272F; border-radius: 12px; padding: 20px 24px; font-size: 15px; color: #C2C8D2; text-align: center; line-height: 1.6;">
        By the end: five currency rows and two greeting keys, <strong style="color: #FF6D28;">all inserted from the command line</strong>.
      </div>

    </div>
tabs:
- id: zrc4o7udrvmq
  title: Workstation Terminal
  type: terminal
  hostname: workstation
  workdir: /root
- id: lmbz9ikokjve
  title: Management Center
  type: service
  hostname: workstation
  port: 8080
difficulty: intermediate
timelimit: 1080
enhanced_loading: false
---

<style>
.step-header { color: #326CE5; font-weight: bold; }
</style>

Your Hazelcast member from Challenge 1 (cluster `dev`, port 5701) and Management Center from Challenge 2 are both still running in the background. This challenge introduces the third and last free tool in this track — CLC, Hazelcast's command-line client — and uses it to actually put data into the cluster: a SQL-mapped `currency` table and a schemaless `greeting` map, written and read entirely from the terminal. Management Center stays open in its tab, untouched — Challenge 4 is where you'll come back to it and watch this exact data appear live.

---

<h2 class="step-header">Step 1: Confirm which CLC you're running, and why it matters</h2>

```bash,run
clc version
```

**Result:** you should see `5.3.3`.

> [!NOTE]
> That's not the newest CLC release — and that's deliberate. Hazelcast still ships newer CLC builds for free, but starting with v5.3.4 those builds are binary-only: no source code is published alongside them. v5.3.3 is the last CLC release Hazelcast published under the Apache 2.0 license with source available. Since this whole track is about proving what you can do with genuinely open-source Hazelcast tooling — not just free-to-download tooling — every command below runs through this specific, fully open-source build.

---

<h2 class="step-header">Step 2: Tell CLC how to reach the cluster</h2>

```bash,run
clc config add dev cluster.name=dev cluster.address=localhost:5701
```

Two different things are both called `dev` here on purpose — don't let that trip you up. The `dev` right after `config add` is just a nickname you're choosing for this saved connection; you could call it anything (`myconn`, `local`) and nothing below would change. `cluster.name=dev` is the actual Hazelcast cluster name you read in Challenge 1's log — CLC will refuse to connect if this doesn't match the running cluster. They're spelled the same here only because it was convenient to name the connection after the cluster it points to.

---

<h2 class="step-header">Step 3: Two ways to talk to that connection</h2>

Every command below is a **one-shot** command: `clc -c dev ...` runs once against the `dev` connection and returns you straight to the prompt, so each line here is safe to copy/paste on its own. CLC also offers an **interactive shell** — run `clc -c dev` by itself, and you land in a `>` prompt where you can type statement after statement until you type `\exit`. Both reach the identical cluster the identical way; the shell is just more convenient once you're running many statements in a row. Feel free to try it, but everything graded in this challenge is shown below in one-shot form.

---

<h2 class="step-header">Step 4: Put a SQL schema on top of a map</h2>

A Hazelcast map (an `IMap`) is schemaless key-value storage by default — but `CREATE MAPPING` lets you describe columns for one, so you can then use ordinary `INSERT`/`SELECT` against it. Create a mapping for a `currency` map:

```bash,run
clc -c dev sql "CREATE OR REPLACE MAPPING currency (__key INT, code VARCHAR, name VARCHAR) TYPE IMap OPTIONS ('keyFormat'='int','valueFormat'='json-flat');"
```

`OR REPLACE` means this is always safe to re-run if you make a typo — it won't complain that the mapping already exists.

---

<h2 class="step-header">Step 5: Insert rows with ordinary SQL</h2>

```bash,run
clc -c dev sql "INSERT INTO currency VALUES (1,'USD','US Dollar'),(2,'EUR','Euro'),(3,'GBP','British Pound'),(4,'JPY','Japanese Yen'),(5,'INR','Indian Rupee');"
```

This is a real `INSERT INTO`, exactly like you'd write against any relational table — underneath, it's writing five entries into the `currency` `IMap`, keyed `1` through `5`.

> [!NOTE]
> Unlike `CREATE OR REPLACE MAPPING` above, `INSERT INTO` is append-only: if you run this exact line a second time, it will error on a duplicate key instead of silently overwriting. That's expected — it means your data is already there; there's no need to re-run it.

---

<h2 class="step-header">Step 6: Read it back with SELECT</h2>

```bash,run
clc -c dev sql -f table "SELECT * FROM currency ORDER BY code;"
```

`-f table` formats the result as readable columns. **Result:** you should see all five currencies you just inserted.

---

<h2 class="step-header">Step 7: Now skip SQL entirely — talk to a map directly</h2>

CLC can also read and write a map's entries with no schema and no SQL at all, using `clc map set`/`clc map get`. This works against *any* map, including ones that have never had a `CREATE MAPPING` run against them — this is the same kind of storage as `currency`, just accessed a different way. Populate a small `greeting` map:

```bash,run
clc -c dev map set --name greeting en Hello
clc -c dev map get --name greeting en
```

```bash,run
clc -c dev map set --name greeting fr Bonjour
clc -c dev map get --name greeting fr
```

**Result:** each `get` should print back exactly the value you just `set` for that key.

---

<h2 class="step-header">Completion</h2>

In eight commands, you've configured a CLI connection, created a SQL-queryable table backed by a distributed map, inserted and queried real rows, and read/written a second map directly by key — all without a license key, a sign-up, or a GUI.

Management Center is still open in its tab from Challenge 2. Next, you'll come back to CLC to change this data further, then switch to that dashboard and watch your changes show up live.

Once `clc -c dev sql "SELECT * FROM currency;"` shows your five currencies and `clc -c dev map get --name greeting en`/`fr` return `Hello`/`Bonjour`, click **Check**.
