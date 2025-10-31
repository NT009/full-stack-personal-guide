# All Deployment Docs

This document contains deployment guides for different types of applications (frontend, backend, fullstack, etc.).  
Use this as a reference to avoid searching for the same steps again in the future.

---

## Table of Contents

1. [Deploying a Next.js Application to a DigitalOcean Droplet](#deploying-a-nextjs-application-to-a-digitalocean-droplet)
2. [Configuring AWS CLI for DigitalOcean Spaces or Ubuntu Server](#configuring-aws-cli-for-digitalocean-spaces-or-ubuntu-server)
3. [Continuous Deployment Workflow](#continuous-deployment-workflow)

---

## Deploying a Next.js Application to a DigitalOcean Droplet

### Step 1: Access the Droplet

Login to your Droplet using **Termius** or any SSH client.

```bash
ssh root@your_server_ip
```

### Step 2: Update and Upgrade Packages

```bash
sudo apt update && sudo apt upgrade -y
```

### Step 3: Install Node.js and npm

```bash
sudo apt install -y nodejs npm
```

### Step 4: Install Nginx

```bash
sudo apt install -y nginx
```

### Step 5: Install PM2

```bash
sudo npm install -g pm2
```

### step 6 : Configure Nginx as a Reverse Proxy

Create a new Nginx configuration file:

```bash
sudo nano /etc/nginx/sites-available/project-name
```

### step 7: Add the Configuration

Paste the following configuration into the file:

```bash
server {
    listen 80;
    server_name <your_domain_or_droplet_ip>;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

**Save and exit the file in Nano:**

- Press `CTRL + O` → then press `Enter` to save the file.
- Press `CTRL + X` to exit Nano.

### step 8:Enable the Configuration

Create a symbolic link to enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/project-name /etc/nginx/sites-enabled/
```

### Step 9: Test Nginx Configuration

```bash
sudo nginx -t
```

### Step 10: Restart Nginx

```bash
sudo systemctl restart nginx
```

### Step 10: Create a Swap File

```bash
sudo fallocate -l 2G /swapfile
```

### Step 11: Set the Correct Permissions

```bash
sudo chmod 600 /swapfile
```

### Step 12: Set Up the Swap Area

```bash
sudo mkswap /swapfile
```

### Step 13: Enable the Swap File

```bash
sudo swapon /swapfile
```

### Step 14: Make the Swap File Permanent

Edit the **fstab** file:

```bash
sudo nano /etc/fstab
```

Add the following line at the end of the file:

```bash
/swapfile none swap sw 0 0
```

### Step 15: Clone Your Next.js Repository

```bash
git clone <repository_url>
```

### Step 16: Navigate to the Project Directory

```bash
cd <project_directory>
```

### Step 17: Install Dependencies

```bash
npm install
```

### Step 18: Set Up Environment Variables

- Use **SFTP** to upload the `.env` file, **or**
- Create it manually with Nano and write your environment variables inside.

```bash
nano .env
```

### Step 19: Build the Application

```bash
npm run build
```

### Step 20: Start the Application Using PM2

```bash
pm2 start npm --name "project-name" -- start
```

### Step 21: Save the PM2 Process List

```bash
pm2 save
```

### Step 22: Enable PM2 to Start on Boot

```bash
pm2 startup
```

Follow the instructions printed in the terminal to complete the setup.

### Step 23: Install Certbot (For HTTPS)

For production environments, securing your application with HTTPS is crucial.

```bash
sudo apt install certbot python3-certbot-nginx
```

### Step 24: Obtain an SSL Certificate

```bash
sudo certbot --nginx -d <your_domain>
```

### Step 25: Verify Automatic Renewal

```bash
sudo certbot renew --dry-run
```

---
## Configuring AWS CLI for DigitalOcean Spaces or Ubuntu Server

### Step 1: Install the AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install
```

### Step 2: Basic Configuration

```bash
aws configure
```

When prompted, enter the following values:
- **AWS Access Key ID**: `<your_access_key>`  
- **AWS Secret Access Key**: `<your_secret_key>`  
- **Default region name**: `<your_region>` (e.g., `sgp1` for Singapore)  
- **Default output format**: `json`  

---

## Continuous Deployment Workflow

### Step 1: Set Up an SSH Key in GitHub and on the Server

1. Go to your GitHub project → **Settings** → **Secrets and variables** → **Actions** → **Repository secrets** → **New repository secret**.
2. Add a **Name** and **Secret** (your private SSH key).
3. Ensure the corresponding **public key** is added to the Droplet/Server.

To add the SSH public key manually on the server:

```bash
mkdir -p ~/.ssh
nano ~/.ssh/authorized_keys
```

Paste your public key on a new line, then save.

Adjust permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chown -R $USER:$USER ~/.ssh
```

---

### Step 2: Write a GitHub Action

1. Go to **Actions** in your GitHub repository.
2. Click on **Set up a workflow yourself**.
3. Add the following sample workflow (update `ip_address`, `ssh key name`, and project details as needed):


```yaml
name: Deploy to DigitalOcean

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Setup SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519
          ssh-keyscan -H ip_address >> ~/.ssh/known_hosts

      - name: Deploy via SSH
        run: |
          ssh root@ip_address << 'EOF'
          set -e  # Stop immediately if any command fails
          cd /root/project-directory
          git fetch origin main
          git reset --hard origin/main
          npm install
          npm run deploy
          pm2 restart 0
          EOF
```

---

### Step 3: Commit the Workflow

Commit and push the workflow file to the repository.  
Whenever changes are pushed to the `main` branch, the application will be automatically deployed to the server.
