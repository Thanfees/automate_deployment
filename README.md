# Automate Deployment Webhook

A lightweight, secure, and fast automated deployment webhook server built with Python and Flask. This project allows you to seamlessly trigger custom bash deployment scripts on your server whenever code is pushed to specific branches in your GitHub repositories.

## Why use this instead of Jenkins for Startups?

When running a startup with a small-sized server (e.g., 1GB - 2GB RAM instances), Jenkins can be massive overkill. Here are the benefits of using this lightweight webhook listener instead of a heavy CI/CD tool like Jenkins:

1. **Extremely Low Memory Footprint**: Jenkins is a Java-based application that typically requires at least 1GB of RAM just to run the master node. This Python/Flask app consumes minimal memory (~50MB), leaving your server's resources available for your actual production applications.
2. **Zero Maintenance Headaches**: Jenkins requires frequent plugin updates, JVM tuning, workspace cleaning, and security patching. This script is basically a "set and forget" solution. 
3. **Simplicity and Speed**: No complex UI, no XML configurations, and no specific pipeline language (Groovy) to learn. You just write standard bash scripts for deployment, configure a JSON mapping in Python, and you are done.
4. **Instant Execution**: The deployment starts the moment GitHub sends the webhook payload. There's no queuing system or polling delay weighing down your server's CPU.
5. **Secure**: Built-in validation of GitHub's HMAC SHA-256 signatures ensures that only authorized payloads originating from your GitHub repository can trigger the server scripts.

## How it Works

1. A developer pushes code to a GitHub branch (e.g., `main`).
2. GitHub sends a POST request with a payload and an encrypted signature to your server's webhook URL (e.g., `https://auto.yourdomain.com/webhook/my-project`).
3. This Flask app receives the request, securely verifies the GitHub signature using your `GITHUB_SECRET`, and identifies the project and branch.
4. It looks up the associated bash script for that branch in its configuration.
5. It safely executes the bash script, deploying your code automatically!

---

## Step-by-Step Beginner's Guide

### Prerequisites
- A Linux server (Ubuntu/Debian recommended).
- Python 3 installed on your server.
- [PM2](https://pm2.keymetrics.io/) installed globally via Node.js/npm (Recommended for keeping the application running in the background as a daemon).
- Nginx or another reverse proxy installed (to receive external internet traffic).

### Step 1: Clone the Repository
Clone this repository to your server:
```bash
git clone <your-repo-url>
cd automate_deployment
```

### Step 2: Install Dependencies
It is considered a Python best practice to use a virtual environment:
```bash
# Create a virtual environment named "venv"
python3 -m venv venv

# Activate the virtual environment
source venv/bin/activate

# Install the required Python packages
pip install -r requirements.txt
```

### Step 3: Configure your Environment
Create a `.env` file in the project's root directory:
```bash
nano .env
```
Inside the `.env` file, add a highly secure, random secret token:
```ini
GITHUB_SECRET=your_super_secret_random_token_here_12345
```

*(You will use this exact token in your GitHub Webhook configuration in Step 6).*

### Step 4: Map your Deployment Scripts
Open `automate.py` and map your webhook endpoints to the location of your Linux bash scripts under the `PROJECT_CONFIG` dictionary.

You can configure different scripts for different repositories and branches (e.g., `main` vs. `dev`). Make sure your `.sh` deployment scripts exist on your server and are executable (`chmod +x your_script.sh`).

Example mapping in `automate.py`:
```python
PROJECT_CONFIG = {
    # If URL is /webhook/ecommerce
    "ecommerce": {
        "main": "/home/ubuntu/scripts/ecommerce_prod.sh",
        "dev": "/home/ubuntu/scripts/ecommerce_dev.sh"
    },
    # If URL is /webhook/admin-panel
    "admin-panel": {
        "master": "/home/ubuntu/scripts/admin_prod.sh"
    }
}
```

### Step 5: Start the Server (Production Mode)
Run the application continuously in the background using Gunicorn and PM2, so it restarts automatically if the server reboots or crashes:

```bash
# Make sure your virtual environment is still activated
# This runs the app with 4 background workers on port 7070
pm2 start "gunicorn -w 4 -b 0.0.0.0:7070 --timeout 600 automate:app" --name auto-deploy-bot

# Save the PM2 process list and configure it to start on server boot
pm2 save
pm2 startup
```

Your webhook listener is now internally running on `http://127.0.0.1:7070`.

### Step 6: Expose the Webhook using Nginx (Recommended)
You should use a reverse proxy to expose internal port 7070 to the internet over HTTPS (Port 443).

Create/Add an Nginx configuration snippet for your domain:
```nginx
server {
    server_name auto.yourdomain.com;

    location /webhook/ {
        proxy_pass http://127.0.0.1:7070;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```
*Note: Make sure to secure this domain with an SSL Certificate (e.g., using Let's Encrypt Certbot) so GitHub can send payloads securely.*

### Step 7: Connect to GitHub
1. Go to your repository on **GitHub**.
2. Navigate to **Settings** > **Webhooks** > **Add webhook**.
3. **Payload URL**: Enter your exposed URL, ending with the project key. E.g., `https://auto.yourdomain.com/webhook/ecommerce` (Make sure `ecommerce` matches your key in `PROJECT_CONFIG`).
4. **Content type**: Set to `application/json`.
5. **Secret**: Enter the exact same secret token you put in your server's `.env` file.
6. **Which events**: Select **"Just the push event."**
7. Click **Add webhook**.

That's it! Next time a developer pushes code to the configured branch, GitHub will notify your webhook server, and your deployment script will run instantly.

---

## Troubleshooting

- **Payload ignores push / HTTP 200 "No deployment script configured" / HTTP 200 "Not a branch push":** The push event payload from GitHub is valid, but the branch pushed (e.g., `feature-test`) is not listed in your `PROJECT_CONFIG`.
- **Security verification failed (HTTP 403):** The secret in your GitHub Webhook settings does not exactly match the `GITHUB_SECRET` in your `.env` file.
- **Project is not configured (HTTP 404):** Make sure the URL endpoint path matches a primary key in the `PROJECT_CONFIG` dictionary.
- **Script missing on server (HTTP 500):** Verify that the path to the bash script in `PROJECT_CONFIG` is absolute, spelled correctly, and the file actually exists on the server instance.
