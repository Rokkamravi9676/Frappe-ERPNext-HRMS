# ERPNext + HRMS Deployment on Rocky Linux 8.10

> Production-grade ERP infrastructure using Docker Compose — by **Ravi Rokkam**

---

## 📋 Overview

Full deployment of **ERPNext v15** with **Frappe HRMS** on Rocky Linux 8.10 using Docker Compose. Includes a custom-built Docker image with HRMS pre-installed, MariaDB, Redis, Nginx, and all supporting services.

- **Server:** template.kmvpl.com (172.16.30.123)
- **OS:** Rocky Linux 8.10 (Green Obsidian)
- **Access URL:** http://172.16.30.123

---

## 🏗️ Architecture

| Container | Image | Role |
|---|---|---|
| psadmin-backend-1 | frappe/erpnext-hrms:version-15 | Gunicorn WSGI (port 8000) |
| psadmin-frontend-1 | frappe/erpnext-hrms:version-15 | Nginx reverse proxy (port 80) |
| psadmin-websocket-1 | frappe/erpnext-hrms:version-15 | Socket.IO (port 9000) |
| psadmin-queue-short-1 | frappe/erpnext-hrms:version-15 | Short job worker |
| psadmin-queue-long-1 | frappe/erpnext-hrms:version-15 | Long job worker |
| psadmin-scheduler-1 | frappe/erpnext-hrms:version-15 | Scheduled tasks |
| psadmin-db-1 | mariadb:10.6 | Database |
| psadmin-redis-cache-1 | redis:6.2-alpine | Cache |
| psadmin-redis-queue-1 | redis:6.2-alpine | Job queue |

---

## 🛠️ Tech Stack

- **OS:** Rocky Linux 8.10
- **Containerization:** Docker + Compose v2
- **ERP:** ERPNext v15 + Frappe HRMS 15.58.2
- **Database:** MariaDB 10.6
- **Cache/Queue:** Redis 6.2
- **Web Server:** Nginx 1.22
- **Runtime:** Python 3.10.14, Node.js 18.20.8
- **Bench CLI:** 5.29.1

---

## 📁 Project Structure

```
/home/psadmin/
├── Dockerfile              # Custom image with HRMS pre-installed
└── frappe-compose.yml      # Docker Compose configuration
```

### Dockerfile

```dockerfile
FROM frappe/erpnext:version-15
USER frappe
WORKDIR /home/frappe/frappe-bench
RUN bench get-app --skip-assets hrms --branch version-15
```

### common_site_config.json

```json
{
  "redis_cache": "redis://redis-cache:6379",
  "redis_queue": "redis://redis-queue:6379",
  "redis_socketio": "redis://redis-cache:6379",
  "socketio_port": 9000
}
```

---

## 🚀 Deployment Steps

### 1. Build Custom Image
```bash
cd /home/psadmin
sudo docker build -t frappe/erpnext-hrms:version-15 .
```

### 2. Start All Services
```bash
sudo docker compose -p psadmin -f /home/psadmin/frappe-compose.yml up -d
```

### 3. Verify All Containers
```bash
sudo docker compose -p psadmin ps
```

### 4. Install HRMS App on Site
```bash
sudo docker exec -it psadmin-backend-1 bench --site 172.16.30.123 install-app hrms
```

### 5. Run Migrations
```bash
sudo docker exec -it psadmin-backend-1 bench --site 172.16.30.123 migrate
```

---

## 🐛 Issues & Resolutions

### Issue 1 — Redis Connection Refused (Port 13311)
**Error:** `ConnectionError: Error 111 connecting to 127.0.0.1:13311`  
**Cause:** `common_site_config.json` was missing Redis URLs  
**Fix:** Manually wrote correct Redis config into shared sites volume

---

### Issue 2 — ModuleNotFoundError: No module named 'hrms'
**Error:** Worker containers crashing on startup  
**Cause:** Each container has its own isolated Python virtualenv. Installing via bench only updated backend's env, not workers  
**Fix:** Built custom Docker image with HRMS pre-installed via `bench get-app`

---

### Issue 3 — Bench Refusing to Run as Root
**Error:** `WARN: You should not run this command as root`  
**Cause:** Dockerfile had `USER root` at the end  
**Fix:** Removed `USER root`, kept `USER frappe` as final user

---

### Issue 4 — MariaDB Access Denied
**Error:** `Access denied for user '_dea3222a1145c189'@'172.20.0.10'`  
**Cause:** DB user locked to old container IP after rebuild  
**Fix:**
```sql
RENAME USER '_dea3222a1145c189'@'172.20.0.8' TO '_dea3222a1145c189'@'%';
GRANT ALL PRIVILEGES ON `_dea3222a1145c189`.* TO '_dea3222a1145c189'@'%';
FLUSH PRIVILEGES;
```

---

### Issue 5 — Site 404 When Accessing via IP
**Error:** nginx returning 404 for `http://172.16.30.123`  
**Cause:** `FRAPPE_SITE_NAME_HEADER` was set to `erp.kmvpl.com` but browser sent IP as Host header  
**Fix:**
```bash
cp -r sites/erp.kmvpl.com sites/172.16.30.123
# Updated compose: FRAPPE_SITE_NAME_HEADER: 172.16.30.123
```

---

### Issue 6 — Firewall Blocking Port 80
**Error:** External access timing out  
**Cause:** Port 80 not open in Rocky Linux firewalld  
**Fix:**
```bash
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --reload
```

---

## 🔧 Common Commands

```bash
# Start services
sudo docker compose -p psadmin -f /home/psadmin/frappe-compose.yml up -d

# Stop services
sudo docker compose -p psadmin -f /home/psadmin/frappe-compose.yml down

# Restart all
sudo docker compose -p psadmin restart

# View status
sudo docker compose -p psadmin ps

# View logs
sudo docker logs psadmin-backend-1 --tail 30

# Backup site
sudo docker exec -it psadmin-backend-1 bench --site 172.16.30.123 backup

# Migrate
sudo docker exec -it psadmin-backend-1 bench --site 172.16.30.123 migrate

# Rebuild image
sudo docker build -t frappe/erpnext-hrms:version-15 /home/psadmin/
```

---

## ⏭️ Pending

- [ ] Add GoDaddy DNS A record: `erp` → `103.44.0.2`
- [ ] Configure router port forwarding 80 → 172.16.30.123
- [ ] Set up SSL/HTTPS with Let's Encrypt
- [ ] Complete ERPNext setup wizard
- [ ] Configure HRMS modules (departments, payroll, leaves)
- [ ] Schedule automated backups
- [ ] Change default admin password

---

## 👤 Author

**Ravi Rokkam**  
📧 rokkamravi1999@gmail.com  
📞 +91 9676969045  
🔗 [LinkedIn](https://www.linkedin.com/in/ravi-rokkam-aa0b9a1b7)  
🐙 [GitHub](https://github.com/Rokkamravi9676)

---

*Deployed: March 03, 2026 | Rocky Linux 8.10 | ERPNext v15 + HRMS*
