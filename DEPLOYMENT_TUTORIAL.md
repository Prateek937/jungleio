# Deploying a Static Website to EC2 with GitHub Actions — Step-by-Step Tutorial

This guide walks you through setting up automatic deployment of your static website
to an AWS EC2 instance using Nginx. Every push to the `main` branch will automatically
deploy your latest code.

---

## How It Works

```
You push to main
      │
      ▼
GitHub Actions triggers
      │
      ▼
Workflow SSHes into your EC2 instance
      │
      ▼
Runs: git pull → copies files to /var/www/jungleio/
      │
      ▼
Nginx serves your updated static site ✅
```

---

## Prerequisites

- An AWS account
- A GitHub repository with this code
- Basic familiarity with the terminal

---

## Step 1 — Launch an EC2 Instance

1. Go to **AWS Console → EC2 → Launch Instance**
2. Choose **Ubuntu Server 22.04 LTS** (Free Tier eligible)
3. Choose instance type: `t2.micro` (Free Tier) or larger
4. **Key pair**: Create a new key pair → Download the `.pem` file → **keep it safe**
5. **Security Group** — add these inbound rules:

   | Type | Port | Source    |
   |------|------|-----------|
   | SSH  | 22   | 0.0.0.0/0 |
   | HTTP | 80   | 0.0.0.0/0 |
   | HTTPS | 443 | 0.0.0.0/0 |

6. Launch the instance and note its **Public IP address**

---

## Step 2 — Set Up the EC2 Instance

SSH into your instance from your local machine:

```bash
ssh -i /path/to/your-key.pem ubuntu@<YOUR_EC2_PUBLIC_IP>
```

### Install Nginx

```bash
sudo apt-get update
sudo apt-get install -y nginx git
sudo systemctl enable nginx
sudo systemctl start nginx
```

### Create the web root directory for your site

```bash
sudo mkdir -p /var/www/jungleio
sudo chown -R ubuntu:ubuntu /var/www/jungleio
```

### Configure Nginx to serve your site

```bash
sudo nano /etc/nginx/sites-available/jungleio
```

Paste the following:

```nginx
server {
    listen 80;
    server_name <YOUR_EC2_PUBLIC_IP>;   # or your domain name

    root /var/www/jungleio;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Enable the site and reload Nginx:

```bash
sudo ln -s /etc/nginx/sites-available/jungleio /etc/nginx/sites-enabled/
sudo nginx -t          # test config — should print "syntax is ok"
sudo systemctl reload nginx
```

### Clone your repository on EC2

```bash
cd ~
git clone https://github.com/<YOUR_GITHUB_USERNAME>/<YOUR_REPO_NAME>.git jungleio
```

### Copy files to the web root for the first time

```bash
sudo cp -r ~/jungleio/. /var/www/jungleio/
sudo chown -R www-data:www-data /var/www/jungleio/
```

Visit `http://<YOUR_EC2_PUBLIC_IP>` in your browser — your site should be live!

---

## Step 3 — Generate an SSH Key for GitHub Actions

GitHub Actions needs to SSH into your EC2. We'll create a dedicated key pair for this.

**On your EC2 instance**, run:

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github_actions -N ""
```

This creates:
- `~/.ssh/github_actions` → **private key** (goes into GitHub Secrets)
- `~/.ssh/github_actions.pub` → **public key** (goes into EC2's authorized_keys)

### Add the public key to EC2's authorized keys

```bash
cat ~/.ssh/github_actions.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### Copy the private key — you'll need it in the next step

```bash
cat ~/.ssh/github_actions
```

Copy the entire output (including `-----BEGIN OPENSSH PRIVATE KEY-----` and `-----END OPENSSH PRIVATE KEY-----`).

---

## Step 4 — Allow Nginx File Copy Without Password (sudoers)

The deploy script runs `sudo cp` to copy files to `/var/www/`. To allow this without
a password prompt during the automated deploy, add a sudoers rule:

```bash
sudo visudo -f /etc/sudoers.d/github-actions
```

Add this line:

```
ubuntu ALL=(ALL) NOPASSWD: /bin/cp, /bin/chown
```

Save and exit.

---

## Step 5 — Add GitHub Secrets

Go to your GitHub repository → **Settings → Secrets and variables → Actions → New repository secret**

Add these three secrets:

| Secret Name    | Value                                          |
|----------------|------------------------------------------------|
| `EC2_HOST`     | Your EC2 public IP, e.g. `54.123.45.67`        |
| `EC2_USERNAME` | `ubuntu` (default for Ubuntu AMI)              |
| `EC2_SSH_KEY`  | The entire private key you copied in Step 3    |

> ⚠️ **Never** commit secrets or `.pem` files to your repository.

---

## Step 6 — Push to Main and Watch It Deploy

```bash
git add .
git commit -m "Set up GitHub Actions deployment"
git push origin main
```

Go to your GitHub repo → **Actions** tab.

You'll see the workflow running with real-time logs:

```
✓ Deploy to EC2 via SSH
  → Pulling latest changes from main...
  → Copying files to Nginx web root...
  → Deployment complete!
```

From now on, every push to `main` automatically deploys your site. 🎉

---

## Verifying the Deployment

```bash
# On EC2 — check Nginx is running
sudo systemctl status nginx

# Check your files are in the web root
ls /var/www/jungleio/

# Check Nginx logs if something looks wrong
sudo tail -f /var/log/nginx/error.log
sudo tail -f /var/log/nginx/access.log
```

---

## Troubleshooting

### ❌ Site not loading in browser
- Make sure port 80 is open in your EC2 Security Group
- Check Nginx is running: `sudo systemctl status nginx`
- Test config: `sudo nginx -t`

### ❌ SSH connection refused
- Check port 22 is open in your EC2 Security Group
- Verify `EC2_HOST` is the correct public IP (it changes on restart — use an Elastic IP to fix this)
- Confirm the public key is in `~/.ssh/authorized_keys` on EC2

### ❌ Permission denied on sudo cp
- Make sure you added the sudoers rule in Step 4

### ❌ git pull asks for credentials
- Switch the remote to SSH:
  ```bash
  git remote set-url origin git@github.com:<USER>/<REPO>.git
  ```

---

## Optional Improvements

### Use an Elastic IP (Recommended)
EC2 public IPs change when the instance restarts. Allocate an **Elastic IP** in the
AWS console so your IP never changes.

> AWS Console → EC2 → Elastic IPs → Allocate → Associate with instance

### Add a custom domain + HTTPS (free with Let's Encrypt)

```bash
sudo apt-get install -y certbot python3-certbot-nginx
sudo certbot --nginx -d yourdomain.com
```

Certbot auto-renews the certificate. Update `server_name` in your Nginx config to your domain first.

### Add a Slack/Discord notification on deploy
Append to the workflow's `script:` block:

```bash
curl -X POST -H 'Content-type: application/json' \
  --data '{"text":"✅ jungleio deployed successfully!"}' \
  YOUR_WEBHOOK_URL
```

---

## File Structure

```
.github/
  workflows/
    deploy.yml            ← GitHub Actions workflow (auto-deploy on push to main)
DEPLOYMENT_TUTORIAL.md    ← this file
index.html                ← your static site entry point
```
