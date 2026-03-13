# Deploying Code-Sync on AWS

This guide covers how to deploy Code-Sync on Amazon Web Services (AWS). Two recommended options are described below, from simplest to most production-ready.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Option A — EC2 with Docker (Quickest)](#option-a--ec2-with-docker-quickest)
4. [Option B — EC2 with Nginx (Single-Server Production)](#option-b--ec2-with-nginx-single-server-production)
5. [Option C — ECS + ECR (Scalable / Container-Native)](#option-c--ecs--ecr-scalable--container-native)
6. [Environment Variables Reference](#environment-variables-reference)
7. [Security Group Rules](#security-group-rules)
8. [HTTPS with Let's Encrypt](#https-with-lets-encrypt)
9. [Troubleshooting](#troubleshooting)

---

## Architecture Overview

Code-Sync consists of two parts:

| Component | Technology | Default Port |
|-----------|-----------|--------------|
| **Server** | Node.js + Express + Socket.io | 3000 |
| **Client** | React + Vite (static build) | 80 (via nginx) |

The server handles all real-time WebSocket connections. The client is a static Single Page Application (SPA) that connects to the server via WebSocket.

> **Tip:** The server already serves static files from its `public/` directory. If you want a single-process deployment you can build the frontend, copy it into `server/public/`, and run only the server.

---

## Prerequisites

- An AWS account
- A domain name (optional but recommended for HTTPS)
- Basic familiarity with the AWS Console or AWS CLI

---

## Option A — EC2 with Docker (Quickest)

This option uses the existing `docker-compose.yml` to run both containers on one EC2 instance.

### 1. Launch an EC2 Instance

1. Open **EC2 → Launch Instance** in the AWS Console.
2. Choose **Ubuntu 22.04 LTS** (or Amazon Linux 2).
3. Select an instance type: **t2.small** or larger is recommended.
4. Configure **Security Group** — see [Security Group Rules](#security-group-rules).
5. Create or select a key pair, then launch.

### 2. Connect to the Instance

```bash
ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>
```

### 3. Install Docker and Docker Compose

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

# Add Docker's official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Allow current user to run docker without sudo
sudo usermod -aG docker $USER
newgrp docker
```

### 4. Clone the Repository

```bash
git clone https://github.com/BhuvanXcode/Code-Sync1.git
cd Code-Sync1
```

### 5. Configure Environment Variables

```bash
# Replace the placeholder with your EC2 public IP or domain name
export VITE_BACKEND_URL=http://<EC2_PUBLIC_IP>:3000
```

Or create a `.env` file in the project root:

```bash
echo "VITE_BACKEND_URL=http://<EC2_PUBLIC_IP>:3000" > .env
```

### 6. Start the Application

```bash
docker compose up -d --build
```

### 7. Verify

- **Client (UI):** `http://<EC2_PUBLIC_IP>` (port 80)
- **Server (API/WebSocket):** `http://<EC2_PUBLIC_IP>:3000`

To view logs:

```bash
docker compose logs -f
```

---

## Option B — EC2 with Nginx (Single-Server Production)

This option runs the backend directly with **PM2** (process manager) and uses **Nginx** as a reverse proxy for both the API and the static frontend. No Docker required.

### 1. Launch EC2 and Connect

Follow steps 1–2 from [Option A](#option-a--ec2-with-docker-quickest).

### 2. Install Node.js 18, Nginx, Git, and PM2

```bash
sudo apt-get update
sudo apt-get install -y nginx git

# Install Node.js 18 via NodeSource
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Install PM2 globally
sudo npm install -g pm2
```

### 3. Clone the Repository

```bash
git clone https://github.com/BhuvanXcode/Code-Sync1.git
cd Code-Sync1
```

### 4. Build the Frontend

```bash
cd client
npm install

# Set the backend URL (replace with your EC2 public IP or domain)
# Use https:// here once you have completed the Let's Encrypt step (step 9)
VITE_BACKEND_URL=http://<EC2_PUBLIC_IP>:3000 npm run build
cd ..
```

### 5. Build the Backend

```bash
cd server
npm install
npm run build
cd ..
```

### 6. Deploy Frontend to Server's `public/` Directory

The Express server is already configured to serve static files from `server/public/`. Copying the frontend build there means only one process handles everything:

```bash
mkdir -p server/public
cp -r client/dist/* server/public/
```

### 7. Configure the Server Environment

```bash
cp server/.env.example server/.env
# Optionally edit server/.env to change PORT
```

### 8. Start the Server with PM2

```bash
cd server
pm2 start dist/server.js --name code-sync
pm2 save
pm2 startup   # Follow the printed command to enable auto-start on reboot
cd ..
```

### 9. Configure Nginx as a Reverse Proxy

Create an Nginx site configuration:

```bash
sudo tee /etc/nginx/sites-available/code-sync > /dev/null << 'EOF'
server {
    listen 80;
    server_name <YOUR_DOMAIN_OR_EC2_IP>;

    # Proxy API and WebSocket traffic to the Node.js server
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;

        # WebSocket upgrade headers (required for Socket.io)
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_read_timeout 86400;
    }
}
EOF

sudo ln -s /etc/nginx/sites-available/code-sync /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

### 10. Verify

Open `http://<EC2_PUBLIC_IP>` in your browser. The app should be running.

---

## Option C — ECS + ECR (Scalable / Container-Native)

Use this option when you want AWS-managed container orchestration with auto-scaling and load balancing.

### 1. Install Prerequisites Locally

```bash
# AWS CLI
pip install awscli
aws configure   # Enter your AWS Access Key ID, Secret, region, and output format

# Docker must also be installed locally
```

### 2. Create ECR Repositories

```bash
AWS_REGION=us-east-1   # Change to your preferred region
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create repositories
aws ecr create-repository --repository-name code-sync-server --region $AWS_REGION
aws ecr create-repository --repository-name code-sync-client --region $AWS_REGION
```

### 3. Build and Push Docker Images

```bash
# Authenticate Docker to ECR
aws ecr get-login-password --region $AWS_REGION | \
  docker login --username AWS --password-stdin \
  $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

# Build and push the server image
docker build -t code-sync-server ./server
docker tag code-sync-server:latest \
  $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/code-sync-server:latest
docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/code-sync-server:latest

# Build and push the client image (pass backend URL as a build argument)
# Replace http:// with https:// once your ALB has an HTTPS listener configured
docker build \
  --build-arg VITE_BACKEND_URL=http://<ALB_DNS_NAME>:3000 \
  -t code-sync-client ./client
docker tag code-sync-client:latest \
  $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/code-sync-client:latest
docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/code-sync-client:latest
```

### 4. Create an ECS Cluster

In the **AWS Console → ECS → Clusters → Create Cluster**:

- Cluster name: `code-sync`
- Infrastructure: **AWS Fargate** (serverless, no EC2 management)

### 5. Create Task Definitions

Create two task definitions — one for the server and one for the client — using the images pushed in step 3. Key settings per task:

**Server task:**
| Setting | Value |
|---------|-------|
| Launch type | Fargate |
| CPU | 0.5 vCPU |
| Memory | 1 GB |
| Container port | 3000 |
| Environment variable | `PORT=3000` |

**Client task:**
| Setting | Value |
|---------|-------|
| Launch type | Fargate |
| CPU | 0.25 vCPU |
| Memory | 512 MB |
| Container port | 80 |

### 6. Create an Application Load Balancer (ALB)

1. Go to **EC2 → Load Balancers → Create Load Balancer → Application Load Balancer**.
2. Add two target groups:
   - `code-sync-server` → port 3000
   - `code-sync-client` → port 80
3. Add listener rules:
   - `/socket.io/*` and `/api/*` → forward to `code-sync-server`
   - All other requests → forward to `code-sync-client`

> **Important:** Enable **sticky sessions** on the server target group so WebSocket connections are not broken by load-balancer routing changes.

### 7. Create ECS Services

Create one ECS Service per task definition, attaching them to the ALB target groups created above.

---

## Environment Variables Reference

### Server (`server/.env`)

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `3000` | Port the Express/Socket.io server listens on |

### Client (build-time)

| Variable | Default | Description |
|----------|---------|-------------|
| `VITE_BACKEND_URL` | `http://localhost:3000` | Full URL of the server (used by Socket.io client) |

> `VITE_BACKEND_URL` must be set at **build time** (not runtime) because Vite inlines it into the compiled JavaScript bundle.

---

## Security Group Rules

When creating or editing an EC2 Security Group, add the following **Inbound Rules**:

| Type | Protocol | Port | Source | Purpose |
|------|----------|------|--------|---------|
| SSH | TCP | 22 | Your IP | Remote management |
| HTTP | TCP | 80 | 0.0.0.0/0 | Client web traffic |
| HTTPS | TCP | 443 | 0.0.0.0/0 | Client web traffic (SSL) |
| Custom TCP | TCP | 3000 | 0.0.0.0/0 | Server API / WebSocket |

> Once Nginx is configured as a reverse proxy (Option B) you can restrict port 3000 to the EC2's own security group and remove it from public access.

---

## HTTPS with Let's Encrypt

After completing Option A or B, secure the site with a free TLS certificate:

```bash
sudo apt-get install -y certbot python3-certbot-nginx
sudo certbot --nginx -d <YOUR_DOMAIN>
```

Certbot will automatically update the Nginx configuration. Certificates renew automatically every 90 days.

> Make sure your domain's DNS **A record** points to the EC2 instance's public IP before running certbot.

---

## Troubleshooting

### WebSocket connection fails

- Ensure port `3000` is open in the Security Group (or Nginx is proxying it).
- Check that `VITE_BACKEND_URL` was set correctly **at build time** to the server's public URL.
- If using an ALB, verify sticky sessions are enabled on the server target group.

### Application crashes after reboot (Option B)

Run `pm2 startup` and execute the command it prints. Then run `pm2 save` to persist the process list.

### Containers not starting (Option A)

```bash
docker compose logs server
docker compose logs client
```

### Check server logs (Option B)

```bash
pm2 logs code-sync
```

### Port 80 already in use

```bash
sudo systemctl status nginx
sudo lsof -i :80
```
