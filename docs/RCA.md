# Root Cause Analysis (RCA)

## Incident Summary

* **Incident Title**: Django Application Unreachable on Port 8000 Post-Deployment
* **Date**: August 25, 2026
* **Affected Environment**: AWS EC2 (`t3.micro`, Ubuntu, Jenkins Agent)
* **Severity**: High (Service Unreachable)
* **Status**: Resolved

---

## Executive Summary

Following a Jenkins CI/CD pipeline run (clone, build, test, and deploy via Docker Compose), the Django application deployed on the Jenkins EC2 agent was inaccessible on port `8000`. Inspection revealed that while `docker ps` showed the containers as running, `django_cont` was stuck in a continuous crash/restart loop (`STATUS: Up 23 seconds (health: starting)`).

The primary root cause was an **Out-Of-Memory (OOM)** kernel kill of the MySQL database container (`db_cont`) due to memory constraints on the `t3.micro` EC2 instance (1 GB RAM, 0 B Swap). Provisioning a **2 GB Swap file**, clearing the corrupted database volume, and restarting the Docker Compose stack successfully restored service on ports `8000` and `80`.

---

## Timeline of Events

| Time (UTC) | Event |
| :--- | :--- |
| **13:43:11** | Jenkins pipeline executed `docker compose up -d`, initializing `nginx_cont`, `django_cont`, and `db_cont`. |
| **13:45:01** | `mysqld` process allocation exceeded available system RAM. Linux kernel invoked `oom-killer`, killing `mysqld` (PID 12772). Container `db_cont` exited with status `137`. |
| **13:45:02** | `django_cont` failed database initialization (`OperationalError: Unknown server host 'db_cont'`) and entered a continuous restart loop. |
| **19:24:15** | SSH diagnostic session initiated. Checked Docker logs and container status. |
| **19:26:44** | Inspected `dmesg -T`, confirming kernel OOM killer events (`Out of memory: Killed process 12772 (mysqld)`). |
| **19:26:53** | Provisioned and enabled 2 GB Swap space (`/swapfile`) on the EC2 instance. |
| **19:27:49** | Removed stale/partially-initialized database volume (`./data/mysql/db`) and restarted Docker Compose stack. |
| **19:28:54** | `db_cont` and `django_cont` status transitioned to `Up (healthy)`. Verified `HTTP/1.1 200 OK` on ports `8000` and `80`. |

---

## 5 Whys Analysis

1. **Why was the Django application unreachable on port 8000?**
   * *Answer*: `django_cont` was stuck in a restart loop and never completed starting Gunicorn to bind to port 8000.
2. **Why was `django_cont` stuck in a restart loop?**
   * *Answer*: Django database checks failed on startup with `django.db.utils.OperationalError: (2005, "Unknown server host 'db_cont'")`.
3. **Why was `db_cont` host unreachable?**
   * *Answer*: The MySQL container `db_cont` had crashed and exited with status `Exited (137)`.
4. **Why did `db_cont` exit with code 137?**
   * *Answer*: Code 137 indicates SIGKILL by the Linux Kernel Out-Of-Memory (OOM) killer.
5. **Why did the OOM killer terminate MySQL?**
   * *Answer*: The EC2 `t3.micro` instance has 1 GB total RAM and **0 B Swap space**, which was exhausted when running the Jenkins agent, system processes, Nginx, Django, and MySQL 9.x simultaneously.

---

## Root Cause Details

1. **Lack of Swap Memory**: The EC2 instance lacked a swap file. When `mysql:latest` attempted memory allocation during database initialization, system memory was depleted, triggering `oom-killer`.
2. **Cascading Failure**: Once `db_cont` was killed, Docker internal DNS could no longer resolve `db_cont`, causing `django_cont`'s startup command (`python manage.py migrate`) to crash continuously.
3. **Corrupted Volume State**: The initial OOM kill left `var/lib/mysql` in a partially initialized state, which subsequently prevented `root` remote authentication (`Host '172.18.0.3' is not allowed to connect to this MySQL server`).

---

## Corrective & Preventive Actions

### Actions Taken (Completed)
- [x] **Configured Swap Space**: Created and activated a 2 GB `/swapfile` on the EC2 host.
- [x] **Volume Reset**: Purged the corrupted database volume directory `./data/mysql/db`.
- [x] **Service Restoration**: Re-deployed containers via Docker Compose and verified HTTP 200 responses on ports 8000 and 80.

### Recommended Preventive Measures
1. **Make Swap Permanent**: Ensure `/swapfile swap swap defaults 0 0` is present in `/etc/fstab` on the EC2 agent.
2. **Pin MySQL Version & Limit Memory in `docker-compose.yml`**:
   * Change `image: mysql` to a explicit lower-footprint image (e.g., `mysql:8.0-oracle` or `mariadb:10.5`).
   * Add memory limits and MySQL startup flags (e.g. `--innodb-buffer-pool-size=64M`).
3. **Add Dependency Readiness Checks**: Use Docker Compose `depends_on` with `condition: service_healthy` so `django_app` waits for `db` to be fully ready before executing migrations.
