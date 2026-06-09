# Zero Downtime Deployments with Blue-Green Strategy Using Jenkins, Docker, and Nginx on AWS EC2

> **A complete, production-grade walkthrough of automating blue-green deployments — from infrastructure setup to zero-downtime traffic switching — with every real error and fix documented.**

---

## The Incident That Made This Click

Picture this: it's a Friday evening, a team pushes a release to their e-commerce platform. The deployment takes the server down for nearly four minutes while the new version boots up. In those four minutes, hundreds of users hit a dead page. Orders are lost. Customer support gets flooded. The team rolls back, and now the whole weekend becomes a post-mortem.

That scenario plays out more often than people admit — not because the code was broken, but because the **deployment strategy itself was flawed**. The idea that you have to take something offline to update it is a design choice, not a technical necessity.

Reading about this kind of incident is what pushed me to explore blue-green deployment properly — not just as a theoretical concept, but as a fully implemented, automated pipeline. This article is the result of that deep-dive.

---

## Why This Problem Matters

Traditional deployment workflows have a fundamental flaw: there's always a window where the old version is gone but the new version isn't ready yet. Even if that window is thirty seconds, it's thirty seconds of downtime — and in production, that's unacceptable.

Blue-green deployment solves this by keeping two identical environments alive. One serves users right now. The other receives the new version quietly in the background. When the new version passes a health check, Nginx flips the traffic — instantly, atomically, with zero dropped requests.

The benefits compound quickly:

- **Zero downtime** on every release, regardless of the size of the change.
- **Instant rollback** — if something goes wrong after the switch, flipping back takes seconds.
- **Safe testing** — the new version is fully deployed and validated before a single user sees it.
- **Build traceability** — every Docker image is tagged with the Jenkins build number, so you always know exactly what's running.

---

## Project Overview

Here's what this project implements end-to-end:

- Two AWS EC2 instances: one for Jenkins CI/CD, one for the application environments.
- A Jenkins pipeline that triggers on every git push, builds Docker images, pushes them to Docker Hub, and orchestrates the full deployment.
- Two isolated environments — **Blue** (ports 3001/5001) and **Green** (ports 3002/5002) — alternating with each deployment.
- Nginx on the application server acting as the single entry point, dynamically switching upstream targets.
- Shell scripts that handle deployment, health checks, traffic switching, and rollback.
- A minimal Node.js + Express backend and an Nginx-served HTML frontend, both containerised.

---

## Architecture and Workflow

```
Your Local Machine
      |
      |  git push
      v
GitHub Repository
      |
      |  Jenkins polls / triggered manually
      v
Jenkins Server (EC2 #1)
      |  docker build frontend + backend images
      |  docker push --> Docker Hub
      |  SSH --> App Server
      v
Application Server (EC2 #2)
      |
      |-- Blue Environment:  frontend-blue:3001, backend-blue:5001
      |-- Green Environment: frontend-green:3002, backend-green:5002
      |
      v
Nginx (port 80)
      |-- /       --> active frontend container
      |-- /api/   --> active backend container
      v
Browser: http://<APP_SERVER_IP>
```

### The Switch Logic

At any moment, exactly one environment is live and the other is idle. The pipeline:

1. Reads `/opt/blue-green/active-env` to know which environment is currently serving traffic.
2. Deploys the new version to the **inactive** environment.
3. Runs health checks — waits up to 50 seconds with retries before declaring success.
4. Updates the Nginx config and reloads it (a graceful reload — no dropped connections).
5. Writes the new active environment to `active-env`.
6. Tears down the old environment's containers.

The next deployment reverses direction. Blue → Green → Blue → Green, cycling automatically forever.

---

## Prerequisites

Before starting, make sure you have the following in place:

| Requirement | Details |
|---|---|
| AWS Account | Active account with EC2 permissions in `ap-south-1` (Mumbai) or your preferred region |
| Docker Hub Account | Account at hub.docker.com — images will be pushed here |
| GitHub Account | A repository to host your code and Jenkinsfile |
| SSH Client | Terminal with SSH support (Mac/Linux or Windows PowerShell) |
| Local Git | Git installed locally for pushing code |
| Browser | Any modern browser for accessing the Jenkins UI |

---

## Step-by-Step Implementation

### Step 1: Create EC2 Instances on AWS

Log in to the [AWS Console](https://console.aws.amazon.com) and confirm your region in the top-right corner.

#### 1.1 Jenkins Server (EC2 #1)

Go to **EC2 → Launch Instance** and configure:

```
Name:           jenkins-server
AMI:            Ubuntu Server 24.04 LTS (HVM), SSD Volume Type
Instance Type:  t2.medium  (minimum 2GB RAM — Jenkins is memory-hungry)
Key Pair:       Create new → Name: blue-green-key → Download .pem file
Storage:        20 GB gp3
```

> ⚠️ **Warning:** Download the `.pem` file immediately. AWS only lets you download it once. Lose it and you cannot SSH into your instance.

Under **Security Group**, create `jenkins-sg` with these inbound rules:

| Type | Port | Source |
|---|---|---|
| SSH | 22 | My IP only |
| Custom TCP | 8080 | 0.0.0.0/0 (Jenkins UI) |
| Custom TCP | 50000 | 0.0.0.0/0 (Jenkins agent communication) |

[Insert Screenshot Here: EC2 Launch Instance form filled out for jenkins-server]

#### 1.2 Application Server (EC2 #2)

Launch a second instance:

```
Name:           app-server
AMI:            Ubuntu Server 24.04 LTS (HVM)
Instance Type:  t2.small
Key Pair:       Select existing → blue-green-key
Storage:        20 GB gp3
```

Create `app-server-sg` with these inbound rules:

| Type | Port | Purpose |
|---|---|---|
| SSH | 22 | My IP only |
| HTTP | 80 | Public Nginx entry point |
| Custom TCP | 3001 | Blue frontend (for direct testing) |
| Custom TCP | 3002 | Green frontend (for direct testing) |
| Custom TCP | 5001 | Blue backend (for direct testing) |
| Custom TCP | 5002 | Green backend (for direct testing) |

#### 1.3 Connect via SSH

Once both instances show **Running** status, note their public IPs from the EC2 dashboard. Then connect:

```bash
# Fix key permissions first — required on Mac/Linux
chmod 400 ~/Downloads/blue-green-key.pem

# Connect to Jenkins Server
ssh -i ~/Downloads/blue-green-key.pem ubuntu@<JENKINS_IP>

# Open a second terminal and connect to App Server
ssh -i ~/Downloads/blue-green-key.pem ubuntu@<APP_SERVER_IP>
```

The `chmod 400` is mandatory — SSH will outright refuse to use a key file that has overly permissive permissions.

[Insert Screenshot Here: Both EC2 instances showing "Running" status in the AWS dashboard]

---

### Step 2: Install Tools on the Jenkins Server

All commands in this section run on the **Jenkins Server** terminal.

#### 2.1 Install Java 21

Jenkins's latest versions require Java 21 as a minimum. Java 17 will cause a startup failure.

```bash
sudo apt-get update -y
sudo apt-get install -y openjdk-21-jdk
java -version
```

**Expected output:** `openjdk version "21.x.x" ...`

> 💡 **Note:** If you already have Java 17 installed from a previous attempt, run: `sudo apt-get install -y openjdk-21-jdk && sudo update-alternatives --set java /usr/lib/jvm/java-21-openjdk-amd64/bin/java` to switch the default.

#### 2.2 Create the Jenkins User and Directories

Jenkins runs as its own system user for security isolation. We create that user and set up its home and log directories:

```bash
sudo adduser --system --home /var/lib/jenkins --shell /bin/bash jenkins
sudo groupadd jenkins
sudo usermod -g jenkins jenkins
sudo mkdir -p /var/lib/jenkins
sudo mkdir -p /var/log/jenkins
sudo chown -R jenkins:jenkins /var/lib/jenkins
sudo chown -R jenkins:jenkins /var/log/jenkins
```

> ⚠️ **Warning:** The `--system` flag on `adduser` does not automatically create a matching group. If the `chown` commands fail with `invalid group: 'jenkins:jenkins'`, run `sudo groupadd jenkins && sudo usermod -g jenkins jenkins` first, then retry.

#### 2.3 Download Jenkins via WAR File

The standard Jenkins apt repository had GPG key verification issues on Ubuntu 24.04. Downloading the WAR file directly is the reliable path — it's the same Jenkins, just installed differently:

```bash
sudo wget -O /var/lib/jenkins/jenkins.war \
  https://get.jenkins.io/war-stable/latest/jenkins.war

sudo chown jenkins:jenkins /var/lib/jenkins/jenkins.war
ls -lh /var/lib/jenkins/jenkins.war
```

**Expected output:** `-rw-r--r-- 1 jenkins jenkins 95M ... jenkins.war`

#### 2.4 Create a Systemd Service for Jenkins

Instead of running Jenkins manually, we register it as a proper system service so it starts automatically on reboot and restarts on failure:

```bash
sudo tee /etc/systemd/system/jenkins.service > /dev/null <<'EOF'
[Unit]
Description=Jenkins Automation Server
After=network.target

[Service]
Type=simple
User=jenkins
Environment="JENKINS_HOME=/var/lib/jenkins"
Environment="JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64"
ExecStart=/usr/lib/jvm/java-21-openjdk-amd64/bin/java -jar /var/lib/jenkins/jenkins.war --httpPort=8080
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
```

#### 2.5 Start Jenkins

```bash
sudo systemctl daemon-reload
sudo systemctl start jenkins
sudo systemctl enable jenkins
sleep 40 && sudo systemctl status jenkins --no-pager
```

The `sleep 40` gives Jenkins time to fully initialise before checking its status.

**Expected output:** `Active: active (running) since ...`

[Insert Screenshot Here: Jenkins systemd status showing "active (running)"]

#### 2.6 Install Docker on the Jenkins Server

The Jenkins pipeline builds Docker images, so Docker must be present — and critically, the `jenkins` user must be in the `docker` group:

```bash
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update -y
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ubuntu
sudo usermod -aG docker jenkins
docker --version
```

> ⚠️ **Warning:** Adding the `jenkins` user to the `docker` group only takes effect after restarting Jenkins. If the pipeline fails with `permission denied while trying to connect to the Docker daemon socket`, run: `sudo usermod -aG docker jenkins && sudo systemctl restart jenkins` — then wait 40 seconds before re-running the build.

#### 2.7 Set Up the Jenkins UI

Get the initial admin password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Open your browser at `http://<JENKINS_IP>:8080`, paste the password, click **Install suggested plugins**, wait 2–3 minutes, then create your admin user.

[Insert Screenshot Here: Jenkins "Getting Started" screen showing plugin installation in progress]

---

### Step 3: Install Tools on the Application Server

Switch to the **App Server** terminal. Run this complete installation script in one go:

```bash
sudo apt-get update -y && sudo apt-get upgrade -y

# Install Docker
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update -y
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ubuntu

# Install Nginx
sudo apt-get install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx

# Create deployment directories
sudo mkdir -p /opt/blue-green/scripts
sudo mkdir -p /opt/blue-green/logs
sudo chown -R ubuntu:ubuntu /opt/blue-green
```

After the script completes, verify everything installed correctly:

```bash
docker --version
docker compose version
nginx -v
```

Log out and back in so the `docker` group membership takes effect:

```bash
exit
ssh -i ~/Downloads/blue-green-key.pem ubuntu@<APP_SERVER_IP>
docker ps   # Should show an empty list with no permission error
```

---

### Step 4: Set Up the GitHub Repository

All the code, scripts, Dockerfiles, and the Jenkinsfile live in one GitHub repository. Jenkins fetches this automatically on every build.

#### 4.1 Repository Structure

```
blue-green-project-2/
├── frontend/
│   ├── src/
│   │   └── index.html
│   ├── Dockerfile
│   └── nginx.conf
├── backend/
│   ├── src/
│   │   └── server.js
│   ├── package.json
│   └── Dockerfile
├── docker-compose.blue.yml
├── docker-compose.green.yml
├── scripts/
│   ├── deploy-blue.sh
│   ├── deploy-green.sh
│   ├── health-check.sh
│   ├── switch.sh
│   └── rollback.sh
└── Jenkinsfile
```

Clone the repo and create the directories locally:

```bash
git clone https://github.com/<your-username>/<your-repo>.git
cd <your-repo>
mkdir -p frontend/src backend/src scripts
```

#### 4.2 Frontend Application

**File: `frontend/src/index.html`**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Blue-Green App</title>
  <style>
    body { font-family: Arial, sans-serif; display: flex;
           justify-content: center; align-items: center;
           min-height: 100vh; background: #f0f4f8; }
    .card { background: white; padding: 40px 60px;
            border-radius: 12px; text-align: center; }
    .version { background: #48bb78; color: white;
               padding: 4px 16px; border-radius: 20px; }
  </style>
</head>
<body>
  <div class="card">
    <h1>Hello from Frontend!</h1>
    <p>Blue-Green Deployment Project</p>
    <span class="version">Version 1</span>
    <div>
      <button onclick="callBackend()">Call /api/hello</button>
      <div id="api-result"></div>
    </div>
  </div>
  <script>
    async function callBackend() {
      const res = await fetch('/api/hello');
      const data = await res.json();
      document.getElementById('api-result').textContent = JSON.stringify(data);
    }
  </script>
</body>
</html>
```

**File: `frontend/nginx.conf`**

```nginx
server {
    listen 3000;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

This configures Nginx inside the container to serve static files on port 3000 (which gets mapped to 3001 or 3002 on the host).

**File: `frontend/Dockerfile`**

```dockerfile
FROM nginx:alpine
RUN rm /etc/nginx/conf.d/default.conf
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY src/ /usr/share/nginx/html/
EXPOSE 3000
CMD ["nginx", "-g", "daemon off;"]
```

#### 4.3 Backend Application

**File: `backend/src/server.js`**

```javascript
const express = require('express');
const app = express();
const PORT = 5000;

app.get('/', (req, res) => {
  res.json({ message: 'Hello from Backend!', version: '1',
             environment: process.env.ENV || 'unknown' });
});

app.get('/hello', (req, res) => {
  res.json({ message: 'Hello from Backend API!', version: '1',
             environment: process.env.ENV || 'unknown' });
});

app.get('/health', (req, res) => {
  res.status(200).json({ status: 'healthy', version: '1' });
});

app.listen(PORT, () => console.log(`Backend running on port ${PORT}`));
```

The `/health` endpoint is what the health check script polls. It must return HTTP 200 — that's the signal the container is ready.

**File: `backend/package.json`**

```json
{
  "name": "backend-app",
  "version": "1.0.0",
  "main": "src/server.js",
  "scripts": { "start": "node src/server.js" },
  "dependencies": { "express": "^4.18.2" }
}
```

**File: `backend/Dockerfile`**

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package.json ./
RUN npm install --production
COPY src/ ./src/
EXPOSE 5000
CMD ["node", "src/server.js"]
```

Using `node:18-alpine` keeps the image small. `--production` skips dev dependencies.

#### 4.4 Docker Compose Files

> ⚠️ **Critical:** Each compose file must have a unique `name` field and a unique network name. Without this, Docker treats both as the same project and removes the wrong containers during cleanup — this caused a major issue that's documented in the Challenges section below.

**File: `docker-compose.blue.yml`**

```yaml
name: blue-project

services:
  frontend-blue:
    image: bhavyatank13/frontend-app:${TAG}
    container_name: frontend-blue
    ports:
      - "3001:3000"
    environment:
      - ENV=blue
    restart: unless-stopped
    networks:
      - blue-network

  backend-blue:
    image: bhavyatank13/backend-app:${TAG}
    container_name: backend-blue
    ports:
      - "5001:5000"
    environment:
      - ENV=blue
    restart: unless-stopped
    networks:
      - blue-network

networks:
  blue-network:
    name: blue-network
```

**File: `docker-compose.green.yml`**

```yaml
name: green-project

services:
  frontend-green:
    image: bhavyatank13/frontend-app:${TAG}
    container_name: frontend-green
    ports:
      - "3002:3000"
    environment:
      - ENV=green
    restart: unless-stopped
    networks:
      - green-network

  backend-green:
    image: bhavyatank13/backend-app:${TAG}
    container_name: backend-green
    ports:
      - "5002:5000"
    environment:
      - ENV=green
    restart: unless-stopped
    networks:
      - green-network

networks:
  green-network:
    name: green-network
```

#### 4.5 Deployment Scripts

**File: `scripts/deploy-blue.sh`**

```bash
#!/bin/bash
set -e
TAG=$1
if [ -z "$TAG" ]; then echo "ERROR: TAG required"; exit 1; fi
echo "=== Deploying BLUE | Tag: $TAG ==="
cd /opt/blue-green
export TAG=$TAG
docker pull bhavyatank13/frontend-app:$TAG
docker pull bhavyatank13/backend-app:$TAG
docker compose -f docker-compose.blue.yml down --remove-orphans || true
docker compose -f docker-compose.blue.yml up -d
echo "=== Blue deployed ==="
```

What this script does:
- Accepts the image tag (build number) as its first argument.
- Pulls the latest images from Docker Hub explicitly — ensuring the newest version is used rather than a cached local copy.
- Brings down any existing blue containers cleanly before starting fresh.
- Starts the new containers in detached mode.

**File: `scripts/deploy-green.sh`** (mirrors deploy-blue.sh but for the green environment)

```bash
#!/bin/bash
set -e
TAG=$1
if [ -z "$TAG" ]; then echo "ERROR: TAG required"; exit 1; fi
echo "=== Deploying GREEN | Tag: $TAG ==="
cd /opt/blue-green
export TAG=$TAG
docker pull bhavyatank13/frontend-app:$TAG
docker pull bhavyatank13/backend-app:$TAG
docker compose -f docker-compose.green.yml down --remove-orphans || true
docker compose -f docker-compose.green.yml up -d
echo "=== Green deployed ==="
```

**File: `scripts/health-check.sh`**

```bash
#!/bin/bash
ENV=$1
MAX_RETRIES=10
SLEEP_SECONDS=5
if [ "$ENV" = "blue" ]; then FRONTEND_PORT=3001; BACKEND_PORT=5001;
elif [ "$ENV" = "green" ]; then FRONTEND_PORT=3002; BACKEND_PORT=5002;
else echo "ERROR: specify blue or green"; exit 1; fi

echo "=== Health Check: $ENV ==="
sleep 10   # Wait for containers to fully start

for i in $(seq 1 $MAX_RETRIES); do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:$FRONTEND_PORT)
  if [ "$STATUS" = "200" ]; then echo "Frontend healthy!"; break; fi
  echo "Attempt $i/$MAX_RETRIES - waiting..."; sleep $SLEEP_SECONDS
  if [ $i -eq $MAX_RETRIES ]; then echo "FAILED"; exit 1; fi
done

for i in $(seq 1 $MAX_RETRIES); do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:$BACKEND_PORT/health)
  if [ "$STATUS" = "200" ]; then echo "Backend healthy!"; break; fi
  echo "Attempt $i/$MAX_RETRIES - waiting..."; sleep $SLEEP_SECONDS
  if [ $i -eq $MAX_RETRIES ]; then echo "FAILED"; exit 1; fi
done
echo "=== Health check PASSED ==="
```

What this script does:
- Waits 10 seconds after deployment for containers to fully initialise.
- Polls both the frontend and backend separately, retrying up to 10 times with 5-second gaps (up to 50 seconds total).
- Exits with a non-zero code if either check fails — causing the Jenkins stage to fail and triggering the rollback.

**File: `scripts/switch.sh`**

```bash
#!/bin/bash
ENV=$1
if [ "$ENV" = "blue" ]; then FRONTEND_PORT=3001; BACKEND_PORT=5001;
elif [ "$ENV" = "green" ]; then FRONTEND_PORT=3002; BACKEND_PORT=5002;
else echo "ERROR: specify blue or green"; exit 1; fi

sudo tee /etc/nginx/sites-available/blue-green > /dev/null <<NGINX
upstream frontend_active { server 127.0.0.1:${FRONTEND_PORT}; }
upstream backend_active  { server 127.0.0.1:${BACKEND_PORT};  }
server {
    listen 80; server_name _;
    location / {
        proxy_pass http://frontend_active;
        proxy_set_header Host \$host;
    }
    location /api/ {
        rewrite ^/api(/.*)$ \$1 break;
        proxy_pass http://backend_active;
        proxy_set_header Host \$host;
    }
}
NGINX

sudo ln -sf /etc/nginx/sites-available/blue-green /etc/nginx/sites-enabled/blue-green
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
echo $ENV | sudo tee /opt/blue-green/active-env > /dev/null
echo "=== Nginx now pointing to: $ENV ==="
```

What this script does:
- Dynamically writes a fresh Nginx config file pointing to the correct port.
- Creates a symlink in `sites-enabled` so Nginx picks it up.
- Removes the default Nginx site to avoid conflicts.
- Validates the config with `nginx -t` before reloading — if the config is broken, the reload never happens, protecting the current live environment.
- Writes the new environment name to the `active-env` file so the next Jenkins build knows which environment to target.

**File: `scripts/rollback.sh`**

```bash
#!/bin/bash
ENV=$1
if [ -z "$ENV" ]; then
  CURRENT=$(cat /opt/blue-green/active-env 2>/dev/null)
  if [ "$CURRENT" = "blue" ]; then ENV="green"; else ENV="blue"; fi
  echo "Auto rollback: $CURRENT -> $ENV"
fi
bash /opt/blue-green/scripts/switch.sh $ENV
echo "=== Rollback complete. Now pointing to: $ENV ==="
```

Rollback is just a traffic switch. It auto-detects which environment is currently live and flips to the opposite one.

#### 4.6 The Jenkinsfile

This is the heart of the pipeline. Every stage is connected:

```groovy
pipeline {
    agent any
    environment {
        DOCKER_HUB_USER = 'bhavyatank13'
        FRONTEND_IMAGE  = 'bhavyatank13/frontend-app'
        BACKEND_IMAGE   = 'bhavyatank13/backend-app'
        APP_SERVER_IP   = '<YOUR_APP_SERVER_IP>'
        APP_SERVER_USER = 'ubuntu'
        TAG             = "${BUILD_NUMBER}"
    }
    stages {
        stage('Checkout Code') {
            steps { checkout scm }
        }
        stage('Build Docker Images') {
            steps {
                sh """
                    docker build -t ${FRONTEND_IMAGE}:${TAG} ./frontend
                    docker build -t ${BACKEND_IMAGE}:${TAG}  ./backend
                """
            }
        }
        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${FRONTEND_IMAGE}:${TAG}
                        docker push ${BACKEND_IMAGE}:${TAG}
                    """
                }
            }
        }
        stage('Detect Active Environment') {
            steps {
                script {
                    def result = sh(script: """ssh -i /var/lib/jenkins/.ssh/app-server-key \\
                        -o StrictHostKeyChecking=no ${APP_SERVER_USER}@${APP_SERVER_IP} \\
                        'cat /opt/blue-green/active-env 2>/dev/null || echo none'""",
                        returnStdout: true).trim()
                    if (result == 'blue') { env.ACTIVE_ENV='blue'; env.INACTIVE_ENV='green' }
                    else                  { env.ACTIVE_ENV='green'; env.INACTIVE_ENV='blue' }
                }
            }
        }
        stage('Deploy to Inactive Environment') {
            steps {
                sh """ssh -i /var/lib/jenkins/.ssh/app-server-key \\
                    -o StrictHostKeyChecking=no ${APP_SERVER_USER}@${APP_SERVER_IP} \\
                    'bash /opt/blue-green/scripts/deploy-${env.INACTIVE_ENV}.sh ${TAG}'"""
            }
        }
        stage('Health Check') {
            steps {
                sh """ssh -i /var/lib/jenkins/.ssh/app-server-key \\
                    -o StrictHostKeyChecking=no ${APP_SERVER_USER}@${APP_SERVER_IP} \\
                    'bash /opt/blue-green/scripts/health-check.sh ${env.INACTIVE_ENV}'"""
            }
        }
        stage('Switch Nginx Traffic') {
            steps {
                sh """ssh -i /var/lib/jenkins/.ssh/app-server-key \\
                    -o StrictHostKeyChecking=no ${APP_SERVER_USER}@${APP_SERVER_IP} \\
                    'bash /opt/blue-green/scripts/switch.sh ${env.INACTIVE_ENV}'"""
            }
        }
        stage('Remove Old Environment') {
            steps {
                sh """ssh -i /var/lib/jenkins/.ssh/app-server-key \\
                    -o StrictHostKeyChecking=no ${APP_SERVER_USER}@${APP_SERVER_IP} \\
                    'cd /opt/blue-green && TAG=${TAG} docker compose -f docker-compose.${env.ACTIVE_ENV}.yml down --remove-orphans || true'"""
            }
        }
    }
    post {
        success { echo "SUCCESS! Active: ${env.INACTIVE_ENV} | Tag: ${TAG}" }
        failure {
            sh """ssh -i /var/lib/jenkins/.ssh/app-server-key \\
                -o StrictHostKeyChecking=no ${APP_SERVER_USER}@${APP_SERVER_IP} \\
                'bash /opt/blue-green/scripts/rollback.sh ${env.ACTIVE_ENV}' || true"""
        }
    }
}
```

What each stage does:
- **Checkout Code** — fetches the latest commit from the GitHub branch.
- **Build Docker Images** — builds separate images for frontend and backend, tagged with the Jenkins `BUILD_NUMBER`.
- **Push to Docker Hub** — authenticates using stored credentials and pushes both images.
- **Detect Active Environment** — SSHs into the app server and reads `active-env` to figure out which environment is live.
- **Deploy to Inactive** — runs the appropriate deploy script on the app server with the new image tag.
- **Health Check** — waits for both containers to respond with HTTP 200.
- **Switch Nginx** — atomically moves traffic to the new environment.
- **Remove Old** — tears down the previous environment's containers to free resources.
- **post.failure** — if anything above fails, automatically switches Nginx back to the previous environment.

#### 4.7 Push to GitHub

```bash
chmod +x scripts/*.sh
git add .
git commit -m "v1 - initial project setup"
git push origin main
```

---

### Step 5: Create Docker Hub Repositories

Log in to [hub.docker.com](https://hub.docker.com) and create two public repositories:

- `frontend-app` → Public
- `backend-app` → Public

> 💡 **Note:** Jenkins will push images tagged with the build number (1, 2, 3, ...) to these repos. Each build gets a unique, immutable tag — so you can always roll back to any previous version by its number.

[Insert Screenshot Here: Docker Hub showing both repositories created and public]

---

### Step 6: Set Up SSH Key (Jenkins → App Server)

Jenkins needs to SSH into the app server to run deployment scripts. We use key-based authentication — no passwords involved.

#### 6.1 Generate the Key on the Jenkins Server

```bash
# Switch to the jenkins user
sudo su - jenkins

# Generate the key pair (press Enter for all prompts — no passphrase)
ssh-keygen -t rsa -b 4096 -f ~/.ssh/app-server-key

# Copy the public key — you'll paste this on the app server next
cat ~/.ssh/app-server-key.pub
```

Copy the entire output starting with `ssh-rsa`.

#### 6.2 Add the Public Key to the App Server

On the **App Server** terminal:

```bash
nano ~/.ssh/authorized_keys
# Paste the public key on a new line, then save (Ctrl+X, Y, Enter)

chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

#### 6.3 Test the Connection

Back on the Jenkins Server (still as the `jenkins` user):

```bash
ssh -i ~/.ssh/app-server-key -o StrictHostKeyChecking=no ubuntu@<APP_SERVER_IP> "echo SSH works!"
exit   # exit jenkins user back to ubuntu
```

**Expected output:** `SSH works!`

> ⚠️ **Warning:** If you get `Permission denied (publickey)`, verify the key was pasted correctly in `authorized_keys` and that the file permissions are set exactly as above. A single extra character or wrong permission will break SSH auth.

---

### Step 7: Copy Files to the App Server

The app server needs the compose files and scripts locally so Jenkins can trigger them via SSH:

```bash
# On the App Server
git clone https://github.com/<your-username>/<your-repo>.git /tmp/blue-green-repo
cp /tmp/blue-green-repo/docker-compose.blue.yml  /opt/blue-green/
cp /tmp/blue-green-repo/docker-compose.green.yml /opt/blue-green/
cp -r /tmp/blue-green-repo/scripts/ /opt/blue-green/
chmod +x /opt/blue-green/scripts/*.sh

# Verify
ls -la /opt/blue-green/
ls -la /opt/blue-green/scripts/
```

> ⚠️ **Important:** Whenever you update compose files or scripts in GitHub, you must also update them on the App Server manually. Jenkins only deploys your application — it doesn't sync infrastructure scripts automatically.

---

### Step 8: Configure Jenkins

#### 8.1 Add Docker Hub Credentials

In Jenkins UI: **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**

```
Kind:        Username with password
Username:    <your Docker Hub username>
Password:    <your Docker Hub password>
ID:          dockerhub-credentials   ← exact ID used in Jenkinsfile
Description: Docker Hub credentials
```

> ⚠️ **Warning:** The `credentialsId` in the Jenkinsfile must exactly match the ID you set here. A typo causes the push stage to fail with an authentication error.

#### 8.2 Create the Pipeline Job

- Click **New Item**
- Name: `blue-green-pipeline`
- Select: **Pipeline** → Click OK
- Scroll to Pipeline section:

```
Definition:       Pipeline script from SCM
SCM:              Git
Repository URL:   https://github.com/<your-username>/<your-repo>.git
Branch:           */main
Script Path:      Jenkinsfile
```

Click **Save**.

[Insert Screenshot Here: Jenkins pipeline job configuration showing SCM settings]

---

### Step 9: Initial Nginx Setup on the App Server

Before the first Jenkins build, initialise Nginx to point to blue. This also creates the `active-env` file that the pipeline reads on its first run:

```bash
bash /opt/blue-green/scripts/switch.sh blue
```

**Expected output:** `=== Nginx now pointing to: blue ===`

Verify Nginx is healthy:

```bash
sudo nginx -t
sudo systemctl status nginx --no-pager
```

---

### Step 10: Run the First Jenkins Build

Go to **Jenkins UI → blue-green-pipeline → Build Now**. Click the build number, then **Console Output** to watch the stages execute in real time.

#### What the Pipeline Does on the First Run

| Stage | What Happens |
|---|---|
| Checkout Code | Pulls latest from GitHub |
| Build Docker Images | Builds both images tagged as build `#1` |
| Push to Docker Hub | Pushes `frontend-app:1` and `backend-app:1` |
| Detect Active Environment | Reads `active-env` → finds `blue` → sets inactive to `green` |
| Deploy to Inactive Env | Runs `deploy-green.sh 1` on the app server |
| Health Check | Waits and polls ports 3002 and 5002 |
| Switch Nginx Traffic | Updates Nginx to point to green |
| Remove Old Environment | Brings down blue containers |

[Insert Screenshot Here: Jenkins pipeline showing all stages green / successful]

After the build completes, verify on the app server:

```bash
docker ps                          # Should show green containers running
cat /opt/blue-green/active-env     # Should show: green
curl http://localhost/             # Should return the HTML page
curl http://localhost/api/health   # Should return: {"status":"healthy"}
```

Open your browser at `http://<APP_SERVER_IP>` — you should see "Hello from Frontend! Version 1".

[Insert Screenshot Here: Browser showing the frontend app with "Version 1" badge]

---

### Step 11: Deploy Version 2 — Watch the Switch Happen

Update your code locally to push a new version and see the blue-green switch in action.

#### 11.1 Update the Frontend

In `frontend/src/index.html`, change:

```html
<span class="version">Version 1</span>
```

to:

```html
<span class="version">Version 2</span>
```

Update `backend/src/server.js` — change all instances of `version: '1'` to `version: '2'`.

#### 11.2 Push and Trigger

```bash
git add .
git commit -m "version 2"
git push origin main
```

Then in Jenkins, click **Build Now**. The pipeline now detects green as active and deploys to blue. After the health check passes, Nginx switches to blue and tears down green.

Open the browser at `http://<APP_SERVER_IP>` — it now shows "Version 2". No interruption, no reload required.

[Insert Screenshot Here: Browser showing the frontend app with "Version 2" badge after the switch]

---

### Step 12: Manual Rollback

If a deployment causes issues after going live, roll back by switching Nginx to the previous environment. Since the old containers were removed by the pipeline, you first need to start them with the old image tag.

#### 12.1 Find the Previous Image Tag

```bash
docker images | grep frontend-app
```

Note the older build number.

#### 12.2 Start the Old Environment

```bash
cd /opt/blue-green
export TAG=<old-build-number>
docker compose -f docker-compose.blue.yml up -d
```

#### 12.3 Switch Nginx Back

```bash
bash /opt/blue-green/scripts/switch.sh blue
```

The previous version is live again within seconds. This is the core advantage — rollback is not a new deployment, it's just a traffic redirect.

---

## Challenges Faced and How They Were Solved

This section documents every real error encountered during the project — because these are exactly the problems you'll hit too.

### Challenge 1: Jenkins GPG Key Verification Failed

**Error:** `W: GPG error: https://pkg.jenkins.io/debian-stable Release: NO_PUBKEY 7198F4B714ABFC68 / E: The repository is not signed`

The standard Jenkins apt repository had a GPG key issue on Ubuntu 24.04. Rather than wrestling with GPG key imports, the cleaner solution was to download the Jenkins WAR file directly from the official source and run it as a systemd service. Same functionality, no repository dependency.

---

### Challenge 2: Jenkins Failed to Start — Java Version Too Old

**Error:** `Running with Java 17 — minimum required version is Java 21. Supported versions: [21, 25]`

Installing Java 17 first seemed reasonable since it's still widely used. However, the latest Jenkins WAR requires Java 21 minimum. Fix: install `openjdk-21-jdk` and update the systemd service's `ExecStart` path to the Java 21 binary.

---

### Challenge 3: Jenkins User Group Issue

**Error:** `chown: invalid group: 'jenkins:jenkins'`

The `adduser --system` command creates a user but not a group by default. Running `sudo groupadd jenkins && sudo usermod -g jenkins jenkins` before the `chown` commands resolves this.

---

### Challenge 4: Docker Permission Denied in Pipeline

**Error:** `permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock`

Adding a user to a group via `usermod -aG` only takes effect after that user starts a new session. Since Jenkins was already running when the `jenkins` user was added to `docker`, it needed a restart to pick up the new group membership:

```bash
sudo usermod -aG docker jenkins && sudo systemctl restart jenkins
```

---

### Challenge 5: The Containers Were Being Removed Immediately

**Error:** `Health check failed: backend not healthy after 10 attempts / Error response from daemon: No such container: backend-green`

This was the most puzzling issue. The containers deployed successfully but vanished before the health check could reach them. The root cause: both compose files had the same implicit Docker project name (`blue-green`, derived from the directory name) and the same default network name. When the pipeline ran `docker compose down` on one file, it was actually removing containers from the other file too.

Fix: add explicit `name:` fields (`blue-project` and `green-project`) at the top of each compose file, and define distinct named networks (`blue-network` and `green-network`). This makes Docker treat them as completely separate projects.

---

### Challenge 6: Site Unreachable After a Successful Build

**Error:** Site showed 502 Bad Gateway or was completely unreachable, even though Jenkins reported deployment success.

The same project name conflict from Challenge 5 was the culprit. The "Remove Old Environment" stage was removing the newly deployed containers instead of the old ones. Fixed with the same compose file changes described above.

---

### Challenge 7: Rollback Failing on the First Ever Build

**Error:** `ROLLBACK to: null / ERROR: specify blue or green`

On the very first pipeline run, the `active-env` file didn't exist yet — so the Detect Active Environment stage couldn't read it. Fix: manually run `bash /opt/blue-green/scripts/switch.sh blue` on the app server before triggering the first build. This creates the `active-env` file with `blue` as the initial value.

---

## Key Learnings

A few things that would be done differently with hindsight:

**Unique project names from day one.** The Docker compose project name collision caused the most time-consuming debugging session. Always define explicit `name:` fields in compose files the moment you have more than one.

**Test scripts before wiring them to Jenkins.** Running each shell script manually on the app server before connecting them to the pipeline catches issues much faster than reading pipeline logs.

**The `active-env` file is critical infrastructure.** It's a simple text file, but the entire blue-green logic depends on it. Initialising it before the first build should be part of the server setup, not an afterthought.

**`nginx -t` before `systemctl reload` is non-negotiable.** The switch script validates the Nginx config before reloading. Without that guard, a config syntax error would take down the live site.

---

## Best Practices Followed

**Tag every image with the build number.** Using `${BUILD_NUMBER}` as the Docker tag means every image is traceable. You always know which Jenkins build produced which version, and you can redeploy any previous version by tag.

**Health check before traffic switch.** Traffic never switches unless both containers are confirmed healthy. This prevents routing users to a broken deployment.

**Automatic rollback in the pipeline's `post.failure` block.** If any stage fails, the pipeline automatically switches Nginx back to the previous environment — without any human intervention.

**Separate networks per environment.** Giving blue and green their own Docker networks prevents any risk of cross-environment container communication.

**Store credentials in Jenkins, not in code.** Docker Hub credentials are stored in Jenkins's credential store and injected via `withCredentials`. No passwords in the Jenkinsfile or the repository.

---

## Final Results

After the full setup:

- Deployments complete in approximately 2–3 minutes end-to-end.
- Zero downtime on every switch — Nginx reload is graceful and doesn't drop active connections.
- Rollback takes under 10 seconds (just a traffic switch, no redeployment).
- Every build is tagged and traceable on Docker Hub.
- The pipeline automatically recovers from failed deployments.

[Insert Screenshot Here: Jenkins showing multiple successful builds with incrementing build numbers]

[Insert Screenshot Here: Docker Hub showing tagged images with build numbers]

---

## Conclusion

Blue-green deployment is one of those patterns that seems complex until you actually implement it — and then it feels obvious. The entire strategy boils down to a simple principle: keep a spare environment ready, deploy quietly into it, verify it works, then switch. The hard part is the plumbing — wiring together Jenkins, Docker, Nginx, and SSH in a way that's automated, recoverable, and predictable.

This project covers that plumbing completely. Every tool choice has a reason, every script has a guard, and every failure path has a recovery. The result is a deployment pipeline that you can run with confidence — knowing that if anything goes wrong, you're never more than ten seconds away from the previous working version.

---

## Future Improvements

A few natural next steps beyond this implementation:

- **HTTPS with Let's Encrypt** — add Certbot to the app server and configure SSL termination at Nginx.
- **Custom domain** — connect a Route 53 hosted zone to the app server's Elastic IP.
- **Automated rollback threshold** — extend the health check to also verify response time or error rate, and auto-rollback if metrics degrade post-switch.
- **Multi-region deployment** — replicate the same blue-green pattern across two AWS regions for geographic redundancy.
- **Kubernetes migration** — replace Docker Compose with Kubernetes Deployments and Services, using the same blue-green concept but with native Kubernetes traffic splitting.
- **Slack / email notifications** — hook the Jenkins `post` block to send deployment success/failure alerts to a team channel.

---

## License

> The scripts and code in this article are free to use and modify for personal or commercial use. No attribution required, but always appreciated. Use at your own risk in production — always test in a safe environment first.

---

Feel free to connect and discuss with me. Whether you hit a different error, extended this setup in an interesting direction, or just want to talk DevOps — I'm always happy to hear from you.

📧 [bhavyat2520@gmail.com](mailto:bhavyat2520@gmail.com)

---

**Medium Tags:** `DevOps`, `Docker`, `AWS`, `Jenkins`, `Nginx`

**SEO Subtitle:** Learn how to implement zero-downtime blue-green deployments using Jenkins, Docker, and Nginx on AWS EC2 — with every real error and fix documented.

**Estimated Reading Time:** 18–22 minutes
