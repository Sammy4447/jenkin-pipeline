# Jenkins CI/CD Pipeline — Learning Repo

A hands-on walkthrough of setting up Jenkins on AWS EC2 and running a declarative pipeline
from a `Jenkinsfile` stored in this repository.

Read the guides in order.

---

## Guides

| # | Guide | What it covers |
|---|---|---|
| 01 | [Installing Jenkins on EC2](01-installation-guide.md) | Launch the instance, security group, Java 21, install and unlock Jenkins, plus good-to-have tooling (Git, Node.js, Docker) |
| 02 | [Creating Your First Pipeline](02-creating-first-pipeline.md) | Create a Pipeline job, point it at this repo's `Jenkinsfile` via *Pipeline script from SCM*, run the build |
| 03 | [Troubleshooting](03-troubleshooting.md) | Built-In Node going offline on the Free Temp Space threshold, and other common failures |

---

## What's in this Repo

| File | Purpose |
|---|---|
| [Jenkinsfile](Jenkinsfile) | The declarative pipeline Jenkins executes |
| [01-installation-guide.md](01-installation-guide.md) | Jenkins installation on Amazon Linux EC2 |
| [02-creating-first-pipeline.md](02-creating-first-pipeline.md) | Wiring a Jenkins job to this repo |
| [03-troubleshooting.md](03-troubleshooting.md) | Fixes for problems encountered along the way |

---

## The Pipeline

[Jenkinsfile](Jenkinsfile) defines five stages, run in order — the first failure aborts the build:

| Stage | Command | Purpose |
|---|---|---|
| Checkout | `git` | Clone branch `main` of this repo |
| Install Dependencies | `npm install` | Install packages from `package.json` |
| Build | `npm run build` | Run the build script |
| Test | `npm test` | Run the test suite |
| Deploy | `echo` | Placeholder for a real deployment |

> **Current status:** this repo contains documentation only — there is no `package.json` yet,
> so the Install / Build / Test stages will fail. To get a green build, either add a Node
> project or comment out those three stages. See
> [02-creating-first-pipeline.md](02-creating-first-pipeline.md#5-what-the-pipeline-does).

---

## Quick Start

1. Follow [01-installation-guide.md](01-installation-guide.md) to get Jenkins running at
   `http://<EC2_PUBLIC_IP>:8080`.
2. Install Git and Node.js on the instance (section 11 of the same guide).
3. Follow [02-creating-first-pipeline.md](02-creating-first-pipeline.md) to create the job:
   - Definition: **Pipeline script from SCM**
   - SCM: **Git**, URL `https://github.com/Sammy4447/jenkin-pipeline.git`
   - Branch Specifier: `*/main`
   - Script Path: `Jenkinsfile`
4. **Build Now** → open **Console Output**.
5. If the build won't start, check [03-troubleshooting.md](03-troubleshooting.md).

---

## Environment

| Item | Value |
|---|---|
| Platform | AWS EC2 |
| OS | Amazon Linux 2023 |
| Instance type | `t3.small` (2 GB RAM minimum for Jenkins) |
| Java | Amazon Corretto 21 (required by Jenkins 2.500+) |
| Jenkins port | 8080 |
