---
slug: monitor-with-management-center
id: ao2p428ypuea
type: challenge
title: Monitor the Cluster with the Native Management Center
teaser: Connect the official Management Center dashboard to your running cluster —
  still zero license, zero cost.
notes:
- type: text
  contents: |-
    <div style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; max-width: 680px; color: #E6E8EB; background: #0B0D12; border: 1px solid #24272F; border-radius: 14px; padding: 40px;">

      <div style="margin-bottom: 28px;">
        <div style="font-family: 'SF Mono', Menlo, monospace; font-size: 13px; color: #FF6D28; letter-spacing: 2px; text-transform: uppercase; margin-bottom: 10px;">Challenge 2 · Watch It Live</div>
        <div style="font-size: 28px; font-weight: 700; color: #fff; margin-bottom: 12px; line-height: 1.3;">Connect the Official Dashboard</div>
        <div style="font-size: 16px; color: #9AA2AF; line-height: 1.65;">Challenge 1 proved your member was alive by grepping its log file. This challenge connects the official Management Center dashboard to that same member — over the same port 5701 — and watches it live.</div>
      </div>

      <div style="display: flex; flex-direction: column; gap: 14px; margin-bottom: 28px;">
        <div style="background: #12151C; border: 1px solid #24272F; border-left: 3px solid #FF6D28; border-radius: 10px; padding: 20px; display: flex; gap: 18px; align-items: flex-start;">
          <div style="font-family: 'SF Mono', Menlo, monospace; color: #FF6D28; font-size: 16px; min-width: 30px; padding-top: 2px;">01</div>
          <div>
            <div style="color: #fff; font-weight: 600; font-size: 17px; margin-bottom: 6px;">Pre-wire the connection</div>
            <div style="font-size: 14px; color: #9AA2AF; line-height: 1.5;"><code style="color:#FF6D28; background:#1a1512; padding:2px 6px; border-radius:4px;">MC_DEFAULT_CLUSTER</code> and <code style="color:#FF6D28; background:#1a1512; padding:2px 6px; border-radius:4px;">MC_DEFAULT_CLUSTER_MEMBERS</code> pre-register the <code style="color:#FF6D28; background:#1a1512; padding:2px 6px; border-radius:4px;">dev</code> cluster so it's already sitting on the Cluster Connections list — no "Add Cluster" wizard, no address to type by hand. You still click into it to open the dashboard.</div>
          </div>
        </div>
        <div style="background: #12151C; border: 1px solid #24272F; border-left: 3px solid #FF6D28; border-radius: 10px; padding: 20px; display: flex; gap: 18px; align-items: flex-start;">
          <div style="font-family: 'SF Mono', Menlo, monospace; color: #FF6D28; font-size: 16px; min-width: 30px; padding-top: 2px;">02</div>
          <div>
            <div style="color: #fff; font-weight: 600; font-size: 17px; margin-bottom: 6px;">Run it detached, on the host network</div>
            <div style="font-size: 14px; color: #9AA2AF; line-height: 1.5;"><code style="color:#FF6D28; background:#1a1512; padding:2px 6px; border-radius:4px;">docker run -d --network host</code> keeps your terminal free and lets the container reach the host-native member on <code style="color:#FF6D28; background:#1a1512; padding:2px 6px; border-radius:4px;">localhost:5701</code>.</div>
          </div>
        </div>
        <div style="background: #12151C; border: 1px solid #24272F; border-left: 3px solid #FF6D28; border-radius: 10px; padding: 20px; display: flex; gap: 18px; align-items: flex-start;">
          <div style="font-family: 'SF Mono', Menlo, monospace; color: #FF6D28; font-size: 16px; min-width: 30px; padding-top: 2px;">03</div>
          <div>
            <div style="color: #fff; font-weight: 600; font-size: 17px; margin-bottom: 6px;">See your member, with zero license prompt</div>
            <div style="font-size: 14px; color: #9AA2AF; line-height: 1.5;">Open the dashboard's Cluster ▸ Members view — confirming the free tier needs no license entry for a cluster this small (up to 3 members).</div>
          </div>
        </div>
      </div>

      <div style="background: #12151C; border: 1px solid #24272F; border-radius: 12px; padding: 20px 24px; font-size: 15px; color: #C2C8D2; text-align: center; line-height: 1.6;">
        By the end: Management Center showing your one member live — <strong style="color: #FF6D28;">no license key, anywhere</strong>.
      </div>

    </div>
tabs:
- id: hfp4svyh2ras
  title: Workstation Terminal
  type: terminal
  hostname: workstation
  workdir: /root
- id: wjucygesoxsl
  title: Management Center
  type: service
  hostname: workstation
  port: 8080
difficulty: basic
timelimit: 720
enhanced_loading: false
---

<style>
.step-header { color: #326CE5; font-weight: bold; }
</style>

Your Hazelcast member from Challenge 1 is still running — cluster `dev`, one member, listening on `5701`. Right now the only way to see its state is grepping a log file. In this challenge you'll connect the official Hazelcast Management Center — a free, separate monitoring tool — and watch the same cluster live in a dashboard, without ever entering a license key.

---

<h2 class="step-header">Step 1: Management Center is a monitor, not a member</h2>

Management Center is its own process that observes a cluster over the client protocol. It never joins the cluster as a member, and it never counts toward the free tier's member ceiling. The member you started in Challenge 1 stays exactly what it was — one member, cluster `dev` — Management Center just watches it from the outside.

There's nothing to run for this step — it's context for why the next step matters.

---

<h2 class="step-header">Step 2: Why the container needs `--network host`</h2>

Hazelcast itself runs natively on this VM (Challenge 1), but Management Center runs inside a Docker container. On Docker's default network, `localhost` inside a container refers to the container itself, not the VM — so `localhost:5701` would never reach the host-native member. `--network host` puts the container directly on the VM's own network namespace, so `localhost:5701` correctly finds it.

This is the same kind of problem `nohup` solved in Challenge 1 — making one process reachable to another — just solved with a different mechanism because a container is involved this time.

---

<h2 class="step-header">Step 3: Two environment variables pre-wire the connection</h2>

Two `--env` flags on the `docker run` command tell Management Center exactly which cluster to open on first load:

- `MC_DEFAULT_CLUSTER=dev` — must match the cluster name you already read in Challenge 1's log.
- `MC_DEFAULT_CLUSTER_MEMBERS=localhost:5701` — a seed address, the same port you confirmed was listening in Challenge 1.

With both set, the `dev` cluster is already sitting on the Cluster Connections list the moment Management Center loads — no "Add Cluster" wizard, no address to type in by hand. You'll still click into the `dev` card to open its dashboard, in Step 7.

---

<h2 class="step-header">Step 4: Start Management Center, detached</h2>

Run it:

```bash,run
docker run -d --network host --env MC_DEFAULT_CLUSTER=dev --env MC_DEFAULT_CLUSTER_MEMBERS=localhost:5701 hazelcast/management-center:5.11.0
```

**Result:** the shell prints a long container ID and immediately returns the prompt. The `-d` flag plays the same role `nohup ... &` played in Challenge 1 — it keeps this terminal free for CLC in the next challenge. A returned prompt confirms the container is running detached; it does **not** yet mean Management Center has finished booting inside it.

---

<h2 class="step-header">Step 5: Confirm the container is up</h2>

```bash,run
docker ps --filter ancestor=hazelcast/management-center:5.11.0
```

**Result:** one row, with the `STATUS` column reading `Up ...`.

---

<h2 class="step-header">Step 6: Read the connection in the container's log</h2>

You read Hazelcast's own log for cluster state in Challenge 1. Apply the same skill to a new source — Management Center's container log:

```bash,run
docker logs $(docker ps -q --filter ancestor=hazelcast/management-center:5.11.0) 2>&1 | grep -i "connected to cluster dev"
```

**Result:** a line containing `connected to cluster dev`. That confirms Management Center actually reached the Challenge 1 member — not merely that the container is running. If nothing prints yet, give it a few more seconds and try again; the dashboard takes a little longer to boot than the container itself.

---

<h2 class="step-header">Step 7: Open the dashboard and take a look</h2>

Switch to the Management Center tab:

[Open the Management Center dashboard](tab-1 block=true)

On first load you'll hit a **Security Configuration** screen asking which security provider to use. Pick **Dev Mode** and click **ENABLE** — it's built for exactly this case (local/eval use, no credentials), and none of the other providers (LOCAL, LDAP, SAML, etc.) apply here.

Because of the environment variables from Step 3, it then loads to a Cluster Connections list with `dev` already there — no "add cluster" form, and no license key field anywhere. Click **VIEW CLUSTER** on the `dev` card to open its dashboard, then click into **Cluster ▸ Members** (or the members panel on the overview) and confirm your one member is listed.

> [!NOTE]
> This step is not auto-graded. Check can confirm the container is running, host-networked, correctly configured, and connected to the cluster — but it can't verify what you see rendered in the dashboard UI. The absence of a license prompt is a fixed property of the Community Edition image, not something that could regress at runtime, so it's called out here for you to see with your own eyes.

---

<h2 class="step-header">Completion</h2>

No license key was entered anywhere in this challenge either — the free tier covers everything you just did, for clusters up to 3 members.

Leave Management Center open. Next, you'll drive real data into this same cluster from the command line with CLC, and the challenge after that brings you back to this exact dashboard to watch that data appear live.

Once `docker ps` shows the Management Center container `Up` and its logs show it connected to cluster `dev`, click **Check**.
