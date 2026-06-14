# From Manual Deploys at Midnight to Automated Pipelines: A Jenkins + Docker + AWS CI/CD Story

> **SEO Subtitle:** Build a production-grade CI/CD pipeline using Jenkins, Docker, and AWS EC2 — step by step, with real errors and real fixes included.
>
> **Estimated Reading Time:** 18–22 minutes
>
> **Medium Tags:** `DevOps`, `Jenkins`, `Docker`, `AWS`, `CI/CD`

---

## The Incident That Made Automation Non-Negotiable

Picture this scenario — it's 11:45 PM. A team has pushed a hotfix to production. The process? SSH into the server, manually pull the latest code, rebuild the Docker container, hope the environment variables are correct, and pray nothing breaks. It's nerve-wracking, fragile, and entirely human-dependent.

Reading about incidents like this in DevOps communities is surprisingly common. One missed step during a manual deploy — a stale `.env` file, a missed `docker pull`, an old container still running — and suddenly the application is down. Users are complaining. The on-call engineer is frantically rolling back changes at midnight with shaky hands.

This is exactly the kind of pain that CI/CD pipelines are designed to eliminate. And once you've seen it in action — where a simple `git push` triggers a complete build, test, and deployment cycle without anyone touching a server — you understand why this is considered a non-negotiable in modern engineering teams.

In this article, I'll walk through how to set up a complete, automated CI/CD pipeline from scratch using **Jenkins**, **Docker**, and **AWS EC2**. Every step is documented, every error I encountered along the way is included (with fixes), and by the end you'll have a working pipeline that deploys a full-stack application automatically on every code push.

---

## Why This Problem Matters

Manual deployments aren't just inconvenient — they're a liability. Here's what they actually cost teams:

- **Human error at scale.** The more you deploy, the more opportunities there are to make a mistake. Automation is consistent by definition.
- **Deployment fear.** Teams that deploy manually tend to deploy less often, which means larger, riskier releases. Frequent, small automated deploys are far safer.
- **No rollback story.** Without versioned images and a structured deploy process, rolling back a bad release becomes an ordeal.
- **Context switching.** Every manual deploy interrupts a developer's flow. Automation gives that time back.

A properly configured pipeline solves all of these at once. Once it's in place, the developer's only job is to push good code. Everything else is handled automatically.

---

## Project Overview

Here's what this pipeline does, end to end:

- Two AWS EC2 instances are provisioned — one for Jenkins (the CI/CD server) and one for the application
- Jenkins monitors a GitHub repository for changes
- On every `git push`, Jenkins automatically clones the latest code
- Docker images are built for both the frontend and backend services
- Images are pushed to Docker Hub with version tags for rollback support
- Jenkins SSHes into the App Server and deploys the updated containers via Docker Compose
- The application is live within minutes of a code push, with zero manual intervention

**Technologies used:**

| Component | Technology |
|---|---|
| Cloud Platform | AWS EC2 (Ubuntu 24.04 LTS) |
| CI/CD Server | Jenkins 2.504.3 |
| Containerization | Docker + Docker Compose v2 |
| Container Registry | Docker Hub |
| Version Control | GitHub |
| Deployment Method | SSH + Docker Compose |

---

## Architecture & Workflow

Understanding how the pieces connect is important before touching a single command. Here's the flow:

```
Developer Machine
       │
       │  git push
       ▼
   GitHub Repository
       │
       │  Webhook / Polling
       ▼
┌─────────────────────────┐
│     Jenkins Server      │  (EC2 Instance #1)
│  ┌─────────────────┐    │
│  │  Clone Repo     │    │
│  │  Build Frontend │    │
│  │  Build Backend  │    │
│  │  Docker Login   │    │
│  │  Push Images    │    │
│  └────────┬────────┘    │
└───────────┼─────────────┘
            │  SSH + Docker Compose Commands
            ▼
┌─────────────────────────┐
│     App Server          │  (EC2 Instance #2)
│  ┌─────────────────┐    │
│  │  docker compose │    │
│  │  pull           │    │
│  │  down           │    │
│  │  up -d          │    │
│  └────────┬────────┘    │
└───────────┼─────────────┘
            │
            ▼
     Running Application
     ├── Frontend :3000
     └── Backend  :5000
```

```
Docker Hub (Registry)
├── your-username/frontend-app:latest
├── your-username/frontend-app:1
├── your-username/frontend-app:2
└── your-username/backend-app:latest
    (versioned tags for rollback)
```

The Jenkins Server and App Server never share the same machine in this setup — that's intentional. It keeps build processes isolated from the running application, and it mirrors real-world production architectures.

---

## Prerequisites

Before diving in, make sure you have:

- An AWS account with permission to create EC2 instances
- Two EC2 instances running Ubuntu 24.04 LTS (t2.micro is fine for learning)
- A GitHub account with a repository containing your frontend and backend code (each in their own folder: `/frontend` and `/backend`)
- A Docker Hub account (free tier works)
- Basic familiarity with SSH and Linux terminal commands
- Your EC2 `.pem` key file downloaded locally

> 💡 **Note:** All IP addresses, usernames, and repository names shown in this guide are placeholders. Replace them with your actual values as you follow along.

---

## Step-by-Step Implementation

### Jenkins Server Setup

All commands in this section run on your **Jenkins EC2 instance**.

---

#### Step 1 — Download the Jenkins Package

```bash
wget https://get.jenkins.io/debian-stable/jenkins_2.504.3_all.deb
```

**Why this way?** On Ubuntu 24, the Jenkins APT repository has known key verification issues that cause `apt install jenkins` to fail or behave inconsistently. Downloading the `.deb` package directly is more reliable and gives you control over the exact version.

---

#### Step 2 — Install Jenkins

```bash
sudo dpkg -i jenkins_2.504.3_all.deb
```

> ⚠️ **Common Error:** You'll likely see this after running the above:
> ```
> dpkg: dependency problems prevent configuration of jenkins
> jenkins depends on net-tools; however: Package net-tools is not installed.
> ```
> This is completely normal — `net-tools` is a missing dependency. Don't panic. The next step handles it automatically.

---

#### Step 3 — Fix Missing Dependencies

```bash
sudo apt --fix-broken install -y
```

This command tells APT to look at what's broken and pull in whatever's missing. After this runs, you should see:

```
Setting up net-tools ...
Setting up jenkins (2.504.3) ...
Created symlink ... jenkins.service
```

---

#### Step 4 — Start and Enable Jenkins

```bash
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

`enable` ensures Jenkins starts automatically after every server reboot. `start` launches it right now.

---

#### Step 5 — Verify Jenkins is Running

```bash
sudo systemctl status jenkins
```

**Expected output:**
```
● jenkins.service - Jenkins Continuous Integration Server
   Active: active (running)
```

Press `q` to exit the status view.

> [Insert Screenshot Here: Jenkins systemctl status showing "active (running)"]

---

#### Step 6 — Open Port 8080 in AWS Security Group

Jenkins listens on port 8080. By default, AWS blocks all inbound traffic. Here's how to allow it:

Go to **AWS Console → EC2 → Your Jenkins Instance → Security → Security Group → Edit Inbound Rules → Add Rule**

| Field | Value |
|---|---|
| Type | Custom TCP |
| Port | 8080 |
| Source | Anywhere (0.0.0.0/0) |

Click **Save Rules**.

---

#### Step 7 — Access Jenkins in Browser

Open your browser and navigate to:

```
http://YOUR_JENKINS_SERVER_PUBLIC_IP:8080
```

> [Insert Screenshot Here: Jenkins initial unlock screen in browser]

---

#### Step 8 — Get the Initial Admin Password

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Copy the output and paste it into the Jenkins browser UI when prompted. This is a one-time unlock step.

---

#### Step 9 — Install Suggested Plugins and Create Admin User

Click **Install Suggested Plugins** and wait a few minutes. Once the plugins are installed, fill in your admin username, password, and email. Click **Save**.

> [Insert Screenshot Here: Jenkins plugin installation progress screen]

---

#### Step 10 — Install Docker on the Jenkins Server

```bash
sudo apt install docker.io -y
```

Jenkins needs Docker to be installed locally because it will run `docker build` and `docker push` commands as part of the pipeline.

---

#### Step 11 — Start Docker

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

---

#### Step 12 — Add Users to the Docker Group

```bash
sudo usermod -aG docker jenkins
sudo usermod -aG docker ubuntu
```

**Why this matters:** By default, running `docker` requires `sudo`. Jenkins runs as its own system user (called `jenkins`), so it won't have sudo access during pipeline execution. Adding it to the `docker` group lets Jenkins call Docker commands directly — without privilege escalation.

The `ubuntu` user is added as well, so you can also run Docker commands from your SSH session without `sudo`.

---

#### Step 13 — Restart Jenkins

```bash
sudo systemctl restart jenkins
```

Group membership changes take effect only after a restart.

---

#### Step 14 — Verify Docker

```bash
docker --version
```

**Expected output:**
```
Docker version 24.x.x, build xxxxxxx
```

---

#### Step 15 — Verify Jenkins Can Use Docker

```bash
sudo su - jenkins
docker ps
```

Switch to the Jenkins user and confirm it can communicate with the Docker daemon.

> ⚠️ **If you see:**
> ```
> permission denied while trying to connect to the Docker daemon socket
> ```
> Run this on the Jenkins server:
> ```bash
> sudo reboot
> ```
> Reconnect via SSH after a minute and test again. A full reboot is sometimes needed for group changes to propagate.

---

#### Step 16 — Install Git

```bash
sudo apt install git -y
```

Jenkins uses Git to clone your repository during the pipeline's first stage.

---

#### Step 17 — Remove the Broken Jenkins APT Repository

```bash
sudo rm -f /etc/apt/sources.list.d/jenkins.list
sudo apt update
```

Since Jenkins was installed via `.deb` (not via APT), the broken APT repository entry — if it was created — will throw errors on every `apt update`. This cleanup step removes it permanently.

---

### Docker Hub Setup

#### Step 18 — Create Docker Hub Repositories

Go to [hub.docker.com](https://hub.docker.com) and create two repositories inside your account:

- `frontend-app`
- `backend-app`

---

#### Step 19 — Generate a Docker Hub Access Token

Instead of using your Docker Hub password directly in Jenkins, generate a dedicated access token — it's more secure and can be revoked independently.

Go to **Docker Hub → Profile Icon → Account Settings → Personal Access Tokens → Generate New Token**

| Field | Value |
|---|---|
| Token Description | Jenkins |
| Access Permissions | Read, Write, Delete |

Click **Generate** and copy the token immediately. Docker Hub won't show it again.

> 💡 **The token looks like:** `dckr_pat_xxxxxxxxxxxxxxxxx` — store it somewhere safe before closing that window.

---

### SSH Key Setup (Jenkins → App Server)

The pipeline's deploy stage SSHes into the App Server to run Docker Compose commands. This requires passwordless authentication via SSH keys.

#### Step 20 — Generate SSH Keys on the Jenkins Server

```bash
ssh-keygen
```

Press Enter for all prompts. No passphrase needed.

This creates two files:

| File | Purpose |
|---|---|
| `~/.ssh/id_ed25519` | Private key — stays on Jenkins server, also added to Jenkins credentials |
| `~/.ssh/id_ed25519.pub` | Public key — goes onto the App Server |

> 💡 **Note:** Modern Ubuntu generates `ed25519` keys by default (not `id_rsa`). If you see `id_ed25519` instead of `id_rsa`, that's correct — `ed25519` is actually the more secure algorithm.

---

#### Step 21 — Copy the Public Key to the App Server

```bash
ssh-copy-id ubuntu@YOUR_APP_SERVER_PUBLIC_IP
```

> ⚠️ **Common Error:** 
> ```
> ubuntu@YOUR_APP_SERVER_IP: Permission denied (publickey)
> ```
> This happens because AWS EC2 instances launched with a `.pem` key disable password authentication by default. Here's the manual fix:
>
> **1. On the Jenkins server, display the public key:**
> ```bash
> cat ~/.ssh/id_ed25519.pub
> ```
> Copy the full output.
>
> **2. From your local machine, SSH into the App Server using your `.pem` key:**
> ```bash
> ssh -i "your-key.pem" ubuntu@YOUR_APP_SERVER_PUBLIC_IP
> ```
>
> **3. On the App Server, add the public key:**
> ```bash
> mkdir -p ~/.ssh
> nano ~/.ssh/authorized_keys
> ```
> Paste the copied public key at the bottom. Save with `CTRL+O`, `Enter`, `CTRL+X`.
>
> **4. Fix permissions:**
> ```bash
> chmod 700 ~/.ssh
> chmod 600 ~/.ssh/authorized_keys
> ```

---

#### Step 22 — Test Passwordless SSH

From the Jenkins server:

```bash
ssh ubuntu@YOUR_APP_SERVER_PUBLIC_IP
```

**Expected:** You're logged into the App Server with no password or `.pem` file required. Type `exit` to return.

> [Insert Screenshot Here: Successful passwordless SSH from Jenkins server to App Server]

---

### Store Credentials in Jenkins

Jenkins stores secrets (passwords, SSH keys) in a secure internal credentials store. Your Jenkinsfile references them by ID — the actual secret is never written in plain text in the pipeline code.

#### Step 23 — Store Docker Hub Credentials

Go to: **Manage Jenkins → Credentials → (System) → Global credentials → Add Credentials**

| Field | Value |
|---|---|
| Kind | Username with password |
| Username | your-dockerhub-username |
| Password | your-dockerhub-access-token |
| ID | `dockerhub-creds` |

> ⚠️ **The ID must be exactly `dockerhub-creds`** — this string is referenced inside the Jenkinsfile. If you change it, the pipeline will fail.

---

#### Step 24 — Store the SSH Private Key in Jenkins

First, get the private key content from the Jenkins server:

```bash
cat ~/.ssh/id_ed25519
```

Copy the complete output — including the `-----BEGIN OPENSSH PRIVATE KEY-----` and `-----END OPENSSH PRIVATE KEY-----` lines.

Go to: **Manage Jenkins → Credentials → Global → Add Credentials**

| Field | Value |
|---|---|
| Kind | SSH Username with private key |
| Username | ubuntu |
| Private Key | Enter directly → paste `id_ed25519` content |
| ID | `app-server-ssh` |

> ⚠️ **The ID must be exactly `app-server-ssh`** — referenced in the Jenkinsfile's `sshagent` block.

---

### App Server Setup

SSH into your App Server using your `.pem` key from your local machine:

```bash
ssh -i "your-key.pem" ubuntu@YOUR_APP_SERVER_PUBLIC_IP
```

#### Step 25 — Install Docker on the App Server

```bash
sudo apt update
sudo apt install docker.io docker-compose-v2 -y
```

The `docker-compose-v2` package installs the modern `docker compose` plugin (with a space, not a hyphen). More on why this matters in the troubleshooting section.

---

#### Step 26 — Start Docker

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

---

#### Step 27 — Add Ubuntu User to Docker Group

```bash
sudo usermod -aG docker ubuntu
newgrp docker
```

`newgrp docker` applies the group change to the current session without requiring a logout.

---

#### Step 28 — Verify Docker and Docker Compose

```bash
docker --version
docker compose version
```

**Expected output:**
```
Docker version 24.x.x, build xxxxxxx
Docker Compose version v2.x.x
```

---

#### Step 29 — Create the App Directory and docker-compose.yml

```bash
mkdir app
cd app
nano docker-compose.yml
```

Paste the following content (replace `your-dockerhub-username` with your actual Docker Hub username):

```yaml
services:
  frontend:
    image: your-dockerhub-username/frontend-app:latest
    container_name: frontend-container
    ports:
      - "3000:3000"
    restart: always

  backend:
    image: your-dockerhub-username/backend-app:latest
    container_name: backend-container
    ports:
      - "5000:5000"
    restart: always
```

Save: `CTRL+O` → `Enter` → `CTRL+X`

> 💡 **Note:** Do not add `version: '3'` at the top of this file. Modern Docker Compose treats this field as obsolete and will print a warning every time. The file works correctly without it.

---

#### Step 30 — Open Required Ports in the App Server Security Group

In AWS Console, add inbound rules for the App Server's security group:

| Port | Purpose |
|---|---|
| 3000 | Frontend application |
| 5000 | Backend application |
| 22 | SSH (already open if you're connected) |

---

### Writing the Jenkinsfile

The Jenkinsfile is the heart of this entire setup. It lives in the root of your GitHub repository and defines every stage of the pipeline as code.

#### Step 31 — The Complete Jenkinsfile

Create a file named `Jenkinsfile` in the root of your GitHub repository with the following content. Replace the placeholder values as noted in the comments.

```groovy
pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-creds')
        FRONTEND_IMAGE = "your-dockerhub-username/frontend-app"
        BACKEND_IMAGE  = "your-dockerhub-username/backend-app"
        APP_SERVER     = "ubuntu@YOUR_APP_SERVER_PUBLIC_IP"
    }

    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main',
                url: 'https://github.com/your-github-username/your-repo.git'
            }
        }

        stage('Build Frontend Image') {
            steps {
                sh 'docker build -t $FRONTEND_IMAGE:$BUILD_NUMBER -t $FRONTEND_IMAGE:latest ./frontend'
            }
        }

        stage('Build Backend Image') {
            steps {
                sh 'docker build -t $BACKEND_IMAGE:$BUILD_NUMBER -t $BACKEND_IMAGE:latest ./backend'
            }
        }

        stage('Docker Login') {
            steps {
                sh '''
                echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin
                '''
            }
        }

        stage('Push Frontend Image') {
            steps {
                sh '''
                docker push $FRONTEND_IMAGE:$BUILD_NUMBER
                docker push $FRONTEND_IMAGE:latest
                '''
            }
        }

        stage('Push Backend Image') {
            steps {
                sh '''
                docker push $BACKEND_IMAGE:$BUILD_NUMBER
                docker push $BACKEND_IMAGE:latest
                '''
            }
        }

        stage('Deploy to App Server') {
            steps {
                sshagent(['app-server-ssh']) {
                    sh '''
ssh -o StrictHostKeyChecking=no $APP_SERVER << EOF
cd app
docker compose down
docker rmi your-dockerhub-username/frontend-app:latest -f || true
docker rmi your-dockerhub-username/backend-app:latest -f || true
docker compose pull
docker compose up -d
docker image prune -f
EOF
'''
                }
            }
        }
    }
}
```

**What each section does:**

- **`environment` block** — Defines reusable variables. `credentials('dockerhub-creds')` securely pulls the Docker Hub credentials from Jenkins's internal store. Jenkins automatically splits them into `_USR` (username) and `_PSW` (password) variables.
- **`Clone Repository` stage** — Pulls the latest code from GitHub. The `branch: 'main'` parameter specifies which branch to track.
- **`Build Frontend/Backend Image` stages** — Runs `docker build` pointing to the `./frontend` and `./backend` directories. Notice that two tags are applied: one with `$BUILD_NUMBER` (for versioning) and one as `:latest` (for simple deployment).
- **`Docker Login` stage** — Uses `--password-stdin` to securely pass the Docker Hub token without exposing it in the process list.
- **`Push` stages** — Pushes both the numbered and `:latest` tags to Docker Hub, so rollback is always possible.
- **`Deploy to App Server` stage** — SSHes into the App Server using the stored private key, removes stale cached images, pulls fresh ones, and restarts containers.

> ⚠️ **Critical — EOF Indentation:** The `EOF` keyword and the `ssh` command at the beginning of the heredoc **must start at column 1 with no leading spaces or tabs.** If `EOF` is indented, the shell treats it as a regular command and you'll see:
> ```
> -bash: line XX: EOF: command not found
> ```
> This is one of the most common mistakes in Jenkinsfile heredocs.

---

#### Step 32 — Create the Jenkins Pipeline Job

- In Jenkins → **New Item** → Enter a name → Select **Pipeline** → Click **OK**
- Under Pipeline section, choose **Pipeline script from SCM**
- SCM: **Git**
- Repository URL: your GitHub repo URL
- Branches to build: `*/main`
- Script Path: `Jenkinsfile`
- Click **Save**

> [Insert Screenshot Here: Jenkins pipeline job configuration screen]

---

### Running the Pipeline

#### Step 33 — Trigger Your First Build

In Jenkins, open your pipeline job and click **Build Now**. Watch the build progress in real time by clicking the build number → **Console Output**.

> [Insert Screenshot Here: Jenkins pipeline stages view showing all green stages]

**Expected successful output:**
```
[Pipeline] stage (Clone Repository)    ✔
[Pipeline] stage (Build Frontend Image) ✔
[Pipeline] stage (Build Backend Image)  ✔
[Pipeline] stage (Docker Login)         ✔
[Pipeline] stage (Push Frontend Image)  ✔
[Pipeline] stage (Push Backend Image)   ✔
[Pipeline] stage (Deploy to App Server) ✔
Finished: SUCCESS
```

---

## Challenges Encountered and How They Were Solved

This is the part most tutorials skip — the real errors that happen in practice.

---

### Error 1 — Wrong Branch Name (master vs main)

```
fatal: couldn't find remote ref refs/heads/master
ERROR: Couldn't find any revision to build.
```

**What happened:** Jenkins defaulted to `*/master` but the GitHub repository uses `main` as the default branch (GitHub changed this default in 2020).

**Fix:** Go to the Jenkins job → **Configure** → **Branches to build** → change `*/master` to `*/main` → **Save** → **Build Again**.

---

### Error 2 — SSH Credentials Not Found

```
[ssh-agent] Could not find specified credentials: app-server-ssh
Finished: FAILURE
```

**What happened:** The credential ID in the Jenkinsfile doesn't match what was stored in Jenkins. Either the credential wasn't added yet, or there was a typo in the ID.

**Fix:** Double-check that the credential was created with the exact ID `app-server-ssh` (Step 24). The Jenkinsfile is case-sensitive.

---

### Error 3 — docker-compose Not Found on App Server

```
-bash: line 4: docker-compose: command not found
```

**What happened:** The Jenkinsfile used `docker-compose` (with a hyphen) but the App Server has the newer `docker-compose-v2` package, which installs the command as `docker compose` (with a space).

**Fix 1:** Install the compatibility package on the App Server:
```bash
sudo apt install docker-compose-v2 -y
```

**Fix 2:** Update the Jenkinsfile's Deploy stage to use `docker compose` everywhere instead of `docker-compose`.

---

### Error 4 — EOF: command not found

```
-bash: line XX: EOF: command not found
```

**What happened:** The `EOF` terminator in the Jenkinsfile heredoc had leading whitespace due to editor indentation. The shell never found a matching `EOF` at column 1, so it treated it as a command.

**Fix:** In the Jenkinsfile, ensure the `ssh` command and the `EOF` line start at absolute column 1, with no spaces or tabs before them. Refer to the Jenkinsfile in Step 31 for correct formatting.

---

### Error 5 — Docker Permission Denied on Jenkins

```
permission denied while trying to connect to the Docker daemon socket
```

**What happened:** The Jenkins user wasn't yet in the `docker` group, or the group change hadn't taken effect.

**Fix:**
```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
sudo reboot
```

Reconnect SSH after the reboot and retry the pipeline.

---

## Verifying the Deployment

After a successful pipeline run, SSH into the App Server and confirm both containers are running:

```bash
ssh ubuntu@YOUR_APP_SERVER_PUBLIC_IP
cd app
docker compose ps
```

**Expected output:**
```
NAME                 STATUS          PORTS
frontend-container   Up X seconds    0.0.0.0:3000->3000/tcp
backend-container    Up X seconds    0.0.0.0:5000->5000/tcp
```

Then open your browser and verify:

| Service | URL |
|---|---|
| Frontend | `http://YOUR_APP_SERVER_PUBLIC_IP:3000` |
| Backend | `http://YOUR_APP_SERVER_PUBLIC_IP:5000` |

> [Insert Screenshot Here: Both containers running in docker compose ps output]

---

## Image Versioning and Rollback

One issue with tagging everything as `:latest` is that rollbacks become impossible — every deploy overwrites the previous image. Here's how the dual-tag strategy in this pipeline solves that.

After a few pipeline runs, Docker Hub will show:

```
your-dockerhub-username/frontend-app:1
your-dockerhub-username/frontend-app:2
your-dockerhub-username/frontend-app:3
your-dockerhub-username/frontend-app:latest  ← always the newest
```

**To roll back to build 7** if build 8 breaks something:

SSH into the App Server, edit the compose file:

```bash
nano ~/app/docker-compose.yml
```

Change the image tags to the old build number:

```yaml
services:
  frontend:
    image: your-dockerhub-username/frontend-app:7
  backend:
    image: your-dockerhub-username/backend-app:7
```

Then restart:

```bash
docker compose pull
docker compose down
docker compose up -d
```

That's it. You're back on the previous known-good version within seconds.

> ⚠️ **Why `docker rmi` before `docker compose pull`?** Docker caches image layers locally. Without removing the old `:latest` image first, `docker compose pull` might decide nothing has changed and skip downloading the new version. Your updated code would never actually run on the App Server. Always force-remove the cached image before pulling.

---

## Key Learnings

A few things that would have saved a lot of time upfront:

**1. The APT repository for Jenkins is unreliable on Ubuntu 24.** Going straight to the `.deb` package download avoids a class of installation failures.

**2. Group membership changes in Linux don't always take effect immediately.** Adding a user to the `docker` group sometimes requires a full reboot — not just a session restart — to propagate.

**3. `docker-compose` vs `docker compose` is a real compatibility gotcha.** The old hyphenated command is being phased out. Always verify which version is installed before writing pipeline scripts.

**4. Heredoc indentation in Jenkinsfiles is unforgiving.** Shell heredocs require the closing delimiter at column 1. Editors that auto-indent will silently break your pipeline.

**5. SSH key setup with EC2 instances requires manual steps.** AWS's default SSH configuration disables password auth, so `ssh-copy-id` won't work out of the box. The manual `authorized_keys` approach is the reliable alternative.

---

## Best Practices Followed

- **Credentials are never in the Jenkinsfile.** Docker Hub credentials and SSH keys are stored in Jenkins's secure credential store and referenced only by ID.
- **Dual image tagging** — both `:latest` and `BUILD_NUMBER` — enables simple deployment while preserving rollback capability.
- **Two separate EC2 instances** — separating the CI server from the application server prevents build activity from affecting production and mirrors real production architecture.
- **`StrictHostKeyChecking=no`** is used in the SSH command to prevent the pipeline from hanging on an interactive host verification prompt during first connection.
- **`|| true`** is appended to the `docker rmi` commands so the pipeline doesn't fail if the image doesn't exist yet on a fresh server.
- **`docker image prune -f`** at the end of each deploy keeps disk usage clean on the App Server.

---

## Final Results

What was built at the end of this setup:

- A fully automated CI/CD pipeline triggered by `git push`
- Docker images built and versioned on every pipeline run
- Images pushed to Docker Hub with numbered tags for rollback support
- A full-stack application automatically deployed to an EC2 App Server via SSH and Docker Compose
- End-to-end pipeline execution time: approximately 3–5 minutes per build
- Zero manual steps required after initial setup

---

## Conclusion

This kind of automation setup is one of those things that feels complex upfront but pays for itself within the first week. The initial investment in getting Jenkins, Docker, and SSH working together properly is what makes every subsequent deploy completely friction-free.

The scenario of someone manually deploying at midnight with shaky hands, second-guessing every command? That's the old way. This pipeline changes the equation: every code push is treated exactly the same way, every time, with full logs, version history, and rollback capability built in.

Once something like this is in place, you realize how much cognitive overhead manual deployments actually carry — and you never want to go back.

---

## Future Improvements

A few things worth adding to evolve this setup further:

- **Webhook-based triggers** instead of polling — currently Jenkins polls GitHub for changes. Setting up a GitHub webhook makes builds trigger instantly on push.
- **Automated testing stage** — adding a `Run Tests` stage before the build stages means broken code never reaches Docker Hub.
- **Slack or email notifications** — Jenkins can send build success/failure notifications to a Slack channel.
- **SSL/TLS termination** — putting an Nginx reverse proxy with Let's Encrypt certificates in front of the App Server makes the application production-ready.
- **AWS ECR instead of Docker Hub** — for production workloads, AWS Elastic Container Registry is more integrated with the AWS ecosystem and has better access control.
- **Monitoring** — integrating Prometheus and Grafana to track container health and resource usage closes the operational visibility gap.

---

## License

> The scripts and code in this article are free to use and modify for personal or commercial use. No attribution required, but always appreciated. Use at your own risk in production — always test in a safe environment first.

---

Feel free to connect and discuss further — whether you hit a different error, want to extend this pipeline, or just want to geek out about DevOps and cloud infrastructure.

📧 **Email:** [bhavyat2520@gmail.com](mailto:bhavyat2520@gmail.com)

---

*Thanks for reading! If this helped you, consider leaving a clap or sharing it with someone setting up their first CI/CD pipeline.*
