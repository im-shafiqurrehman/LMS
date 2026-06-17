# Deploy

A running app on your laptop is not a product. A running app at a real URL, with HTTPS, that survives a server restart, is.

---

## What you are deploying

- Express backend (Node.js process)
- Next.js frontend (Node.js process)
- MongoDB Atlas (managed cloud database — no need to self-host)
- Redis (self-hosted on the VPS, or use Upstash for a managed option)

---

## The VPS setup

You need a Linux VPS. Popular cheap options: DigitalOcean Droplets, Hetzner CX11, Vultr. A 2GB RAM, 1 vCPU instance is sufficient for EduFlow at launch.

**Step 1 — SSH into your server**

```
ssh root@YOUR_SERVER_IP
```

Create a non-root user and disable root SSH login before anything else. Search "secure new VPS setup" for the standard hardening checklist.

**Step 2 — Install Node.js and PM2**

```
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
npm install -g pm2
```

**Step 3 — Install Nginx**

```
sudo apt install -y nginx
sudo systemctl enable nginx
```

---

## Environment variables in production

Never commit `.env` to Git. On the server, create the `.env` file by hand:

```
sudo nano /home/eduflow/backend/.env
```

Fill in production values. Your `JWT_SECRET` must be a cryptographically random string, not the development placeholder. Generate one:

```
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"
```

---

## Start the app with PM2

PM2 is a process manager that keeps your Node app running after SSH disconnects and restarts it if it crashes.

```
cd /home/eduflow/backend
pm2 start src/server.js --name eduflow-backend
pm2 save
pm2 startup
```

For the frontend:

```
cd /home/eduflow/frontend
npm run build
pm2 start npm --name eduflow-frontend -- start
pm2 save
```

---

## Nginx reverse proxy

Nginx sits in front of your Node processes and handles HTTPS termination. Create `/etc/nginx/sites-available/eduflow`:

```nginx
server {
    server_name yourdomain.com www.yourdomain.com;

    location /api {
        proxy_pass         http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
    }

    location / {
        proxy_pass         http://localhost:3001;
        proxy_http_version 1.1;
        proxy_set_header   Host $host;
    }
}
```

Enable it and test:

```
sudo ln -s /etc/nginx/sites-available/eduflow /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## HTTPS with Let's Encrypt

```
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Certbot modifies the Nginx config to add HTTPS and sets up automatic renewal.

---

## Definition of Done

- [ ] The backend is reachable at `https://yourdomain.com/api/health` and returns `{ "status": "ok" }`
- [ ] The frontend loads at `https://yourdomain.com`
- [ ] HTTPS is enabled; HTTP redirects to HTTPS
- [ ] `pm2 list` shows both processes running
- [ ] Restarting the server (`sudo reboot`) — both processes restart automatically via `pm2 startup`
- [ ] No secrets in `git log` — all sensitive values come from the server's `.env` file
