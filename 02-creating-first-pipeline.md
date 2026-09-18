# Creating Your First Jenkins Pipeline

How to point a Jenkins job at the `Jenkinsfile` in this repository and run it.

Repository used in this guide:

```
https://github.com/Sammy4447/jenkin-pipeline.git
```

---

## 1. Prerequisites

| Requirement | Notes |
|---|---|
| A running Jenkins instance | See [01-installation-guide.md](01-installation-guide.md) |
| Git installed on the Jenkins machine | `sudo -u jenkins git --version` must work |
| Node.js + npm on the Jenkins machine | Needed by the `npm` stages in the `Jenkinsfile` |
| A `Jenkinsfile` in the repo root | Already present in this repo |

The **Pipeline** and **Git** plugins ship with the "Install suggested plugins" option during
Jenkins setup. If they're missing, add them via **Manage Jenkins → Plugins**.

---

## 2. Create the Job

1. **Dashboard → New Item**.
2. Enter an item name — e.g. `test`.
3. Select **Pipeline**.
4. Click **OK**.

You land on the job's configuration page.

---

## 3. Configure the Pipeline Source

Scroll down to the **Pipeline** section at the bottom of the page and set:

| Field | Value |
|---|---|
| Definition | **Pipeline script from SCM** |
| SCM | **Git** |
| Repository URL | `https://github.com/Sammy4447/jenkin-pipeline.git` |
| Credentials | *none* for a public repo |
| Branch Specifier | `*/main` |
| Script Path | `Jenkinsfile` |

Then click **Save**.

### Notes on these fields

- **Pipeline script from SCM** keeps the pipeline definition in the repo, so changes are
  version-controlled. The alternative, *Pipeline script*, pastes the Groovy directly into
  Jenkins and is easy to lose.
- **Branch Specifier** must match your default branch. This repo uses `main`, so `*/main`.
  If you get `Couldn't find any revision to build`, this field is usually why.
- **Script Path** is relative to the repo root. `Jenkinsfile` — capital J, no extension.
- For a **private** repo, add an SSH or token credential under **Manage Jenkins → Credentials**
  first, then select it in the *Credentials* dropdown.

---

## 4. Run the Build

From the job page, click **Build Now**.

The build appears in **Build History** on the left. Click the build number → **Console Output**
to watch it run.

A successful run ends with:

```
Finished: SUCCESS
```

---

## 5. What the Pipeline Does

The `Jenkinsfile` in this repo defines five stages:

| Stage | Command | Purpose |
|---|---|---|
| Checkout | `git` | Clones the repo at branch `main` |
| Install Dependencies | `npm install` | Installs packages from `package.json` |
| Build | `npm run build` | Runs the `build` script |
| Test | `npm test` | Runs the test suite |
| Deploy | `echo` | Placeholder for a real deployment step |

Each stage runs in order; the first failing stage aborts the build.

> **Heads up:** this repo currently has no `package.json`, so the Install / Build / Test stages
> will fail until you add a Node project. To see a green build first, comment out those three
> stages, or add a `package.json` with `build` and `test` scripts.

---

## 6. Useful Additions

### Build automatically on every push

In the job config, under **Build Triggers**, tick **GitHub hook trigger for GITScm polling**,
then on GitHub add a webhook:

- **Settings → Webhooks → Add webhook**
- Payload URL: `http://<JENKINS_PUBLIC_IP>:8080/github-webhook/`
- Content type: `application/json`
- Event: *Just the push event*

This requires Jenkins to be reachable from GitHub. If it isn't, use
**Poll SCM** with a schedule like `H/5 * * * *` (check every ~5 minutes) instead.

### See the stages visually

Install the **Pipeline: Stage View** or **Blue Ocean** plugin for a per-stage timeline
instead of reading raw console output.

---

## 7. Troubleshooting

**`Couldn't find any revision to build`**
- Branch Specifier doesn't match the real branch. Use `*/main`, not `*/master`.

**`hudson.plugins.git.GitException: Failed to connect`**
- Repo URL typo, or a private repo with no credentials attached.

**`No such file: Jenkinsfile` / `Unable to find Jenkinsfile`**
- Script Path is wrong, or the file isn't committed and pushed to `main`.

**`npm: command not found`**
- Node.js isn't installed for the `jenkins` user. Verify with `sudo -u jenkins npm -v`.
  Avoid `nvm` — Jenkins' non-login shell won't load it.

**`ENOENT: no such file or directory, open '/package.json'`**
- Expected for now — see the heads-up in section 5.

**Build hangs on the Test stage**
- A test watcher is waiting for input. Use `npm test -- --watchAll=false`, or set `CI=true`.

**Permission denied (publickey)**
- The private-repo credential is missing or wrong. Check the deploy key and that
  `github.com` is in `/var/lib/jenkins/.ssh/known_hosts`.
