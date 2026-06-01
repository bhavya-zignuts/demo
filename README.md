# How a 3AM Production Crash Taught Me to Build Zero-Downtime Deployments

It was 3:17 AM on a Tuesday when my phone started buzzing. Customer support tickets were flooding in. The e-commerce platform I was responsible for had gone completely dark during what should have been a routine deployment. Twenty minutes of downtime meant thousands of dollars in lost revenue and a very uncomfortable conversation with stakeholders the next morning.

That incident changed everything for me. I realized that traditional deployment strategies — taking the application offline, updating code, and hoping everything works — simply weren't acceptable in today's always-on digital world. I needed a bulletproof deployment strategy that could update applications without anyone even noticing.

That's when I discovered Blue-Green deployment, and in this article, I'll show you exactly how to build a production-grade zero-downtime deployment pipeline that would have saved me from that 3 AM nightmare.

## Why Zero Downtime Matters More Than Ever

In our hyperconnected world, even a few minutes of downtime can be catastrophic. Users expect applications to be available 24/7, and every second of unavailability translates to lost revenue, damaged reputation, and frustrated customers. Traditional deployment approaches create an inevitable window of risk where something can go wrong, leaving your application in an inconsistent or broken state.

Blue-Green deployment eliminates this risk entirely. Instead of updating your live application directly, you deploy to a parallel environment, thoroughly test it, and then instantly switch traffic over. If something goes wrong, you can switch back in seconds, not hours.

## Project Overview

In this comprehensive guide, we'll build a complete CI/CD pipeline that implements Blue-Green deployment using industry-standard tools. The system automatically pulls code from GitHub, builds Docker images, deploys to an inactive environment, runs health checks, and switches live traffic — all without a single second of downtime.

Our application consists of a frontend served by Nginx and a Node.js backend API, both containerized separately for maximum flexibility. The entire pipeline is orchestrated by Jenkins and runs on AWS EC2 infrastructure, making it production-ready and scalable.

## Architecture and Workflow

Here's how our zero-downtime deployment system works:

```
┌─────────────────┐    git push    ┌─────────────────┐    polls    ┌─────────────────┐
│ Developer       │ ──────────────→ │ GitHub Repo     │ ───────────→ │ Jenkins Server  │
│ (Your Machine)  │                │                 │             │ (EC2 Instance)  │
└─────────────────┘                └─────────────────┘             └─────────────────┘
                                                                            │
                                                                            │ SSH Deploy
                                                                            ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           Application Server (EC2 Instance)                         │
│                                                                                     │
│  ┌─────────────────┐                           ┌─────────────────┐                 │
│  │ Blue Environment│                           │Green Environment│                 │
│  │ frontend:3001   │                           │ frontend:3002   │                 │
│  │ backend:5001    │                           │ backend:5002    │                 │
│  └─────────────────┘                           └─────────────────┘                 │
│           │                                             │                          │
│           └─────────────────┐       ┌───────────────────┘                          │
│                             ▼       ▼                                              │
│                     ┌─────────────────┐                                            │
│                     │ Nginx (Port 80) │                                            │
│                     │ Reverse Proxy   │                                            │
│                     └─────────────────┘                                            │
│                             │                                                      │
└─────────────────────────────┼──────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ Browser/Users   │
                    │ (Port 80)       │
                    └─────────────────┘
```

The genius of this architecture is that at any given moment, only one environment (Blue or Green) receives live traffic while the other remains ready for the next deployment. Nginx acts as the traffic director, seamlessly switching between environments when deployments complete successfully.

## Prerequisites

Before we dive into the implementation, you'll need:

- An AWS account with EC2 access
- Basic familiarity with Linux command line
- A GitHub account for source code hosting
- A Docker Hub account for image storage
- Understanding of basic networking concepts

We'll be using two EC2 instances: a Jenkins server for orchestration and an application server for hosting our Blue-Green environments.

## Step-by-Step Implementation

### Setting Up the Infrastructure

Let's start by creating the foundation for our deployment pipeline. We need two EC2 instances that will work together to provide zero-downtime deployments.

First, we'll launch our Jenkins server. This instance needs enough resources to build Docker images and coordinate deployments, so we'll use a t2.medium instance.

```bash
# Launch Jenkins Server (t2.medium, Ubuntu 24.04 LTS)
# Security Group: SSH (22), Custom TCP (8080), Custom TCP (50000)
```

The Jenkins server needs port 8080 for the web interface and port 50000 for build agent communication. We're using Ubuntu 24.04 LTS for its stability and long-term support.

Next, we'll create the application server that will host our Blue and Green environments.

```bash
# Launch Application Server (t2.small, Ubuntu 24.04 LTS)  
# Security Group: SSH (22), HTTP (80), Custom TCP (3001, 3002, 5001, 5002)
```

⚠️ **Warning:** Make sure to save your key pair file (blue-green-key.pem) securely, as you'll need it for SSH access to both instances.

[Insert Screenshot Here: AWS EC2 console showing both instances running with proper security groups configured]

### Installing Jenkins on the Server

Now we'll set up Jenkins on our dedicated server. Modern Jenkins requires Java 21, and we'll install it manually for better control over the setup process.

First, let's install Java 21, which is the minimum required version for current Jenkins releases.

```bash
sudo apt update
sudo apt-get install -y openjdk-21-jdk
```

We need Java 21 because newer Jenkins versions have deprecated support for older Java versions due to security and performance improvements. This ensures our Jenkins installation will be compatible with the latest features and security patches.

Next, we'll create a dedicated Jenkins user for security isolation:

```bash
sudo adduser --system --home /var/lib/jenkins --shell /bin/bash jenkins
sudo groupadd jenkins
sudo usermod -g jenkins jenkins
```

This creates a system user specifically for running Jenkins, following security best practices by not running applications as root.

Now we'll download and set up Jenkins manually:

```bash
sudo mkdir -p /var/lib/jenkins
sudo wget -O /var/lib/jenkins/jenkins.war https://get.jenkins.io/war-stable/latest/jenkins.war
sudo chown -R jenkins:jenkins /var/lib/jenkins
```

⚠️ **Warning:** If you get a "invalid group" error, make sure you've created the jenkins group first with the `groupadd` command above.

Let's create a systemd service to manage Jenkins:

```bash
sudo tee /etc/systemd/system/jenkins.service > /dev/null <<EOF
[Unit]
Description=Jenkins Automation Server
Requires=network.target
After=network.target

[Service]
Type=notify
User=jenkins
Group=jenkins
ExecStart=/usr/lib/jvm/java-21-openjdk-amd64/bin/java -Djenkins.install.runSetupWizard=false -jar /var/lib/jenkins/jenkins.war
WorkingDirectory=/var/lib/jenkins
StandardOutput=journal
StandardError=journal
SyslogIdentifier=jenkins

[Install]
WantedBy=multi-user.target
EOF
```

This service configuration ensures Jenkins starts automatically on boot and runs with appropriate permissions.

```bash
sudo systemctl daemon-reload
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

Let's verify Jenkins is running:

```bash
sudo systemctl status jenkins
```

[Insert Screenshot Here: Terminal showing Jenkins service status as active and running]

### Installing Docker on Jenkins Server

Jenkins will need Docker to build our application images. Let's install Docker and configure it properly for Jenkins integration.

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get update
sudo apt-get install -y docker-ce
```

The critical step is adding the Jenkins user to the Docker group so it can build images without sudo:

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

After adding Jenkins to the docker group, we must restart the Jenkins service for the group membership to take effect. This is crucial for preventing permission errors during the build process.

⚠️ **Warning:** Wait at least 40 seconds after restarting Jenkins before running any builds, as the service needs time to fully initialize with the new group permissions.

### Setting Up the Application Server

Now let's prepare our application server to host the Blue-Green environments. This server needs Docker for running containers and Nginx as our reverse proxy.

```bash
# SSH to Application Server
sudo apt update
sudo apt-get install -y docker-ce docker-compose-plugin nginx
sudo systemctl enable docker nginx
sudo systemctl start docker nginx
```

Create the directory structure for our Blue-Green deployment scripts:

```bash
sudo mkdir -p /opt/blue-green/scripts
sudo chown -R ubuntu:ubuntu /opt/blue-green
```

We're using /opt/blue-green as our deployment directory because /opt is the standard location for optional software packages in Linux systems.

### Configuring Jenkins Web Interface

Access Jenkins at `http://your-jenkins-server-ip:8080`. You'll need the initial admin password, which you can retrieve with:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

[Insert Screenshot Here: Jenkins unlock screen with password field]

Install suggested plugins when prompted, then create your admin user. This setup ensures Jenkins has all the necessary plugins for our CI/CD pipeline.

### Setting Up SSH Key Authentication

For Jenkins to deploy to the application server, we need secure SSH key authentication. Let's generate a dedicated key pair on the Jenkins server.

```bash
# On Jenkins Server, switch to jenkins user
sudo su - jenkins
ssh-keygen -t rsa -b 4096 -f ~/.ssh/app-server-key
```

We're using RSA 4096-bit keys for enhanced security. Copy the public key content:

```bash
cat ~/.ssh/app-server-key.pub
```

Now, on the Application Server, add this public key to authorized_keys:

```bash
# On Application Server
mkdir -p ~/.ssh
echo "PASTE_PUBLIC_KEY_HERE" >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Test the SSH connection from Jenkins server:

```bash
# From Jenkins server as jenkins user
ssh -i ~/.ssh/app-server-key -o StrictHostKeyChecking=no ubuntu@YOUR_APP_SERVER_IP 'echo SSH works!'
```

✅ **Tip:** The StrictHostKeyChecking=no option prevents Jenkins from hanging on unknown host prompts during automated deployments.

### Creating the Application Code

Let's build a simple but complete application that demonstrates real-world deployment scenarios. Our frontend will be a single-page application served by Nginx, and our backend will be a Node.js API server.

Here's our frontend HTML file (frontend/src/index.html):

```html
<!DOCTYPE html>
<html lang='en'>
<head>
  <meta charset='UTF-8' />
  <title>Blue-Green App</title>
  <style>
    body { 
      font-family: Arial, sans-serif; 
      display: flex; 
      justify-content: center; 
      align-items: center; 
      min-height: 100vh; 
      background: #f0f4f8; 
    }
    .card { 
      background: white; 
      padding: 40px 60px; 
      border-radius: 12px; 
      text-align: center; 
    }
    .version { 
      background: #48bb78; 
      color: white; 
      padding: 4px 16px; 
      border-radius: 20px; 
    }
  </style>
</head>
<body>
  <div class='card'>
    <h1>Hello from Frontend!</h1>
    <p>Blue-Green Deployment Project</p>
    <span class='version'>Version 1</span>
    <div>
      <button onclick='callBackend()'>Call /api/hello</button>
      <div id='api-result'></div>
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

This frontend includes a version indicator and a button to test backend connectivity, allowing us to verify both components are working after each deployment.

Create the Nginx configuration for the frontend container (frontend/nginx.conf):

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

The try_files directive ensures that our single-page application works correctly with client-side routing.

Here's the Dockerfile for our frontend (frontend/Dockerfile):

```dockerfile
FROM nginx:alpine
RUN rm /etc/nginx/conf.d/default.conf
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY src/ /usr/share/nginx/html/
EXPOSE 3000
CMD ["nginx", "-g", "daemon off;"]
```

We're using the Alpine Linux variant for smaller image sizes and faster deployment times.

Now let's create our Node.js backend. First, the package.json (backend/package.json):

```javascript
{
  "name": "backend-app",
  "version": "1.0.0",
  "main": "src/server.js",
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

The backend server code (backend/src/server.js):

```javascript
const express = require('express');
const app = express();
const PORT = 5000;

app.get('/', (req, res) => { 
  res.json({ 
    message: 'Hello from Backend!', 
    version: '1', 
    environment: process.env.ENV || 'unknown' 
  }); 
});

app.get('/hello', (req, res) => { 
  res.json({ 
    message: 'Hello from Backend API!', 
    version: '1', 
    environment: process.env.ENV || 'unknown' 
  }); 
});

app.get('/health', (req, res) => { 
  res.status(200).json({ 
    status: 'healthy', 
    version: '1' 
  }); 
});

app.listen(PORT, () => console.log('Backend running on port ' + PORT));
```

The /health endpoint is crucial for our automated health checks. The environment variable allows us to distinguish between Blue and Green deployments.

Backend Dockerfile (backend/Dockerfile):

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package.json ./
RUN npm install --production
COPY src/ ./src/
EXPOSE 5000
CMD ["node", "src/server.js"]
```

### Creating Docker Compose Files

The key to successful Blue-Green deployment is having completely isolated environments. Here's our Blue environment configuration (docker-compose.blue.yml):

```yaml
name: blue-project
services:
  frontend-blue:
    image: bhavyatank13/frontend-app:${TAG}
    container_name: frontend-blue
    ports: ["3001:3000"]
    environment: [ENV=blue]
    restart: unless-stopped
    networks: [blue-network]
  backend-blue:
    image: bhavyatank13/backend-app:${TAG}
    container_name: backend-blue
    ports: ["5001:5000"]
    environment: [ENV=blue]
    restart: unless-stopped
    networks: [blue-network]
networks:
  blue-network:
    name: blue-network
```

The Green environment configuration (docker-compose.green.yml):

```yaml
name: green-project
services:
  frontend-green:
    image: bhavyatank13/frontend-app:${TAG}
    container_name: frontend-green
    ports: ["3002:3000"]
    environment: [ENV=green]
    restart: unless-stopped
    networks: [green-network]
  backend-green:
    image: bhavyatank13/backend-app:${TAG}
    container_name: backend-green
    ports: ["5002:5000"]
    environment: [ENV=green]
    restart: unless-stopped
    networks: [green-network]
networks:
  green-network:
    name: green-network
```

💡 **Critical Note:** Each compose file has a unique project name (blue-project/green-project) and unique network name. This isolation was the key to solving the 502 error we'll discuss later.

The ${TAG} variable will be populated by Jenkins with the build number, ensuring each deployment uses the correct image version.

### Creating Deployment Scripts

These scripts handle the core deployment logic and are executed by Jenkins via SSH. Let's create each script with detailed explanations.

First, the Blue deployment script (/opt/blue-green/scripts/deploy-blue.sh):

```bash
#!/bin/bash
set -e
TAG=$1

if [ -z "$TAG" ]; then 
    echo "ERROR: TAG required"; 
    exit 1; 
fi

echo "=== Deploying BLUE | Tag: $TAG ==="
cd /opt/blue-green
export TAG=$TAG

docker pull bhavyatank13/frontend-app:$TAG
docker pull bhavyatank13/backend-app:$TAG

docker compose -f docker-compose.blue.yml down --remove-orphans || true
docker compose -f docker-compose.blue.yml up -d

echo "=== Blue deployed ==="
```

• `set -e` ensures the script exits immediately if any command fails
• We explicitly pull images to ensure we have the latest version
• `--remove-orphans` cleans up any containers from previous deployments
• `|| true` prevents the script from failing if no containers exist to stop

The Green deployment script is nearly identical (/opt/blue-green/scripts/deploy-green.sh):

```bash
#!/bin/bash
set -e
TAG=$1

if [ -z "$TAG" ]; then 
    echo "ERROR: TAG required"; 
    exit 1; 
fi

echo "=== Deploying GREEN | Tag: $TAG ==="
cd /opt/blue-green
export TAG=$TAG

docker pull bhavyatank13/frontend-app:$TAG
docker pull bhavyatank13/backend-app:$TAG

docker compose -f docker-compose.green.yml down --remove-orphans || true
docker compose -f docker-compose.green.yml up -d

echo "=== Green deployed ==="
```

Now, the health check script (/opt/blue-green/scripts/health-check.sh):

```bash
#!/bin/bash
ENV=$1
MAX_RETRIES=10
SLEEP_SECONDS=5

if [ "$ENV" = "blue" ]; then 
    FRONTEND_PORT=3001; 
    BACKEND_PORT=5001;
elif [ "$ENV" = "green" ]; then 
    FRONTEND_PORT=3002; 
    BACKEND_PORT=5002;
else 
    echo "ERROR: specify blue or green"; 
    exit 1; 
fi

echo "=== Health Check: $ENV ==="
sleep 10

for i in $(seq 1 $MAX_RETRIES); do
    STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:$FRONTEND_PORT)
    if [ "$STATUS" = "200" ]; then 
        echo "Frontend healthy!"; 
        break; 
    fi
    echo "Attempt $i/$MAX_RETRIES - waiting..."; 
    sleep $SLEEP_SECONDS
    if [ $i -eq $MAX_RETRIES ]; then 
        echo "FAILED"; 
        exit 1; 
    fi
done

for i in $(seq 1 $MAX_RETRIES); do
    STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:$BACKEND_PORT/health)
    if [ "$STATUS" = "200" ]; then 
        echo "Backend healthy!"; 
        break; 
    fi
    echo "Attempt $i/$MAX_RETRIES - waiting..."; 
    sleep $SLEEP_SECONDS
    if [ $i -eq $MAX_RETRIES ]; then 
        echo "FAILED"; 
        exit 1; 
    fi
done

echo "=== Health check PASSED ==="
```

• The initial 10-second sleep gives containers time to fully initialize
• We test both frontend and backend endpoints to ensure complete functionality
• The retry logic handles temporary startup delays gracefully

The traffic switching script (/opt/blue-green/scripts/switch.sh):

```bash
#!/bin/bash
ENV=$1

if [ "$ENV" = "blue" ]; then 
    FRONTEND_PORT=3001; 
    BACKEND_PORT=5001;
elif [ "$ENV" = "green" ]; then 
    FRONTEND_PORT=3002; 
    BACKEND_PORT=5002;
else 
    echo "ERROR: specify blue or green"; 
    exit 1; 
fi

sudo tee /etc/nginx/sites-available/blue-green > /dev/null <<NGINX
upstream frontend_active { 
    server 127.0.0.1:${FRONTEND_PORT}; 
}
upstream backend_active  { 
    server 127.0.0.1:${BACKEND_PORT};  
}
server {
    listen 80; 
    server_name _;
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

• The Nginx configuration uses upstream blocks for better load balancing
• The `/api/` location block strips the `/api` prefix before forwarding to backend
• We save the active environment to a file for the next deployment to detect
• `nginx -t` validates configuration before reloading

Finally, the rollback script (/opt/blue-green/scripts/rollback.sh):

```bash
#!/bin/bash
ENV=$1

if [ -z "$ENV" ]; then
    CURRENT=$(cat /opt/blue-green/active-env 2>/dev/null)
    if [ "$CURRENT" = "blue" ]; then 
        ENV="green"; 
    else 
        ENV="blue"; 
    fi
    echo "Auto rollback: $CURRENT -> $ENV"
fi

bash /opt/blue-green/scripts/switch.sh $ENV
echo "=== Rollback complete. Now pointing to: $ENV ==="
```

This script can automatically determine the previous environment for instant rollbacks.

Make all scripts executable:

```bash
chmod +x /opt/blue-green/scripts/*.sh
```

### Creating the Jenkins Pipeline

Now we'll create the Jenkins pipeline that orchestrates the entire deployment process. In Jenkins, create a new Pipeline job and use this Jenkinsfile:

```groovy
pipeline {
    agent any
    
    environment {
        DOCKER_HUB_USER = 'bhavyatank13'
        FRONTEND_IMAGE  = 'bhavyatank13/frontend-app'
        BACKEND_IMAGE   = 'bhavyatank13/backend-app'
        APP_SERVER_IP   = 'YOUR_APP_SERVER_IP'
        APP_SERVER_USER = 'ubuntu'
        TAG             = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }
        
        stage('Build Docker Images') {
            steps {
                script {
                    echo "Building Frontend Image..."
                    sh "cd frontend && docker build -t ${FRONTEND_IMAGE}:${TAG} ."
                    
                    echo "Building Backend Image..."
                    sh "cd backend && docker build -t ${BACKEND_IMAGE}:${TAG} ."
                }
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', 
                                                    usernameVariable: 'DOCKER_USER', 
                                                    passwordVariable: 'DOCKER_PASS')]) {
                        sh "echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin"
                        sh "docker push ${FRONTEND_IMAGE}:${TAG}"
                        sh "docker push ${BACKEND_IMAGE}:${TAG}"
                    }
                }
            }
        }
        
        stage('Detect Active Environment') {
            steps {
                script {
                    def result = sh(script: """
                        ssh -i /var/lib/jenkins/.ssh/app-server-key -o StrictHostKeyChecking=no ${APP_SERVER_USER}@${APP_SERVER_IP} '
                            cat /opt/blue-green/active-env 2>/dev/null || echo blue
                        '
                    """, returnStdout: true).trim()
                    
                    env.ACTIVE_ENV = result
                    env.INACTIVE_ENV = (result == 'blue') ? 'green' : 'blue'
                    
                    echo "Active: ${env.ACTIVE_ENV}, Deploying to: ${env.INACTIVE_ENV}"
                }
            }
        }
        
        stage('Deploy to Inactive Environment') {
            steps {
                script {
                    sh """
                        ssh -i /var/lib/jenkins/.ssh/app-server-key -o StrictHostKeyChecking=no ${APP_SERVER_USER}@${APP_SERVER_IP} '
                            bash /opt/blue-green/scripts/deploy-${env.INACTIVE_ENV}.sh ${TAG}
                        '
                    """
                }
            }
        }
        
        stage('Health Check') {
            steps {
                script {
                    sh """
                        ssh -i /var/lib/jenkins/.ssh/app-server-key -o StrictHostKeyChecking=no ${APP_SERVER_USER}@${APP_SERVER_IP} '
                            bash /opt/blue-green/scripts/health-check.sh ${env.INACTIVE_ENV}
                        '
                    """
                }
            }
        }
        
        stage('Switch Nginx Traffic') {
            steps {
                script {
                    sh """
                        ssh -i /var/lib/jenkins/.ssh/app-server-key -o StrictHostKeyChecking=no ${APP_SERVER_USER}@${APP_SERVER_IP} '
                            bash /opt/blue-green/scripts/switch.sh ${env.INACTIVE_ENV}
                        '
                    """
                }
            }
        }
        
        stage('Remove Old Environment') {
            steps {
                script {
                    sh """
                        ssh -i /var/lib/jenkins/.ssh/app-server-key -o StrictHostKeyChecking=no ${APP_SERVER_USER}@${APP_SERVER_IP} '
                            cd /opt/blue-green && docker compose -f docker-compose.${env.ACTIVE_ENV}.yml down --remove-orphans
                        '
                    """
                }
            }
        }
    }
    
    post {
        success {
            echo "✅ Deployment SUCCESS! Active environment: ${env.INACTIVE_ENV}, Tag: ${TAG}"
        }
        failure {
            script {
                sh """
                    ssh -i /var/lib/jenkins/.ssh/app-server-key -o StrictHostKeyChecking=no ${APP_SERVER_USER}@${APP_SERVER_IP} '
                        bash /opt/blue-green/scripts/rollback.sh ${env.ACTIVE_ENV}
                    '
                """
            }
        }
    }
}
```

This pipeline follows the Blue-Green deployment pattern perfectly:

• **Checkout Code**: Pulls the latest code from your Git repository
• **Build Docker Images**: Creates containerized versions of both frontend and backend
• **Push to Docker Hub**: Makes images available for deployment
• **Detect Active Environment**: Determines which environment is currently live
• **Deploy to Inactive Environment**: Deploys new version without affecting live traffic
• **Health Check**: Verifies new deployment is working correctly
• **Switch Nginx Traffic**: Instantly redirects traffic to new environment
• **Remove Old Environment**: Cleans up previous version
• **Post Actions**: Handles success logging and automatic rollback on failure

[Insert Screenshot Here: Jenkins pipeline view showing all stages completed successfully]

### Copying Files to Application Server

Before running our first deployment, we need to copy our Docker Compose files and scripts to the application server. You can do this by cloning your repository or manually copying files:

```bash
# On Application Server
git clone https://github.com/your-username/blue-green-project.git /tmp/blue-green-repo
cp /tmp/blue-green-repo/docker-compose.blue.yml /opt/blue-green/
cp /tmp/blue-green-repo/docker-compose.green.yml /opt/blue-green/
cp -r /tmp/blue-green-repo/scripts/ /opt/blue-green/
chmod +x /opt/blue-green/scripts/*.sh
```

Initialize the active environment for the first deployment:

```bash
bash /opt/blue-green/scripts/switch.sh blue
```

This creates
