# PM2 Process Setup & Deployment Guide — Next.js

## 1. Prerequisites

Check Node.js and npm:

```bash
node -v
npm -v
```

For the WACRM project:

```text
Node.js: v20+
npm: v10+
```

Check PM2:

```bash
pm2 -v
```

If PM2 is not installed:

```bash
npm install -g pm2
```

---

# 2. Go to the Project

Example:

```bash
cd /var/www/html/wacrm
```

Always verify the project:

```bash
pwd
ls -lah
```

---

# 3. Install Dependencies

```bash
npm install
```

If `package-lock.json` is available and this is production:

```bash
npm ci
```

---

# 4. Configure Environment

For Next.js:

```bash
nano .env.local
```

Example:

```env
NEXT_PUBLIC_SITE_URL=https://wacrm.travash.com
```

Save:

```text
CTRL + O
ENTER
CTRL + X
```

---

# 5. Build the Next.js Application

```bash
npm run build
```

Make sure the build finishes successfully.

For a standalone Next.js application, check:

```bash
find .next/standalone -name server.js
```

Example output:

```text
.next/standalone/wacrm/server.js
```

**Important:** Do not assume the path is always:

```text
.next/standalone/server.js
```

The actual `server.js` location must be checked.

---

# 6. Test the Application Before PM2

First stop any old PM2 process if necessary:

```bash
pm2 stop wacrm
```

Then test the standalone server directly:

```bash
cd /var/www/html/wacrm
PORT=3000 node .next/standalone/wacrm/server.js
```

If successful, you should see something similar to:

```text
Ready in ...
Local: http://localhost:3000
```

Open another SSH terminal and test:

```bash
curl -I http://127.0.0.1:3000
```

If you get:

```text
HTTP/1.1 200
```

or a valid redirect such as:

```text
HTTP/1.1 307
```

the Next.js application is running.

Stop the foreground process with:

```text
CTRL + C
```

---

# 7. Start Next.js Using PM2

For WACRM:

```bash
cd /var/www/html/wacrm
pm2 start .next/standalone/wacrm/server.js \
  --name wacrm \
  --cwd /var/www/html/wacrm
```

Check:

```bash
pm2 status
```

Expected:

```text
wacrm    online
```

---

# 8. Check PM2 Logs

If the process becomes `errored` or keeps restarting:

```bash
pm2 logs wacrm --lines 50
```

Also check:

```bash
pm2 describe wacrm
```

Do **not** repeatedly run `pm2 start` when the application is crashing.

First check:

```bash
pm2 logs wacrm --lines 50
```

---

# 9. Check Application on Port 3000

```bash
curl -I http://127.0.0.1:3000
```

Check which process is using port 3000:

```bash
sudo ss -lntp | grep :3000
```

---

# 10. Save PM2 Process

Once the application is working:

```bash
pm2 save
```

Check:

```bash
pm2 status
```

---

# 11. Enable PM2 After Server Reboot

Run:

```bash
pm2 startup
```

PM2 will display a command similar to:

```text
sudo env PATH=$PATH:/usr/bin pm2 startup systemd -u root --hp /root
```

**Copy and execute the exact command PM2 gives you.**

Then:

```bash
pm2 save
```

Verify:

```bash
pm2 status
```

---

# 12. Useful PM2 Commands

### Check processes

```bash
pm2 status
```

### Start

```bash
pm2 start <server-file> --name <app-name>
```

### Restart

```bash
pm2 restart wacrm
```

### Stop

```bash
pm2 stop wacrm
```

### Delete process

```bash
pm2 delete wacrm
```

### View logs

```bash
pm2 logs wacrm
```

### Last 50 log lines

```bash
pm2 logs wacrm --lines 50
```

### Application information

```bash
pm2 describe wacrm
```

### Monitor CPU/RAM

```bash
pm2 monit
```

### Save processes

```bash
pm2 save
```

---

# 13. After Code Deployment

Whenever new code is deployed:

```bash
cd /var/www/html/wacrm
```

Pull code:

```bash
git pull
```

Install dependencies if required:

```bash
npm ci
```

Build:

```bash
npm run build
```

Restart PM2:

```bash
pm2 restart wacrm
```

Check:

```bash
pm2 status
```

Check logs:

```bash
pm2 logs wacrm --lines 50
```

---

# 14. If Environment Variables Changed

After changing `.env.local`:

```bash
cd /var/www/html/wacrm
npm run build
pm2 restart wacrm
```

For Next.js, variables used during the build may require a **new build**, not just a PM2 restart.

---

# 15. Apache Reverse Proxy

For a domain such as:

```text
wacrm.travash.com
```

Apache can forward traffic to:

```text
http://127.0.0.1:3000
```

Example:

```apache
<VirtualHost *:80>
    ServerName wacrm.travash.com

    ProxyPreserveHost On
    ProxyPass / http://127.0.0.1:3000/
    ProxyPassReverse / http://127.0.0.1:3000/

    ErrorLog ${APACHE_LOG_DIR}/wacrm.travash.com-error.log
    CustomLog ${APACHE_LOG_DIR}/wacrm.travash.com-access.log combined
</VirtualHost>
```

Enable required modules:

```bash
sudo a2enmod proxy
sudo a2enmod proxy_http
```

Enable site:

```bash
sudo a2ensite wacrm.travash.com.conf
```

Check Apache configuration:

```bash
sudo apache2ctl configtest
```

Expected:

```text
Syntax OK
```

Prefer reload instead of restart when possible:

```bash
sudo systemctl reload apache2
```

---

# 16. SSL Certificate

After DNS and HTTP are working:

```bash
sudo certbot --apache -d wacrm.travash.com
```

Then test:

```bash
curl -I https://wacrm.travash.com
```

---

# 17. Standard Production Flow

For a new Next.js standalone application:

```text
1. Clone project
       ↓
2. Configure .env.local
       ↓
3. npm ci
       ↓
4. npm run build
       ↓
5. Find server.js
       ↓
6. Test with node
       ↓
7. Start with PM2
       ↓
8. pm2 status
       ↓
9. pm2 logs
       ↓
10. Configure Apache
       ↓
11. Configure SSL
       ↓
12. pm2 save
       ↓
13. pm2 startup
       ↓
14. pm2 save again
```

---

# 18. WACRM Production Details

Current WACRM deployment:

```text
Server:
216.48.185.204

Project:
 /var/www/html/wacrm

Domain:
https://wacrm.travash.com

Application:
Next.js

Node:
20+

Application port:
3000

PM2 process:
wacrm

Standalone server:
.next/standalone/wacrm/server.js

Reverse proxy:
Apache → 127.0.0.1:3000
```

### Most important PM2 command for WACRM

```bash
cd /var/www/html/wacrm

pm2 start .next/standalone/wacrm/server.js \
  --name wacrm \
  --cwd /var/www/html/wacrm
```

Then:

```bash
pm2 status
```

And:

```bash
pm2 save
```

---

# 19. Quick Troubleshooting

### PM2 says `errored`

Run:

```bash
pm2 logs wacrm --lines 50
```

### Port 3000 already in use

```bash
sudo ss -lntp | grep :3000
```

### Check standalone server manually

```bash
cd /var/www/html/wacrm
PORT=3000 node .next/standalone/wacrm/server.js
```

### PM2 process disappeared after reboot

```bash
pm2 startup
pm2 save
```

### Application works on localhost but not domain

Check:

```bash
curl -I http://127.0.0.1:3000
curl -I http://wacrm.travash.com
```

Then check Apache:

```bash
sudo apache2ctl configtest
```

And logs:

```bash
sudo tail -50 /var/log/apache2/wacrm.travash.com-error.log
```

### Check PM2 process details

```bash
pm2 describe wacrm
```

---

## Golden Rule

**Never guess the Next.js standalone `server.js` path.**

Always run:

```bash
find .next/standalone -name server.js
```

Then use the returned path with PM2.