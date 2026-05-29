## 1. CPU Hike on Linux Server

Problem: Server is slow, CPU ~90–100%.
Likely causes: Too many requests, bad code (infinite loop, heavy query), or a background job. 

What to check (commands): 
- `uptime` → check load average. 
- `top` or `htop` → see which process uses most CPU. 
- `ps aux --sort=-%cpu | head` → confirm top offenders. 
- Check logs for hot service (examples): 
  - Nginx: `tail -n 100 /var/log/nginx/access.log` 
  - Apache: `tail -n 100 /var/log/apache2/access.log` 

What can help fix / mitigate: 
- Restart bad service (temporary): `systemctl restart <service>` 
- Block abusive IPs: `ufw deny from <ip>` or nginx `deny <ip>;` 
- Scale: add more app instances or move to bigger machine (cloud action). 
- Long term: devs fix heavy code / queries (no direct command, but you give them the logs and URLs).

***

## 2. Memory Leak / Out-of-Memory (OOM)

Problem: Apps crash or get OOMKilled, server swaps heavily. 
Likely causes: App leaking memory, too many connections, huge responses. 

What to check (commands): 
- `free -h` → current RAM and swap usage. 
- `top` / `htop` → sort by memory (press `M` in `top`) to see top memory processes. 
- For containers/K8s: `kubectl describe pod <pod>` → look for `OOMKilled` events. 

What can help fix / mitigate: 
- Restart leaking service: `systemctl restart <service>` or `kubectl rollout restart deploy/<deploy>` 
- Kill stuck process: `kill -9 <pid>` (last resort, careful). 
- Reduce memory usage: lower cache sizes in config, lower concurrency (app setting). 
- Increase resources: edit K8s resources or give bigger VM. 
- Long term: devs fix memory leak after profiling.

***

## 3. Disk Full on Server

Problem: “No space left on device”, logs stop, services fail. 
Likely causes: Logs not rotated, big temp files, core dumps, backups on same disk. 

What to check (commands): 
- `df -h` → disk usage per filesystem. 
- `du -sh /* 2>/dev/null | sort -h` → see big top-level dirs. 
- Drill down: `du -sh /var/* 2>/dev/null | sort -h` 

What can help fix / mitigate: 
- Remove old logs: `rm -f /var/log/*.gz` or archive/move them. 
- Clean temp: `rm -rf /tmp/*` (careful, only if safe). 
- Set up logrotate: edit `/etc/logrotate.d/` configs. 
- Move large files to another disk: `mv largefile /mnt/bigdisk/`.

***

## 4. App Down, Port Not Listening

Problem: Web/app not reachable; port not open. 
Likely causes: Service crashed, wrong bind address, firewall blocking. 

What to check (commands): 
- `systemctl status <service>` → running or failed ?
- `journalctl -u <service> -n 50` → recent logs. 
- `ss -tuln | grep <port>` (or `netstat -tuln | grep <port>`) → is port listening?
- `curl -v http://localhost:<port>` → test locally. 

What can help fix / mitigate: 
- Start/restart service: `systemctl start <service>` or `systemctl restart <service>` 
- Fix bind address in config (e.g., listen on `0.0.0.0` if needed) and reload service. 
- Open firewall: 
  - `ufw allow <port>` 
  - or adjust security groups in cloud.

***

## 5. Slow Network / High Latency

Problem: App is slow; responses take a long time. 
Likely causes: Network congestion, bad route, slow upstream API, overloaded LB. 

What to check (commands): 
- `ping <host>` → basic latency and packet loss. 
- `traceroute <host>` or `mtr <host>` → see hops and where latency jumps. 
- `curl -w "@-" -o /dev/null -s <url>` (with a timing template) → measure response time from the server. 
# Example curl template cmd:-
  ```
 curl -w '\n\
time_namelookup:  %{time_namelookup}\n\
time_connect:     %{time_connect}\n\
time_starttransfer (TTFB): %{time_starttransfer}\n\
time_total:       %{time_total}\n\n' \
-o /dev/null -s https://example.com ```


What can help fix / mitigate: 
- If network path bad: escalate to network/ISP team with ping/traceroute outputs. 
- If upstream API is slow: add caching or timeouts/retries in app. 
- Scale out API/LB if they’re overloaded (increase replicas, nodes).

***

## 6. Database Connection Errors

Problem: App returns 500s, “cannot connect to DB” or “too many connections”. 
Likely causes: DB down, max connections exceeded, slow queries causing pile-up. 

What to check (commands): 
- `systemctl status mysql` or `systemctl status postgresql` → DB running?
- From app server: 
  - MySQL: `mysql -h <host> -u <user> -p` 
  - PostgreSQL: `psql -h <host> -U <user> <db>` 
- Check DB logs: 
  - MySQL: `tail -n 50 /var/log/mysql/error.log` 
  - Postgres: `tail -n 50 /var/log/postgresql/*.log` 

What can help fix / mitigate: 
- Start or restart DB: `systemctl start mysql` / `postgresql`. 
- Increase `max_connections` (with care) and restart DB. 
- Kill long-running queries (inside DB client using `SHOW PROCESSLIST` or `pg_stat_activity`). 
- Add connection pooling in app (config change) to avoid connection storms.

***

## 7. CI/CD Pipeline Failing

Problem: Jenkins/GitLab/GitHub Actions pipeline failing; deploy blocked. 
Likely causes: Config error, missing env/secret, failing tests, wrong branch. 

What to check (commands/actions): 
- Open pipeline UI → check full logs at failing stage. 
- On build agent/server (if SSH available): 
  - `df -h` → disk space. 
  - `docker info` → if build uses Docker, check daemon is running. 
  - `env` → verify environment variables. 

What can help fix / mitigate: 
- Fix script/command in Jenkinsfile/.gitlab-ci.yml/workflow. 
- Add or correct secrets/credentials in CI tool (no command, UI). 
- Re-run failed job after fix. 
- If agent broken: restart agent service or recreate runner node.

***

## 8. Container Won’t Start (Docker / Kubernetes)

Problem: Docker container exits, or K8s pod CrashLoopBackOff/ImagePullBackOff. 
Likely causes: Wrong image/tag, bad command, missing env variables, resource limits. 

What to check (commands): 
- Docker: 
  - `docker ps -a` → see container status. 
  - `docker logs <container>` → error message. 
- Kubernetes: 
  - `kubectl get pods` 
  - `kubectl describe pod <pod>` → events like `ImagePullBackOff`, `CrashLoopBackOff`. 
  - `kubectl logs <pod> -c <container>` → app logs. 

What can help fix / mitigate: 
- Fix image name/tag and redeploy. 
- Fix command/entrypoint/env vars in Deployment YAML or Docker run command. 
- Adjust resource requests/limits if pods OOMKilled. 
- For ImagePullBackOff in private registry: fix imagePullSecrets.

***

## 9. Config Drift / Works on One Server, Not on Another

Problem: Same app, one node OK, another node broken. 
Likely causes: Manual edits, different versions, missing config, different env vars. 

What to check (commands): 
- Compare config files: 
  - `diff /etc/app/config.conf server1 server2` (or scp + diff). 
- Compare env vars: 
  - `env | sort` on both systems. 
- Compare package versions: 
  - Debian: `dpkg -l | grep <package>` 
  - RHEL: `rpm -qa | grep <package>` 

What can help fix / mitigate: 
- Make configs identical (copy good config to bad server). 
- Install missing packages or align versions. 
- Move to using Ansible/Terraform/Kubernetes to avoid manual differences.

***

## 10. Git / Code Version Problems

Problem: Wrong code deployed, merge issues, or build uses wrong branch. 
Likely causes: Pipeline pointed to wrong branch, bad merge, force push. 

What to check (commands): 
- On server or CI workspace: 
  - `git status` → clean or dirty. 
  - `git branch` → which branch. 
  - `git log -5 --oneline` → last 5 commits deployed. 

What can help fix / mitigate: 
- Checkout correct branch: `git checkout main` or `git checkout release-x`. 
- Reset to known good commit: `git reset --hard <commit-id>` (careful). 
- Fix pipeline config to use correct branch/tag. 
- Use proper Git flow: PRs, reviews, no force-push to main.


