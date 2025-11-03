

# MySQL / MariaDB Cron Backup with Gotify Notifications

  

Automated backups of a remote **MySQL/MariaDB** database using [`fradelg/mysql-cron-backup`](https://github.com/fradelg/mysql-cron-backup) — with optional **Gotify** push notifications when new backup files are created.


<img width="1096" height="294" alt="afbeelding" src="https://github.com/user-attachments/assets/ed67fb49-50a1-4c35-8d1a-93163470149e" />


  

>  **Highlights**

>

>  - 🕒 Schedule backups with cron (daily, hourly, custom)

>  - 🗃️ Keeps only the most recent *N* backups

>  - 💾 Plain `.sql` or compressed `.sql.gz` dumps

>  - 🔔 Gotify notifications with file name, size and timestamp

>  - 🐳 One-command Docker Compose deployment

  

---

  

## 📦 Quick Start

  

1)  **Create files**

  

-  `docker-compose.yml` (see below)

-  `.env` (see example below)

  

2)  **Start the stack**

  

```bash

docker  compose  up  -d

```

  

3)  **Verify**

  

List generated backups on the host (default `./backups`):

  

```bash

ls  -lh  backups

```

  

If you use Gotify, you should receive a notification when the first backup is written.

  

---

  

## 🐳 docker-compose.yml

  

> Paste this as `docker-compose.yml`. The `gotify-notification` sidecar watches the backup folder and sends a message to Gotify whenever a new `.sql` or `.sql.gz` appears.

  

```yaml

services:
  mysql-cron-backup:
    image: fradelg/mysql-cron-backup:latest
    restart: unless-stopped
    environment:
      MYSQL_HOST: ${DB_HOST}
      MYSQL_PORT: ${DB_PORT}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASS: ${DB_PASS}
      CRON_TIME: "${CRON_TIME}"
      TZ: ${TZ}
      MAX_BACKUPS: ${MAX_BACKUPS}
      INIT_BACKUP: ${INIT_BACKUP}
      USE_PLAIN_SQL: ${USE_PLAIN_SQL}
      MYSQLDUMP_OPTS: "${MYSQLDUMP_OPTS}"
    volumes:
      - ${VOLUME_PATH}:/backup

  gotify-notification:
    image: alpine:3.20
    restart: unless-stopped
    depends_on:
      - mysql-cron-backup
    environment:
      GOTIFY_URL: ${GOTIFY_URL}
      GOTIFY_TOKEN: ${GOTIFY_TOKEN}
      MIN_SIZE_BYTES: ${MIN_SIZE_BYTES:-100}
      CURL_INSECURE: ${CURL_INSECURE:-0}
    volumes:
      - ${VOLUME_PATH}:/backup:ro
    entrypoint: >
      /bin/sh -lc '
        set -eu;
        apk add --no-cache curl inotify-tools bc tzdata >/dev/null;
        export TZ=Europe/Amsterdam;

        send_msg() {
          file="$$1"

          # Detect manual backup
          if echo "$$(basename "$$file")" | grep -q "^manual-"; then
            dbname="manual"
            manual="true"
          else
            # Extract DB name from filename, e.g. 20251103_1010.kuma.sql -> kuma
            dbname=$$(basename "$$file" | sed -E "s/.*[._]([A-Za-z0-9_-]+)\.sql.*/\1/")
            manual="false"
          fi

          # Ensure file is fully written
          sleep 0.5
          size_bytes=$$(wc -c < "$$file" || echo 0)
          size_mb=$$(echo "scale=2; $$size_bytes/1048576" | bc)
          ts=$$(date +"%d-%m-%y - %H:%M (%Z)")

          # Build title
          if [ "$${size_bytes:-0}" -gt "$${MIN_SIZE_BYTES:-100}" ]; then
            status="✅ SUCCESSFUL"; prio=5
          else
            status="❌ FAILED"; prio=8
          fi

          if [ "$$manual" = "true" ]; then
            title="Manual MySQL backup = $${status}"
          else
            title="MySQL backup of $${dbname} = $${status}"
          fi

          # Message body
          msg=$$(printf "📄 File: %s\n💾 Size: %s MB\n🕒 Time: %s" \
                        "$$(basename "$$file")" "$$size_mb" "$$ts")

          CURL_OPTS="-fsS"
          [ "$${CURL_INSECURE:-0}" = "1" ] && CURL_OPTS="$$CURL_OPTS -k"

          curl $$CURL_OPTS -X POST "$${GOTIFY_URL}/message?token=$${GOTIFY_TOKEN}" \
            -F "title=$$title" \
            -F "message=$$msg" \
            -F "priority=$$prio" || true
        }

        # Notify about latest backup on startup (optional)
        latest="$$(ls -1t /backup/*.sql /backup/*.sql.gz 2>/dev/null | head -n1 || true)"
        if [ -n "$${latest:-}" ]; then send_msg "$$latest"; fi

        # Only close_write to avoid duplicates
        inotifywait -m -e close_write --format "%w%f" /backup |
        while read -r f; do
          case "$$f" in
            *.sql|*.sql.gz) send_msg "$$f" ;;
          esac
        done
      '
networks: {}

```

  

---

  

## ⚙️ `.env` Example

  

> Save as `.env` in the same folder as your compose file. Adjust values to your environment.

  

```env

# === Database Connection =============================================

  

# Hostname or IP of your external MariaDB server

DB_HOST=<IP-ADDRESS>

  

# MariaDB TCP port

DB_PORT=3306

  

# Database user with read access for backups (avoid using root)

DB_USER=<USERNAME>

  

# Password for the database user

DB_PASS=<PASSWORD>

  

# === Backup Settings =================================================

  

# Local path on the host where backup files will be stored

VOLUME_PATH=./backups

  

# Keep this many most recent backups (older ones will be deleted)

MAX_BACKUPS=10

  

# Whether to create an initial backup immediately when the container starts

# 0 = No, 1 = Yes

INIT_BACKUP=1

  

# Store backups as plain .sql files instead of compressed .sql.gz

# 1 = plain SQL, 0 = gzip compressed

USE_PLAIN_SQL=1

  
  

# === Schedule & Locale ===============================================

  

# Cron schedule for automatic backups

# Example: "0 2 * * *" = every day at 02:00

CRON_TIME=0 2 * * *

  

# Timezone for the container (affects cron execution time)

TZ=Europe/Amsterdam

  
  

# === mysqldump Options ===============================================

  

# Recommended flags for reliable, transaction-safe dumps

MYSQLDUMP_OPTS=--no-tablespaces --single-transaction --quick --routines --events --triggers

  
  

# === Gotify notification =============================================

  

GOTIFY_URL=https://<GOTIFY-URL>

GOTIFY_TOKEN=<GOTIFY-TOKEN>

MIN_SIZE_BYTES=100

```

  

---

  

## 🔑 Environment Variables Reference

  

| Variable | Required | Default | Description |

|---|:--:|---|---|

| `DB_HOST` | ✅ | – | Host/IP of the MySQL/MariaDB server to back up |

| `DB_PORT` | ✅ | `3306` | TCP port of the DB server |

| `DB_USER` | ✅ | – | Database user for `mysqldump` (read-only is fine) |

| `DB_PASS` | ✅ | – | Password for `DB_USER` |

| `VOLUME_PATH` | ✅ | `./backups` | Host path to store backup files |

| `MAX_BACKUPS` | ✅ | `10` | Number of most recent backups to keep |

| `INIT_BACKUP` | ✅ | `1` | Run an initial backup on first start (`1` yes / `0` no) |

| `USE_PLAIN_SQL` | ✅ | `1` | `1` = plain `.sql`, `0` = `.sql.gz` compressed |

| `CRON_TIME` | ✅ | – | Cron schedule (e.g., `0 2 * * *`) |

| `TZ` | ✅ | `Europe/Amsterdam` | Timezone for cron and notification timestamps |

| `MYSQLDUMP_OPTS` | ✅ | See example | Extra flags for `mysqldump` |

| `GOTIFY_URL` | ⛔ if not using Gotify | – | Base URL of Gotify server |

| `GOTIFY_TOKEN` | ⛔ if not using Gotify | – | App token for Gotify |

| `MIN_SIZE_BYTES` | ⛔ | `100` | Minimum file size that counts as “success” |

| `CURL_INSECURE` | ⛔ | `0` | Set `1` to allow self‑signed TLS in Gotify sidecar |

  

>  **Note:** The `gotify-notification` sidecar is optional. If you don’t need notifications, you can remove that service and its variables.

  

---

  

## 🧪 Test & Verify

  

- Trigger an immediate manual backup by setting `INIT_BACKUP=1` and (re)starting the stack:

```bash

docker compose down

docker compose up -d

```

- Check container logs:

```bash

docker compose logs -f mysql-cron-backup

docker compose logs -f gotify-notification

```

- Confirm files exist:

```bash

ls -lh ./backups

```

  

---

  

## 🔁 Restore (from a backup)

  

Pick the newest file and restore to a target server:

  

```bash

# Plain .sql

mysql  -h <host> -P <port> -u <user> -p <database> < backups/2025-11-03_0200.prod.sql

  

# Gzip file (.sql.gz)

gunzip  -c  backups/2025-11-03_0200.prod.sql.gz | mysql  -h <host> -P <port> -u <user> -p <database>

```

  

> Always restore to a test database first to verify integrity and permissions.

  

---

  

## 🛡️ Security Tips

  

- Use a **least-privileged** DB user that can run `SELECT` and dump routines/events if needed.

- Store `.env` outside of version control (`.gitignore`) if it contains secrets.

- Restrict filesystem permissions on `./backups`.

- Consider encrypting backups at rest (e.g., with age/gnupg) if needed.

  

---

  

## ⚠️ Troubleshooting

  

-  **No backup files appear**

Check DB connectivity, credentials, and `MYSQL_HOST` reachability. Review `mysql-cron-backup` logs.

  

-  **Gotify messages not sent**

Verify `GOTIFY_URL` and `GOTIFY_TOKEN`. Set `CURL_INSECURE=1` only for testing with self-signed TLS.

  

-  **Only the latest N files kept**

Controlled by `MAX_BACKUPS`. Older files are removed by the backup container.

  

-  **Cron timing off**

Ensure `TZ` is set correctly (default: `Europe/Amsterdam`).

  

-  **Permission errors on `./backups`**

Ensure the host directory exists and is writable by Docker.

  

---

  

## 📜 License

  

This setup uses the upstream image **fradelg/mysql-cron-backup** under its respective license. Your additional scripts/config here can be licensed as you prefer.# MySQL / MariaDB Cron Backup with Gotify Notifications

  

Automated backups of a remote **MySQL/MariaDB** database using [`fradelg/mysql-cron-backup`](https://github.com/fradelg/mysql-cron-backup) — with optional **Gotify** push notifications when new backup files are created.

  

>  **Highlights**

>

>  - 🕒 Schedule backups with cron (daily, hourly, custom)

>  - 🗃️ Keeps only the most recent *N* backups

>  - 💾 Plain `.sql` or compressed `.sql.gz` dumps

>  - 🔔 Gotify notifications with file name, size and timestamp

>  - 🐳 One-command Docker Compose deployment

  

---


  

---

  

## 🔁 Restore (from a backup)

  

Pick the newest file and restore to a target server:

  

```bash

# Plain .sql

mysql  -h <host> -P <port> -u <user> -p <database> < backups/2025-11-03_0200.prod.sql

  

# Gzip file (.sql.gz)

gunzip  -c  backups/2025-11-03_0200.prod.sql.gz | mysql  -h <host> -P <port> -u <user> -p <database>

```

  

> Always restore to a test database first to verify integrity and permissions.

  

---

  

## 🛡️ Security Tips

  

- Use a **least-privileged** DB user that can run `SELECT` and dump routines/events if needed.

- Store `.env` outside of version control (`.gitignore`) if it contains secrets.

- Restrict filesystem permissions on `./backups`.

- Consider encrypting backups at rest (e.g., with age/gnupg) if needed.

  

---

  

## ⚠️ Troubleshooting

  

-  **No backup files appear**

Check DB connectivity, credentials, and `MYSQL_HOST` reachability. Review `mysql-cron-backup` logs.

  

-  **Gotify messages not sent**

Verify `GOTIFY_URL` and `GOTIFY_TOKEN`. Set `CURL_INSECURE=1` only for testing with self-signed TLS.

  

-  **Only the latest N files kept**

Controlled by `MAX_BACKUPS`. Older files are removed by the backup container.

  

-  **Cron timing off**

Ensure `TZ` is set correctly (default: `Europe/Amsterdam`).

  

-  **Permission errors on `./backups`**

Ensure the host directory exists and is writable by Docker.

  

---

  

## 📜 License

  

This setup uses the upstream image **fradelg/mysql-cron-backup** under its respective license. Your additional scripts/config here can be licensed as you prefer.
