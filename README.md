# Installing Jenkins on an Amazon Linux EC2 Instance

A step-by-step guide to running a Jenkins controller on Amazon Linux 2023 (and Amazon Linux 2) on AWS EC2.

---

## 1. Prerequisites

| Item | Recommendation |
|---|---|
| Instance type | `t3.small` or larger (Jenkins needs ~2 GB RAM; `t2.micro` will struggle) |
| AMI | Amazon Linux 2023 (or Amazon Linux 2) |
| Storage | 16 GB gp3 minimum — build workspaces fill up fast |
| Key pair | An SSH key pair you already have the `.pem` file for |
| IAM | Optional, but an instance role is cleaner than storing AWS keys in Jenkins |

---

## 2. Launch the EC2 Instance

1. EC2 Console → **Launch instance**.
2. Name it (e.g. `jenkins-controller`).
3. AMI: **Amazon Linux 2023**, architecture x86_64.
4. Instance type: **t3.small**.
5. Key pair: select or create one.
6. Network settings → **Create security group** (see below).
7. Storage: **16 GB gp3**.
8. **Launch instance**.

### Security Group Rules

| Type | Protocol | Port | Source | Purpose |
|---|---|---|---|---|
| SSH | TCP | 22 | *Your IP* | Admin access |
| Custom TCP | TCP | 8080 | *Your IP* | Jenkins web UI |
| HTTP | TCP | 80 | 0.0.0.0/0 | Only if you add a reverse proxy |

> **Do not** open port 8080 to `0.0.0.0/0`. An unlocked Jenkins on the public internet gets compromised within hours.

---

## 3. Connect to the Instance

```bash
chmod 400 my-key.pem
ssh -i my-key.pem ec2-user@<EC2_PUBLIC_IP>
```

---

## 4. Update the System

```bash
sudo dnf update -y          # Amazon Linux 2023
# sudo yum update -y        # Amazon Linux 2
```

---

## 5. Install Java

Current Jenkins releases (2.500+) require **Java 21 or 25**. Java 17 will install fine but Jenkins
will refuse to start with `older than the minimum required version (Java 21)`.

**Amazon Linux 2023:**

```bash
sudo dnf install -y java-21-amazon-corretto-devel
```

**Amazon Linux 2:**

```bash
sudo yum install -y java-21-amazon-corretto-devel
```

Verify — it must report `21.x`:

```bash
java -version
```

If an older JDK is also installed, make 21 the default:

```bash
sudo alternatives --config java
```

---

## 6. Add the Jenkins Repository

```bash
sudo wget -O /etc/yum.repos.d/jenkins.repo \
  https://pkg.jenkins.io/redhat-stable/jenkins.repo

sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
```

> If `wget` is missing: `sudo dnf install -y wget`

---

## 7. Install Jenkins

```bash
sudo dnf install -y jenkins      # AL2023
# sudo yum install -y jenkins    # AL2
```

---

## 8. Start and Enable the Service

```bash
sudo systemctl daemon-reload
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```

You should see `Active: active (running)`.

---

## 9. Unlock Jenkins

Open in your browser:

```
http://<EC2_PUBLIC_IP>:8080
```

Retrieve the initial admin password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Paste it into the **Unlock Jenkins** screen.

---

## 10. Finish Setup

1. Choose **Install suggested plugins** (takes a few minutes).
2. Create your **first admin user** — do not keep using the `admin` bootstrap account.
3. Confirm the **Jenkins URL** (`http://<EC2_PUBLIC_IP>:8080/`).
4. **Start using Jenkins**.

---

## 11. Recommended Post-Install Steps

### Install Git and build tools

```bash
sudo dnf install -y git
```

### Install Docker (if your pipelines build images)

```bash
sudo dnf install -y docker
sudo systemctl enable --now docker
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

### Give Jenkins a stable address

An EC2 public IP changes on stop/start. Attach an **Elastic IP**, or put the instance behind an ALB with a DNS record.

### Serve over HTTPS

Run Nginx or an ALB in front of Jenkins with an ACM/Let's Encrypt certificate and keep port 8080 closed to the internet.

---

## 12. Common Commands

```bash
sudo systemctl restart jenkins        # restart
sudo systemctl stop jenkins           # stop
sudo journalctl -u jenkins -f         # live logs
sudo tail -f /var/log/jenkins/jenkins.log
```

Key paths:

| Path | Contents |
|---|---|
| `/var/lib/jenkins` | JENKINS_HOME — jobs, config, plugins |
| `/var/lib/jenkins/secrets/initialAdminPassword` | Bootstrap password |
| `/usr/lib/systemd/system/jenkins.service` | systemd unit |
| `/var/log/jenkins/jenkins.log` | Application log |

---

## 13. Troubleshooting

**Page won't load at port 8080**
- Check the security group allows 8080 from your IP.
- Confirm the service is up: `sudo systemctl status jenkins`.
- Confirm it's listening: `sudo ss -tlnp | grep 8080`.

**Service fails to start**
- Read the failure first: `sudo journalctl -xeu jenkins.service --no-pager | tail -40`.
- Check Java: `java -version` — must be **21** (or 25).

**`Running with Java 17 ... older than the minimum required version (Java 21)`**

Install Corretto 21 and make it the default:

```bash
sudo dnf install -y java-21-amazon-corretto-devel
sudo alternatives --config java     # select the java-21 entry
java -version
sudo systemctl restart jenkins
```

If the system default must stay on another JDK, pin Java only for Jenkins:

```bash
sudo systemctl edit jenkins
```

```ini
[Service]
Environment="JAVA_HOME=/usr/lib/jvm/java-21-amazon-corretto"
Environment="JENKINS_JAVA_CMD=/usr/lib/jvm/java-21-amazon-corretto/bin/java"
```

```bash
sudo systemctl daemon-reload && sudo systemctl restart jenkins
```

**Out of memory / builds hang**
- Upgrade the instance type; `t2.micro` is not enough for Jenkins.

**Change the port**

```bash
sudo systemctl edit jenkins
```

Add:

```ini
[Service]
Environment="JENKINS_PORT=8081"
```

Then `sudo systemctl daemon-reload && sudo systemctl restart jenkins`.

---

## 14. Backup

`JENKINS_HOME` is everything. Back it up regularly:

```bash
sudo tar -czf /tmp/jenkins-backup-$(date +%F).tar.gz -C /var/lib jenkins
aws s3 cp /tmp/jenkins-backup-$(date +%F).tar.gz s3://my-backup-bucket/
```

EBS snapshots of the root volume work too.
