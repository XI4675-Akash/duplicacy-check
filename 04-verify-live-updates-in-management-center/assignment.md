---
slug: verify-live-updates-in-management-center
id: ez2mmny44hft
type: challenge
title: Verify Live Updates in Management Center
teaser: Drive changes from the CLI and watch them appear live in Management Center
  — closing the loop across all three free, license-free tools.
notes:
- type: text
  contents: |-
    <div style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; max-width: 680px; color: #E6E8EB; background: #0B0D12; border: 1px solid #24272F; border-radius: 14px; padding: 40px;">

      <div style="margin-bottom: 28px;">
        <div style="font-family: 'SF Mono', Menlo, monospace; font-size: 13px; color: #FF6D28; letter-spacing: 2px; text-transform: uppercase; margin-bottom: 10px;">Challenge 4 · Close the Loop</div>
        <div style="font-size: 28px; font-weight: 700; color: #fff; margin-bottom: 12px; line-height: 1.3;">One Cluster, Two Windows</div>
        <div style="font-size: 16px; color: #9AA2AF; line-height: 1.65;">Management Center has been idle since Challenge 2. Challenge 3 gave you a <code style="color:#FF6D28; background:#1a1512; padding:1px 5px; border-radius:4px;">currency</code> table and a <code style="color:#FF6D28; background:#1a1512; padding:1px 5px; border-radius:4px;">greeting</code> map, driven entirely from CLC. This final challenge puts both tools to work at once.</div>
      </div>

      <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 14px; margin-bottom: 28px;">
        <div style="background: #12151C; border: 1px solid #24272F; border-left: 3px solid #FF6D28; border-radius: 10px; padding: 22px;">
          <div style="color: #fff; font-weight: 600; font-size: 18px; margin-bottom: 8px;">Change it from CLC</div>
          <div style="font-family: 'SF Mono', Menlo, monospace; color: #9AA2AF; font-size: 12px; text-transform: uppercase; letter-spacing: 1.5px; margin-bottom: 14px;">Steps 1-5</div>
          <div style="font-size: 14px; line-height: 2; color: #C2C8D2;">
            · Insert a new currency row<br>
            · <code style="color:#FF6D28; background:#1a1512; padding:1px 5px; border-radius:4px;">SINK INTO</code> to update one in place<br>
            · <code style="color:#FF6D28; background:#1a1512; padding:1px 5px; border-radius:4px;">map remove</code>/<code style="color:#FF6D28; background:#1a1512; padding:1px 5px; border-radius:4px;">set</code> on `greeting`
          </div>
        </div>
        <div style="background: #12151C; border: 1px solid #24272F; border-left: 3px solid #FF6D28; border-radius: 10px; padding: 22px;">
          <div style="color: #fff; font-weight: 600; font-size: 18px; margin-bottom: 8px;">Watch it in the dashboard</div>
          <div style="font-family: 'SF Mono', Menlo, monospace; color: #9AA2AF; font-size: 12px; text-transform: uppercase; letter-spacing: 1.5px; margin-bottom: 14px;">Steps 6-7</div>
          <div style="font-size: 14px; line-height: 2; color: #C2C8D2;">
            · View the `dev` cluster's Overview<br>
            · Confirm it's Active, live, right now<br>
            · Re-confirm the data itself from CLC
          </div>
        </div>
      </div>

      <div style="background: #12151C; border: 1px solid #24272F; border-radius: 12px; padding: 20px 24px; font-size: 15px; color: #C2C8D2; text-align: center; line-height: 1.6;">
        The proof that matters: the dashboard reflects your CLI changes <strong style="color: #FF6D28;">live</strong> — no license step, anywhere in the chain.
      </div>

    </div>
tabs:
- id: xqy3bia2v0rk
  title: Workstation Terminal
  type: terminal
  hostname: workstation
  workdir: /root
- id: re3xggzxf3tp
  title: Management Center
  type: service
  hostname: workstation
  port: 8080
difficulty: intermediate
timelimit: 900
enhanced_loading: false
---

<style>
.step-header { color: #326CE5; font-weight: bold; }
</style>

You're back where Challenge 2 left off. Management Center is still open in its tab, still connected to cluster `dev`. Everything below happens in the terminal with CLC; Management Center is only ever the second half of each step — the mirror, not the tool doing the work. By the end you'll have added a currency, corrected one, deleted a greeting, and added a new one — and Management Center will show every change.

---

<h2 class="step-header">Step 1: Confirm the baseline before you touch anything</h2>

```bash,run
clc -c dev sql -f table "SELECT * FROM currency ORDER BY code;"
```

**Result:** the same five rows from Challenge 3's `SELECT` — USD, EUR, GBP, JPY, INR. Hold this picture in mind; it's the "before" you'll compare Management Center against once you switch tabs.

---

<h2 class="step-header">Step 2: Add a sixth currency with the SQL you already know</h2>

```bash,run
clc -c dev sql "INSERT INTO currency VALUES (6,'CAD','Canadian Dollar');"
```

The exact same `INSERT INTO` from Challenge 3, one new row. As before, this is append-only — running it a second time will fail on the duplicate key `6`. That's expected, not a problem; it means the row is already there.

---

<h2 class="step-header">Step 3: Correct an existing row — with a new statement, `SINK INTO`</h2>

```bash,run
clc -c dev sql "SINK INTO currency VALUES (5,'INR','Indian Rupee (updated)');"
```

`INSERT INTO` refuses to touch a key that already exists — that's exactly what made it safe to call "append-only" a moment ago. `SINK INTO` is Hazelcast's own SQL statement for the opposite job: insert-or-replace by key. Row `5` already exists from Challenge 3 — `SINK INTO` overwrites it instead of erroring. Confirm both changes at once:

```bash,run
clc -c dev sql -f table "SELECT * FROM currency ORDER BY code;"
```

**Result:** six rows now, and row 5's name has changed to `Indian Rupee (updated)`.

---

<h2 class="step-header">Step 4: Delete a key outright with `clc map remove`</h2>

```bash,run
clc -c dev map remove --name greeting fr
clc -c dev map get --name greeting fr
```

`map remove` is the direct-access sibling of the `map set`/`map get` you used in Challenge 3: no SQL, no mapping — it deletes a key directly. **Result:** the `get` now returns nothing — the key is gone.

---

<h2 class="step-header">Step 5: Add a new key with the command you already know</h2>

```bash,run
clc -c dev map set --name greeting es Hola
clc -c dev map get --name greeting es
```

**Result:** `Hola`, printed straight back — the same `map set`/`map get` pattern from Challenge 3, applied to a brand-new key.

---

<h2 class="step-header">Step 6: Switch to Management Center and take a look</h2>

Switch to the Management Center tab:

[Open the Management Center dashboard](tab-1 block=true)

Click **VIEW CLUSTER** on the `dev` card. You'll land on the cluster's Overview: **State: Active**, **Safe Members: 1/1**. That's the same live cluster you've been editing from the terminal this whole time — Management Center is watching it from the outside, in real time, with no extra setup on your part.

> [!NOTE]
> This step is not auto-graded — Check cannot inspect a rendered browser page, so there's nothing here for it to grade. The actual proof that your changes landed is the data itself, which you already confirmed with CLC in Steps 3-5, and will confirm again next.

---

<h2 class="step-header">Step 7: Confirm the changes one more time, from the terminal</h2>

Switch back to the Workstation Terminal tab and run the same query and lookups one more time:

```bash,run
clc -c dev sql -f table "SELECT * FROM currency ORDER BY code;"
clc -c dev map get --name greeting es
clc -c dev map get --name greeting fr
```

**Result:** six rows with the updated `INR` name, `Hola` for `es`, and nothing for `fr` — the same picture you already built in Steps 3-5, confirmed once more against the exact same live cluster Management Center is showing as `Active` right now.

---

<h2 class="step-header">Completion</h2>

In four challenges you took Hazelcast from a plain JAR (Challenge 1) to a member in a running cluster, to a fully monitored cluster in the official Management Center dashboard (Challenge 2), to a cluster you drove entirely from the command line with CLC — inserting, querying, updating, and deleting real data (Challenge 3, and again just now). The same dashboard you opened two challenges ago is still watching that same cluster live, right now, while every actual read and write ran through CLC.

At no point did any of it ask for a license key, a sign-up, or a trial that expires. Everything you just did is available today, for free: Hazelcast Community Edition, Management Center's free tier, and the last fully open-source release of CLC.

Once `clc -c dev sql "SELECT * FROM currency;"` shows six rows with an updated `INR` name, `clc -c dev map get --name greeting es` returns `Hola`, and `clc -c dev map get --name greeting fr` returns nothing, click **Check**.
