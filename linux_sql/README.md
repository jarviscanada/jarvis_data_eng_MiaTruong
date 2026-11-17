# Linux Cluster Monitoring Agent
## Introduction

This project is a monitoring agent for a cluster of Linux servers. This program will be installed on each of the servers to gather both hardware specifications and resource usage data and then persist that data into a psql instance. Each host runs two bash scripts: 
- host_info.sh: Records the machine's hardware specifications once.

- host_usage.sh: Captures server usage data. It is triggered by cron every minute.

All data is stored in a PostgreSQL database running in Docker so the infrastructure team can track resource usage across every node from one place.

System administrators use this project to check and plan capacity and detect overloaded servers.

Tech stack: Bash, Docker, PostgreSQL, cron, and Git/GitHub with feature branches for version control.

## Quick Start
```bash
# 1. Create and start the PostgreSQL container
./scripts/psql_docker.sh start|stop|create postgress password
./scripts/psql_docker.sh start|stop
 
# 2. Create the database and tables
psql -h localhost -U postgres -c "CREATE DATABASE host_agent;"
psql -h localhost -U postgres -d host_agent -f sql/ddl.sql
 
# 3. Register this machine's hardware specifications
./scripts/host_info.sh localhost 5432 host_agent postgres password
 
# 4. Record a usage snapshot
./scripts/host_usage.sh localhost 5432 host_agent postgres password
 
# 5. Schedule every minute
crontab -e
# add this line:
* * * * * bash /home/rocky/dev/jarvis_data_eng_MiaTruong/linux_sql/scripts/host_usage.sh localhost 5432 host_agent postgres password >> /tmp/host_usage.log 2>&1
```

## Implementation
I built the project in small steps, each on its own feature branch (feature/psql_docker, feature/rdbms_ddl, feature/monitoring_agent) and the then merged into develop when it was done.

Database: A  script sets up PostgreSQL in Docker. ddl.sql creates the two tables.

Agent: 2 scripts follow the same pattern: collect values with standard Linux tools, clean data, use INSERT statement, and send it to the datatebase with psql.


### Scripts
 
**`psql_docker.sh`**: manages the PostgreSQL container.
```bash
./scripts/psql_docker.sh create <db_username> <db_password>   # create container + volume
./scripts/psql_docker.sh start                                # start existing container
./scripts/psql_docker.sh stop                                 # stop container
```
 
**`host_info.sh`**: reads static hardware details from `lscpu` and `/proc/meminfo` (CPU count, architecture, model, clock speed, L2 cache, total memory) and inserts one row into `host_info`. The `hostname` column is unique, so running it twice on the same machine is rejected instead of creating duplicates.
```bash
./scripts/host_info.sh <psql_host> <psql_port> <db_name> <psql_user> <psql_password>
```
 
**`host_usage.sh`**: takes one `vmstat` snapshot (in MB) so that all values come from the same moment. The host's id is resolved inside the `INSERT` with a subquery on `hostname`, so the script never needs to know the numeric id.
```bash
./scripts/host_usage.sh <psql_host> <psql_port> <db_name> <psql_user> <psql_password>
```
  
**crontab**: runs `host_usage.sh` once a minute and add output to a log file.
```bash
* * * * * bash /path/to/linux_sql/scripts/host_usage.sh localhost 5432 host_agent postgres password >> /tmp/host_usage.log 2>&1
```

## Architecture
![Cluster architecture](./assets/architecture_v2.png)

### Database Modeling
 
**`host_info`**: one row per machine
 
| Column | Type | Description |
|---|---|---|
| `id` | SERIAL, primary key | Auto-generated host identifier |
| `hostname` | VARCHAR, unique | Fully qualified hostname |
| `cpu_number` | INT2 | Number of CPUs |
| `cpu_architecture` | VARCHAR | e.g. `x86_64` |
| `cpu_model` | VARCHAR | CPU model |
| `cpu_mhz` | FLOAT8 | Clock speed in MHz |
| `l2_cache` | INT4 | L2 cache size in kB |
| `timestamp` | TIMESTAMP | Current time (UTC) |
| `total_mem` | INT4 | Total RAM in kB |
 
**`host_usage`**: one row per host per minute
 
| Column | Type | Description |
|---|---|---|
| `timestamp` | TIMESTAMP | Time of the snapshot was taken (UTC) |
| `host_id` | SERIAL, foreign key → `host_info.id` |
| `memory_free` | INT4 | Free memory in MB |
| `cpu_idle` | INT2 | CPU idle time (%) |
| `cpu_kernel` | INT2 | CPU time spent in kernel mode (%) |
| `disk_io` | INT4 | Disk I/O operations currently in progress |
| `disk_available` | INT4 | Free space on the root filesystem in MB |
 
## Test
Each feature was tested manually on a Rocky Linux VM:
- **DDL:** ran `ddl.sql` against `host_agent` and confirmed both tables with `SELECT * FROM host_info (host_usage)`. Inserted sample rows to verify constraints.
- **`host_info.sh`:** returned `INSERT 0 1`, and the row matched `lscpu` output. A second run was correctly rejected by the unique constraint on `hostname`. Testing also showed that sample rows inserted with hard-coded ids had left the id sequence behind; resetting it with `setval` fixed that.
- **`host_usage.sh`:** before the host was registered, the insert failed on the `NOT NULL` `host_id`, which confirmed the subquery link works as intended. After registration it returned `INSERT 0 1`.
- **Cron:** checked `/tmp/host_usage.log` per minute and confirmed new rows in `host_usage`.
- **Debugging:** used `bash -x` to trace variable values and print the final SQL before it was sent.
  
## Deployment
- **Docker** hosts PSQL in an isolated container, with a named volume so data persists across restarts.
- **cron** runs the usage agent every minute.
- **GitHub** stores the source. Each feature was developed on its own `feature/*` branch and merged into `develop`, with `main` reserved for releases (will be reviewed by Senior Technical Instructor).
  
## Improvements
- **Handle hardware changes:** `host_info.sh` currently refuses to run twice. It could upsert (`ON CONFLICT ... DO UPDATE`) so that added RAM or CPUs are reflected automatically.
- **Portability:** column positions in `vmstat` differ between versions, so parsing by header name instead of fixed column number would make the agent safer across distributions.
 
