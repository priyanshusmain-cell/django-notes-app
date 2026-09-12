# Django Notes App — AWS EC2 + Docker Compose Deployment Backup

## 1. Stack
- AWS EC2
- Ubuntu Linux
- Docker
- Docker Compose
- Django
- Gunicorn
- Nginx
- MySQL
- Docker Network
- GitHub

## 2. Architecture

User → Internet → AWS EC2 :80 → Nginx :80 → Django/Gunicorn :8000 → MySQL :3306

All services communicate through the Docker Compose network `notes-app`.

## 3. Connect to EC2

```bash
cd ~/projects/django-notes-app
ls
```

## 4. Verify Docker

```bash
docker --version
docker compose version
```

Use modern Compose syntax:

```bash
docker compose
```

## 5. Validate Compose

Always validate before starting:

```bash
docker compose config
```

## 6. Start / Stop / Rebuild

```bash
docker compose up -d --build
docker compose down
docker compose restart
docker compose ps
docker ps
```

For a fresh Nginx rebuild:

```bash
docker compose build --no-cache nginx
docker compose up -d
```

## 7. Important Compose Syntax

Service networks are lists:

```yaml
networks:
  - notes-app
```

Top-level networks are mappings:

```yaml
networks:
  notes-app:
```

Correct Django settings:

```yaml
ports:
  - "8000:8000"

depends_on:
  - db

healthcheck:
  test: ["CMD-SHELL", "curl -f http://localhost:8000/admin || exit 1"]
  interval: 10s
  timeout: 5s
  retries: 5
  start_period: 60s
```

Correct Gunicorn command:

```yaml
command: sh -c "python manage.py migrate --no-input && gunicorn notesapp.wsgi --bind 0.0.0.0:8000"
```

## 8. Nginx Configuration

`nginx/default.conf`:

```nginx
upstream django {
    server django:8000;
}

server {
    listen 80;

    server_name localhost;

    location / {
        proxy_pass http://django;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Important: use the Compose service name `django` for Docker DNS, not `django_cont`.

After changing Nginx config:

```bash
docker compose down
docker compose build --no-cache nginx
docker compose up -d
```

Check:

```bash
docker compose logs nginx --tail=50
docker exec nginx_cont cat /etc/nginx/conf.d/default.conf
```

## 9. Check Application

```bash
docker compose ps
curl http://localhost
```

Expected containers:

```text
db_cont       Up (healthy)
django_cont   Up (healthy)
nginx_cont    Up
```

The successful `curl http://localhost` response should contain Django HTML.

## 10. AWS Security Group

Allow inbound:

```text
HTTP
TCP
80
0.0.0.0/0
```

Then access:

```text
http://YOUR_EC2_PUBLIC_IP
```

## 11. Docker Networking

Check:

```bash
docker network ls
docker network inspect django-notes-app_notes-app
```

Service-to-service communication:

```text
nginx → django:8000
django → db:3306
```

## 12. MySQL Persistent Data

Compose uses:

```yaml
volumes:
  - ./mysql-data:/var/lib/mysql
```

The database files are stored on the EC2 filesystem in:

```text
~/projects/django-notes-app/mysql-data/
```

Do not delete this directory if the database data is needed.

## 13. Docker Build Permission Problem

A build initially failed because Docker tried to read MySQL files from `mysql-data/`.

Create `.dockerignore` containing at least:

```text
mysql-data/
.env
.git
__pycache__/
*.pyc
.venv/
venv/
```

This prevents MySQL data and secrets from entering the Docker build context.

## 14. Git Security

Do not push:

```text
.env
mysql-data/
```

A suitable `.gitignore` includes:

```gitignore
.env
mysql-data/
__pycache__/
*.pyc
.venv/
venv/
*.sqlite3
staticfiles/
media/
.vscode/
.idea/
.DS_Store
```

## 15. GitHub

Repository:

```text
https://github.com/priyanshusmain-cell/django-notes-app.git
```

Check remote:

```bash
git remote -v
```

Check status:

```bash
git status
```

Add and commit changes:

```bash
git add .
git commit -m "Configure Docker deployment with Nginx"
```

Push:

```bash
git push -u origin main
```

For HTTPS authentication, use GitHub username `priyanshusmain-cell` and a GitHub Personal Access Token instead of the normal GitHub password.

## 16. Common Troubleshooting

YAML error:

```bash
docker compose config
```

Nginx problem:

```bash
docker compose logs nginx --tail=50
```

Django problem:

```bash
docker compose logs django --tail=50
```

MySQL problem:

```bash
docker compose logs db --tail=50
```

Check all logs:

```bash
docker compose logs
```

Follow logs:

```bash
docker compose logs -f
```

Check containers:

```bash
docker compose ps
docker ps -a
```

## 17. EC2 Stop vs Terminate

### Stop
- EC2 compute stops.
- Docker containers stop.
- EBS storage remains.
- Project files remain.
- MySQL data on EBS remains.
- Start the instance again later.
- Because services use `restart: always`, containers can restart when Docker starts.

After restarting:

```bash
cd ~/projects/django-notes-app
docker compose ps
```

If needed:

```bash
docker compose up -d
```

### Terminate
Termination deletes the EC2 instance and can delete its root EBS volume depending on configuration.

Before termination:
- Make sure source code is pushed to GitHub.
- Back up any important database data.
- Confirm there are no other AWS resources that can continue charging.

## 18. MySQL Database Backup

If the database contains important data, create a dump before terminating the EC2:

```bash
docker exec db_cont mysqldump -uroot -proot test_db > backup.sql
```

Check:

```bash
ls -lh backup.sql
```

Do not publish `backup.sql` to a public GitHub repository because it may contain sensitive data.

## 19. Rebuild on a New EC2

Clone the project:

```bash
git clone https://github.com/priyanshusmain-cell/django-notes-app.git
cd django-notes-app
```

Create the required `.env` separately.

Validate:

```bash
docker compose config
```

Build and start:

```bash
docker compose up -d --build
```

Verify:

```bash
docker compose ps
curl http://localhost
```

## 20. Future Deployment Workflow

```text
Modify code
    ↓
Test
    ↓
git status
    ↓
git add .
    ↓
git commit
    ↓
git push
    ↓
SSH to EC2
    ↓
git pull
    ↓
docker compose config
    ↓
docker compose up -d --build
    ↓
docker compose ps
    ↓
Test application
```

## 21. Final Project Achievement

Successfully completed:

- Django application containerization
- MySQL containerization
- Gunicorn application server
- Nginx reverse proxy
- Docker Compose orchestration
- Docker networking
- Container health checks
- AWS EC2 deployment
- Debugging YAML and container issues
- GitHub source-code backup

Final flow:

```text
User
 ↓
Internet
 ↓
AWS EC2
 ↓
Nginx :80
 ↓
Django + Gunicorn :8000
 ↓
MySQL :3306
```

This document is a backup/reference guide for rebuilding and explaining the project.
