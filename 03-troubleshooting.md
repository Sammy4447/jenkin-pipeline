# Jenkins Troubleshooting

Fixes for problems hit while running this repo's pipeline.
For pipeline-job setup errors (wrong branch, missing `Jenkinsfile`), see
[02-creating-first-pipeline.md](02-creating-first-pipeline.md#7-troubleshooting).

---

## Built-In Node is Offline — "Free Temp Space below threshold"

### Symptom

The build never starts. It sits in the queue with a message like:

```
Jenkins doesn't have label 'built-in'
```

or

```
'/tmp' is not writable / Disk space is below threshold
```

And under **Manage Jenkins → Nodes**, the **Built-In Node** shows as **offline**.

### Cause

Jenkins runs a **node monitor** that takes an agent offline when free space in the temp
directory (`/tmp`) drops below a threshold. The default threshold is **1 GiB**.

On a small EC2 instance, `/tmp` often has only a few hundred MB free — so Jenkins marks its
own built-in node offline even though there is plenty of usable space. Nothing is actually
broken; the threshold is simply set higher than the disk.

Check what you actually have:

```bash
df -h /tmp
```

### Fix — lower the Free Temp Space threshold

**Step 1 — Open Nodes**

Jenkins Dashboard → **Manage Jenkins → Nodes**, then click **Built-In Node**.

**Step 2 — Open Configure Monitors**

At the top of the Nodes page, click **Configure Monitors**.

**Step 3 — Find the Free Temp Space section**

It contains three settings:

- Don't mark agents temporarily offline
- Free Space Threshold
- Free Space Warning Threshold

Set them to:

| Setting | Value |
|---|---|
| Free Space Threshold | `100 MiB` |
| Free Space Warning Threshold | `200 MiB` |

This tells Jenkins: *only* take the agent offline if `/tmp` has less than 100 MB free, and
just show a warning below 200 MB.

With roughly 452 MB free in `/tmp`, the agent stays online.

**Step 4 — Save**

Click **Save** at the bottom of the page.

**Step 5 — Verify the node is back**

Return to **Manage Jenkins → Nodes**. The Built-In Node should now show:

```
Built-In Node   🟢 online
```

**Step 6 — Run the pipeline**

Dashboard → **test** → **Build Now**, then open **Build # → Console Output**:

```
[Pipeline] Start of Pipeline

[Pipeline] stage
[Pipeline] { (Build)
Building the application...

[Pipeline] stage
[Pipeline] { (Test)
Running tests...

[Pipeline] stage
[Pipeline] { (Deploy)
Deploying the application...

Pipeline completed successfully!

Finished: SUCCESS
```

### What actually changed

Nothing about the EC2 instance — the RAM, instance type and disk were **not** touched.
Only Jenkins' Free Temp Space monitoring threshold changed:

```
1 GiB   (default, too high for this instance)
   ↓
100 MiB (agent stays online)
```

That alone took the Built-In Node from **offline → online**.

### Alternative: free the space instead

Lowering the threshold silences the monitor; it doesn't create disk space. If `/tmp` is
genuinely filling up, deal with the cause too:

```bash
df -h                                  # what's full
sudo du -sh /tmp/* 2>/dev/null | sort -h | tail
sudo rm -rf /tmp/jenkins-*             # stale build temp files
```

And in long-lived jobs, clean the workspace after each build — install the
**Workspace Cleanup** plugin and add to the `Jenkinsfile`:

```groovy
post {
    always {
        cleanWs()
    }
}
```

If the root volume itself is near full, grow the EBS volume rather than dropping the
threshold further.

> Ticking **Don't mark agents temporarily offline** disables this protection entirely.
> It works, but you lose the warning when the disk really does fill up — prefer a
> realistic threshold.
