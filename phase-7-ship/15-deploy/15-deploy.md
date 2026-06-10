# Deploy — A Real URL on the Internet

Everything you've built lives on your laptop. This chapter changes that. By the end, EduFlow has a public URL, HTTPS, secrets in environment variables (not in Git), a process manager that restarts on crash, and Nginx sitting in front of Node.js.

This is the part most bootcamps skip. Don't. Deploying something yourself — watching it fail, fixing it, watching it work — teaches you more about how production systems behave than any amount of local development.

---

## The production stack

| Component | Role |
|---|---|
| VPS (a small cloud server) | The machine your code runs on |
| Ubuntu 24.04 | The operating system |
| Node.js (via nvm) | Your application runtime |
| PM2 | Process manager — keeps Node running, restarts on crash, logs output |
| Nginx | Reverse proxy — receives HTTPS traffic, forwards to Node on port 3000 |
| Let's Encrypt / Certbot | Free, auto-renewing TLS certificates |
| MongoDB | Running on the same VPS (small projects) or a managed DB (more reliable) |
| Redis | Running on the same VPS |

For a production system at real scale you'd separate the database to a managed service (MongoDB Atlas) and run Redis on ElastiCache or similar. For this project, same-VPS is simpler and teaches the fundamentals — upgrade when you need it.

---

## Provision the VPS

Choose a provider: DigitalOcean Droplet, Hetzner Cloud, or AWS Lightsail are all fine. A 2 vCPU / 4GB RAM instance is enough.

> **Worth reading:** Search "Linux VPS setup guide Ubuntu" — DigitalOcean's "Initial Server Setup with Ubuntu" guide is the standard reference. It covers creating a non-root user, setting up SSH key authentication, and configuring UFW (the firewall). Read it first.

Steps (follow the DigitalOcean guide for these, not this chapter):

1. Create the VPS with your SSH public key
2. SSH in as root, create a deploy user, disable root SSH login
3. Configure UFW: allow SSH (22), HTTP (80), HTTPS (443); deny everything else
4. Set the hostname

---

## Install the runtime

SSH into the VPS as your deploy user:

```bash
# Install nvm (Node Version Manager)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc

# Install Node (use the same major version as your local machine)
nvm install 20
nvm use 20

# Install PM2 globally
npm install -g pm2
```

---

## Clone and configure the application

```bash
# On the VPS
git clone https://github.com/yourusername/eduflow.git
cd eduflow
npm install --production
```

Create `.env` on the VPS with production values. This is the one time you manually create this file — on the server, not in Git. Use `nano .env` and paste the production values. Production values differ from development:

- `NODE_ENV=production`
- A production `DATABASE_URL` pointing to the local or managed MongoDB
- New, long, random `JWT_SECRET` and `JWT_REFRESH_SECRET` (generate with `openssl rand -base64 64`)
- Your real Cloudinary, Resend, and Stripe keys
- `FRONTEND_URL` pointing to your frontend's production domain

Run database migrations:

```bash
npx prisma migrate deploy
npx prisma db seed  # optional — seed categories and an admin user
```

---

## PM2 process management

Create `ecosystem.config.js` in the project root:

```js
module.exports = {
  apps: [{
    name: 'eduflow-api',
    script: 'src/server.js',
    instances: 2,            // 2 processes for a 2-core VPS
    exec_mode: 'cluster',    // Node cluster mode — share the port
    env_production: {
      NODE_ENV: 'production'
    },
    error_file: 'logs/err.log',
    out_file: 'logs/out.log',
    log_date_format: 'YYYY-MM-DD HH:mm:ss'
  }]
};
```

Start the application:

```bash
pm2 start ecosystem.config.js --env production
pm2 save                  # save process list — survives server restarts
pm2 startup               # generates a systemd startup command — run the output command
```

Verify it's running:

```bash
pm2 list
pm2 logs eduflow-api
```

The application is now running on port 3000. You can't reach it yet — that's what Nginx is for.

---

## Nginx reverse proxy

Install Nginx:

```bash
sudo apt install nginx
```

Create `/etc/nginx/sites-available/eduflow`:

```nginx
server {
    listen 80;
    server_name api.yourdomain.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Enable and test:

```bash
sudo ln -s /etc/nginx/sites-available/eduflow /etc/nginx/sites-enabled/
sudo nginx -t       # test the config — must say "syntax is ok"
sudo systemctl reload nginx
```

Point `api.yourdomain.com` DNS A record to your VPS IP. Wait for propagation (a few minutes to an hour).

---

## HTTPS with Let's Encrypt

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d api.yourdomain.com
```

Certbot automatically modifies your Nginx config to add HTTPS and set up auto-renewal. Verify:

```bash
curl https://api.yourdomain.com/health
# Should return: {"status": "ok"}
```

---

## Smoke tests in production

Run every endpoint category once against the production URL:

```bash
# Health
curl https://api.yourdomain.com/health

# Register + verify + login
curl -X POST https://api.yourdomain.com/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email": "smoke@test.com", "password": "SmokePa55!"}'

# Catalogue
curl https://api.yourdomain.com/api/courses
```

Check PM2 logs for any errors:

```bash
pm2 logs eduflow-api --lines 50
```

---

## Definition of Done

- [ ] `https://api.yourdomain.com/health` returns `{"status": "ok"}` from a public network
- [ ] The TLS certificate is valid (browser shows the lock icon; no warnings)
- [ ] `pm2 list` shows the application running in cluster mode
- [ ] The `.env` file is not in Git — secrets are only on the server
- [ ] `pm2 startup` has been run — the application restarts automatically on server reboot (test: `sudo reboot`, wait, SSH back in, `pm2 list`)
- [ ] Database migrations ran successfully (`npx prisma migrate status` shows no pending migrations)
- [ ] Nginx access logs (`sudo tail -f /var/log/nginx/access.log`) show incoming requests

Write in `learning-log/15-deploy.md`:

1. Why does Node.js run behind Nginx rather than listening directly on port 80/443?
2. What does PM2's cluster mode do, and how does it relate to Node.js being single-threaded?
3. Why is `npx prisma migrate deploy` used in production instead of `npx prisma migrate dev`?
