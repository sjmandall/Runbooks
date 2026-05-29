DR / OUTAGE HANDLING RUNBOOK –(DOCKER + NGINX + PHP/PIMCORE + MYSQL)
===========================================================================

Context: 
- OS: Ubuntu (or similar Linux)  
- Stack: Nginx + PHP-FPM / Pimcore app + MySQL DB, mostly in Docker  
- Goal: Clear steps + concrete commands for detection, debugging, and recovery.

Use this runbook during practice and real incidents.

======================================================================
0. MINDSET, SAFETY, AND QUICK TEMPLATE
======================================================================
- Stay calm. No random commands.
- Change ONE thing at a time, observe the result.
- Prefer reversible actions (restart, rollback) before destructive ones (data deletion, full rebuild).
- Log everything you do.

Scratch template for every incident (keep in a note):

- Incident name:
- Start time:
- Initial symptom:
- Impact (who is affected):
- Hypothesis:
- Actions taken (time + command + result):
- Resolution time:
- Root cause:
- Follow-up actions:

======================================================================
1. DETECTION & INITIAL TRIAGE (FIRST 5–10 MINUTES)
======================================================================
1.1 Confirm the outage from your laptop
---------------------------------------
From your machine:

# Replace with actual URL/IP
curl -v http://YOUR_SERVER_IP_OR_DOMAIN
curl -v http://YOUR_SERVER_IP_OR_DOMAIN/admin

If both fail or are very slow, proceed to server checks.

1.2 Check monitoring / dashboards (if any)
------------------------------------------
Look at:
- HTTP error rate
- Latency / response time
- CPU / RAM / Disk
- Alerts from monitoring tools

1.3 Define impact
-----------------
Answer:
- Is it complete outage or partial?
- Frontend only, or admin only, or DB only?
- Only internal team or customers too?

1.4 Start an incident log
-------------------------
Create a note / Slack channel and paste the scratch template.  
Update it with every step and command you run.

======================================================================
2. SERVER / VM BASIC HEALTH CHECKS
======================================================================
SSH into the server
-------------------
# From your laptop
ssh USER@YOUR_SERVER_IP

Check uptime, CPU, memory, disk
--------------------------------
uptime
top -o %CPU         # or: htop
free -m
df -h

If disk is 100% used or memory is exhausted, make a note: this is likely root cause or contributing factor.

======================================================================
3. DOCKER & PROCESS HEALTH CHECKS
======================================================================
3.1 Check Docker containers
---------------------------
cd /opt/pimcore     # or your app directory

sudo docker ps

Look for:
- pimcore_web
- pimcore_app
- pimcore_db
- boomi_atom / others you use

If some containers are restarting / exited:

sudo docker ps -a

3.2 Basic logs for each key service
-----------------------------------
# Nginx container (web)
sudo docker logs pimcore_web --tail=100

# App container (Pimcore PHP)
sudo docker logs pimcore_app --tail=100

# DB container (MySQL)
sudo docker logs pimcore_db --tail=100

Note down:
- Errors about “connection refused”, “File not found”, “cannot connect to database”, etc.

======================================================================
4. LAYERED TROUBLESHOOTING – NETWORK, WEB, APP, DB
======================================================================

4.1 Network & port reachability
-------------------------------
From your laptop:
ping YOUR_SERVER_IP

From server, check if web port is listening:
sudo ss -tlnp | grep ':80'

If nothing is listening on 80 → Nginx not running inside container (see 4.2).

4.2 Web layer – Nginx
---------------------
Check Nginx container health:

sudo docker ps | grep pimcore_web

If not running:
sudo docker start pimcore_web

If running but acting weird, restart:
sudo docker restart pimcore_web

From server:

# Test Nginx from inside container
sudo docker exec -it pimcore_web sh -lc '
nginx -t || echo "nginx config error"
'

# Test HTTP from inside server to Nginx
curl -v http://localhost:8050

If `curl` to localhost:8050 fails or returns Nginx 502/503, suspect app or upstream config.

4.3 App layer – Pimcore PHP container
-------------------------------------
Check container:

sudo docker ps | grep pimcore_app

If not running:
sudo docker start pimcore_app

If running but failing:
sudo docker logs pimcore_app --tail=100

Enter container and check app health:

sudo docker exec -it pimcore_app bash

cd /var/www/html

# Check Symfony/Pimcore
php bin/console about
php bin/console pimcore:system:requirements

# Look at logs
tail -n 50 var/log/dev.log 2>/dev/null
tail -n 50 var/log/prod.log 2>/dev/null

Exit container:
exit

If console commands fail with DB errors → go to DB section.

4.4 DB layer – MySQL
--------------------
Check container:

sudo docker ps | grep pimcore_db

If not running:
sudo docker start pimcore_db

If running, check logs:
sudo docker logs pimcore_db --tail=100

Enter DB and inspect:

sudo docker exec -it pimcore_db mysql -u pimcore -p
# password: pimcore123 (as per your compose)

# Inside MySQL:
SHOW DATABASES;
USE pimcore;
SHOW TABLES;
EXIT;

If cannot connect:
- Check MySQL env vars in docker-compose.
- Check disk space (df -h).
- Check if MySQL restarted frequently in logs.

======================================================================
5. COMMON SYMPTOMS AND QUICK ACTIONS
======================================================================

5.1 Symptom: No response / connection timeout
---------------------------------------------
curl -v http://YOUR_SERVER_IP_OR_DOMAIN # times out

Checklist:
- Is server reachable? (ping YOUR_SERVER_IP)
- Any firewall/Security Group changes?
- Is Docker running?
  - sudo systemctl status docker
- Are containers up?
  - sudo docker ps

Fix candidates:
- If Docker stopped:
  sudo systemctl start docker
  cd /opt/pimcore
  sudo docker compose up -d

- If server CPU/disk full:
  See Section 7 to clean disk / heavy processes.

5.2 Symptom: Nginx 502 / 503 / 504 (Bad gateway)
-------------------------------------------------
curl -v http://localhost:8050 shows 502/503

Checklist:
- Is `pimcore_app` running?
  sudo docker ps | grep pimcore_app

- In Nginx logs (inside web container):
  sudo docker logs pimcore_web --tail=100

Fix candidates:
- Restart app:
  sudo docker restart pimcore_app

- Verify Nginx upstream host:port:
  # View nginx.conf on host:
  cat /opt/pimcore/nginx.conf

  # It should have something like:
  # fastcgi_pass pimcore_app:9000;

If upstream name/port mismatch container name/port → fix config and restart web:

sudo docker compose restart pimcore_web

5.3 Symptom: App-level 500 errors / Pimcore errors
--------------------------------------------------
Checklist:
- Check app logs:
  sudo docker exec -it pimcore_app bash -lc '
    cd /var/www/html;
    tail -n 50 var/log/dev.log 2>/dev/null;
    tail -n 50 var/log/prod.log 2>/dev/null;
  '

- Check DB connectivity from app:
  # Inside app container:
  php -r 'var_dump(getenv("PIMCORE_INSTALL_MYSQL_HOST"));'

- Check recent deployments/config changes.

Fix candidates:
- Roll back to previous image / commit (your CI/CD process).
- Fix config (wrong env vars) and redeploy.
- Clear cache:
  sudo docker exec -it pimcore_app bash -lc 'cd /var/www/html && php bin/console cache:clear'

5.4 Symptom: “File not found.” from PHP
---------------------------------------
Typical cause: Nginx → PHP-FPM path mismatch.

Checklist:
- Both containers see same paths:

sudo docker exec -it pimcore_app bash -lc '
cd /var/www/html;
pwd;
ls;
ls public;
'

sudo docker exec -it pimcore_web sh -lc '
cd /var/www/html;
pwd;
ls;
ls public;
'

They should both have `/var/www/html/public/index.php`.

- Nginx config should be consistent:

cat /opt/pimcore/nginx.conf

It should roughly be:

server {
    listen 80;
    server_name _;

    root /var/www/html/public;
    index index.php;

    location / {
        try_files $uri /index.php$is_args$args;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass pimcore_app:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $request_filename;
        fastcgi_param DOCUMENT_ROOT /var/www/html/public;
        fastcgi_param SCRIPT_NAME $fastcgi_script_name;
    }
}

Fix:
- Ensure docker-compose uses same volume mapping:

# In docker-compose.yml
pimcore_app:
  volumes:
    - /opt/pimcore/project:/var/www/html

pimcore_web:
  volumes:
    - /opt/pimcore/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    - /opt/pimcore/project:/var/www/html:ro

- Restart stack:

cd /opt/pimcore
sudo docker compose down --remove-orphans
sudo docker compose up -d

======================================================================
6. SERVICE RECOVERY ACTIONS (SAFE → STRONGER)
======================================================================

6.1 Restart individual containers
---------------------------------
# Restart web
sudo docker restart pimcore_web

# Restart app
sudo docker restart pimcore_app

# Restart DB
sudo docker restart pimcore_db

6.2 Restart whole stack
-----------------------
cd /opt/pimcore
sudo docker compose down --remove-orphans
sudo docker compose up -d

6.3 Rebuild images (if needed)
------------------------------
# If Dockerfile changed and you need fresh images
sudo docker compose build
sudo docker compose up -d

======================================================================
7. RESOURCE & DISK PROBLEMS
======================================================================

7.1 Check disk
--------------
df -h

If any filesystem at 90–100%, start cleaning.

7.2 Find big directories/files
------------------------------
# Top level space usage
sudo du -h / | sort -hr | head -n 20

# Logs
sudo du -h /var/log | sort -hr | head -n 20

7.3 Clean logs (carefully)
--------------------------
# Compress logs:
sudo find /var/log -name "*.log" -type f -size +10M -exec gzip {} \;

# Delete very old rotated logs (example: older than 30 days)
sudo find /var/log -name "*.gz" -mtime +30 -delete

7.4 Docker disk usage
---------------------
sudo docker system df

# Remove unused images/containers (CAREFUL)
sudo docker system prune -f
# To also remove unused volumes:
sudo docker system prune -a --volumes

Verify:
df -h

======================================================================
8. DATABASE-SPECIFIC RECOVERY
======================================================================

8.1 Basic MySQL health
----------------------
# Container running?
sudo docker ps | grep pimcore_db

# Logs
sudo docker logs pimcore_db --tail=100

# Connect as app user
sudo docker exec -it pimcore_db mysql -u pimcore -p
# password: pimcore123

# Inside MySQL:
SHOW DATABASES;
USE pimcore;
SHOW TABLES;
EXIT;

8.2 Restart DB container
------------------------
sudo docker restart pimcore_db

8.3 If DB won’t start
---------------------
- Check logs for corruption or config error:

sudo docker logs pimcore_db --tail=100

- Validate env vars in docker-compose:
  - MYSQL_ROOT_PASSWORD
  - MYSQL_DATABASE
  - MYSQL_USER
  - MYSQL_PASSWORD

8.4 Restore from backup (practice)
----------------------------------
This depends on your backup method. Example (if you had mysqldump):

# Restore from dump (replace filename and DB/user/pass)
sudo docker exec -i pimcore_db mysql -u root -p pimcore < /path/to/pimcore_backup.sql

======================================================================
9. DISASTER RECOVERY – SERVER LOST OR UNRECOVERABLE
======================================================================

Use this if the VM is gone or extremely broken.

9.1 Provision new server (high level)
-------------------------------------
- Create new VM in cloud/on-prem with same OS.
- Open required ports (SSH, 80/443, etc.)

9.2 Install base tools
----------------------
# On new VM
sudo apt update
sudo apt install -y docker.io docker-compose-plugin git

# Enable Docker
sudo systemctl enable docker
sudo systemctl start docker

9.3 Deploy your stack from repo
-------------------------------
cd /opt
sudo mkdir pimcore
sudo chown $USER:$USER pimcore
cd pimcore

# Get your repo
git clone <YOUR_REPO_URL> .

# Check docker-compose.yml present
ls

# Bring up stack
sudo docker compose up -d

9.4 Restore data
----------------
- Copy / restore DB from backups.
- Restore file assets (if stored on disk or S3).

9.5 Switch traffic
------------------
- Update DNS to point to new server IP or
- Update load balancer target.

======================================================================
10. VALIDATION AFTER RECOVERY
======================================================================

10.1 Technical checks
---------------------
From server:
curl -v http://localhost:8050
curl -v http://localhost:8050/admin

From your laptop:
curl -v http://YOUR_SERVER_IP_OR_DOMAIN
curl -v http://YOUR_SERVER_IP_OR_DOMAIN/admin

Verify:
- Login works.
- Basic CRUD operations (create/read/update/delete) in app.
- No continuous errors in logs:
  - sudo docker logs pimcore_web --tail=50
  - sudo docker logs pimcore_app --tail=50
  - sudo docker logs pimcore_db --tail=50

10.2 Business checks
--------------------
- Ask stakeholders / test users:
  - Can they do their main tasks?
  - Any missing or inconsistent data?

======================================================================
11. POST-INCIDENT REVIEW (EVEN FOR PRACTICE)
======================================================================

After the system is stable, fill this out:

- Incident name:
- Timeline:
  - Start:
  - Detected:
  - Mitigated:
  - Resolved:
- Root cause:
- What helped:
- What slowed you down:
- Action items:
  - Monitoring to add/improve
  - Alerts to configure
  - Runbook updates
  - Automation opportunities


======================================================================
END OF RUNBOOK
======================================================================
