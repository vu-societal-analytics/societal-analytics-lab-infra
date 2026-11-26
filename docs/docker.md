# Docker Compose

Every app runs from one `compose.yml` in `/opt/<app>`. The goal: if an app is compromised, it can't reach the server or the other apps, and everything needed to restore it is in its folder.

## The settings

An example with everything an app may need: the app, its database, a daily dump of that database, and (for a server with only this app) its own Caddy. Everything is in this one file, so a server needs nothing else besides `.env`.

```yaml
name: example

x-logging: &logging              # logs rotate, so they can't fill the disk
  driver: json-file
  options: { max-size: 10m, max-file: "3" }

services:
  app:
    image: ghcr.io/example/app:1.2.3   # a fixed version, never "latest"
    restart: unless-stopped
    logging: *logging
    env_file: .env                     # settings and secrets (chmod 600)
    ports:
      - "127.0.0.1:8080:8080"          # only for the reverse proxy on this server
    volumes:
      - ./data:/app/data               # files that must be kept: backed up
    networks: [default, db]
    depends_on:
      db: { condition: service_healthy }
    # Isolation: no root, no extra privileges, no writes outside its volumes
    user: "1000:1000"
    read_only: true
    tmpfs: [/tmp]
    cap_drop: [ALL]
    security_opt: ["no-new-privileges:true"]
    mem_limit: 1g

  db:
    image: postgres:18.7-alpine
    restart: unless-stopped
    logging: *logging
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?set POSTGRES_PASSWORD in .env}
    volumes:
      - db-data:/var/lib/postgresql    # a named volume: not in our backups (the dumps are)
    networks: [db]                     # only the app and the backup reach it; no internet
    security_opt: ["no-new-privileges:true"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 10s
      retries: 10

  db-backup:
    # A dump at the start and every day at 02:00 (UTC) into ./backups, kept 7 days
    image: postgres:18.7-alpine        # the same version as the database
    restart: unless-stopped
    logging: *logging
    entrypoint: ["/bin/sh", "/db-backup.sh"]
    configs:
      - source: db-backup
        target: /db-backup.sh
    environment:
      PGHOST: db
      PGUSER: app
      PGPASSWORD: ${POSTGRES_PASSWORD}
      PGDATABASE: app
      BACKUP_TIME: "02:00"
      KEEP_DAYS: 7
    volumes:
      - ./backups:/backups
    networks: [db]
    depends_on:
      db: { condition: service_healthy }

  caddy:
    # Only with COMPOSE_PROFILES=caddy in .env: HTTPS for a server with no other reverse proxy
    image: caddy:2.11.9-alpine
    profiles: [caddy]
    restart: unless-stopped
    logging: *logging
    network_mode: host                 # the one exception: see Pitfalls
    configs:
      - source: caddyfile
        target: /etc/caddy/Caddyfile
    volumes:
      - caddy-data:/data               # certificates: keep them (Let's Encrypt has rate limits)

volumes:
  db-data:
  caddy-data:

networks:
  db:
    internal: true

configs:
  caddyfile:
    content: |
      ${PUBLIC_HOST:?set PUBLIC_HOST in .env} {
      	reverse_proxy 127.0.0.1:8080
      }
  db-backup:
    # $$ is a literal $ (Compose fills in ${...} from .env)
    content: |
      set -eu
      umask 077
      time="$${BACKUP_TIME:-02:00}"; keep="$${KEEP_DAYS:-7}"
      hours="$${time%%:*}"; minutes="$${time##*:}"
      at=$$(( $${hours#0} * 3600 + $${minutes#0} * 60 ))   # "08" would be read as octal
      dump() {
      	file="/backups/$$PGDATABASE-$$(date -u +%Y%m%d).dump"
      	if pg_dump --format=custom $${DUMP_OPTIONS:-} > "$$file.partial"; then
      		mv "$$file.partial" "$$file"; echo "$$(date -u -Iseconds) dump written: $$file"
      	else
      		rm -f "$$file.partial"; echo "$$(date -u -Iseconds) ERROR: pg_dump failed" >&2
      	fi
      	find /backups -name "$$PGDATABASE-*.dump" -type f -mtime +"$$((keep - 1))" -delete
      }
      until pg_isready -q; do sleep 5; done
      dump
      while true; do
      	now=$$(date -u +%s); next=$$(( now - now % 86400 + at ))
      	[ "$$next" -gt "$$now" ] || next=$$(( next + 86400 ))
      	sleep $$(( next - now )); dump
      done
```

In short:

| Setting | Why |
| --- | --- |
| A fixed image version | Upgrades happen when we choose, after reading the release notes |
| Ports only on `127.0.0.1` | Only the reverse proxy on this server reaches the app ([Pitfalls](#pitfalls)) |
| An `internal` network for the database | The database can't be reached by other containers, and can't reach the internet |
| `user`, `read_only`, `cap_drop`, `no-new-privileges` | A compromised app can't gain rights or change its own files. Use them where the image allows it: official database images need their default capabilities to start |
| `mem_limit`, log rotation | One app can't take the server's memory or disk |
| `./data` for files to keep, `./backups` for dumps, named volumes for the rest | Our backup of `/opt` contains exactly what's needed to restore the app ([Setting up a server](server-setup.md#backups)) |
| `configs:` with `content:` | Small config files live in the compose file itself: one file to download and to read |

## A database with daily dumps

A running database's own files can't be backed up reliably, so the database stays in a named volume, and the `db-backup` service above dumps it every day into `./backups`. The server's nightly backup fails (and so emails us) when a `backups/` folder has no dump from the last 26 hours.

### Restoring a database dump

From the app's folder; this replaces the database with the dump:

```bash
sudo docker compose stop app db-backup
sudo docker compose exec -T db dropdb -U app app
sudo docker compose exec -T db createdb -U app app
sudo cat backups/<dump> | sudo docker compose exec -T db pg_restore -U app -d app --no-owner
sudo docker compose up -d
```

An older dump: restore it from the nightly backup first ([deploy-backup](https://github.com/vu-societal-analytics/deploy-backup#restoring)). Test a restore once when setting up an app.

??? note "Data that may only be kept for a limited time"
    E.g. uploaded audio and transcripts. Put it in a named volume (so it isn't in our backups), and delete it with a cleanup service in the same compose file, or with the app's own retention setting if it has one:

    ```yaml
    services:
      cleanup:
        # Every hour, delete uploads not changed for 30 days
        image: busybox:1.37
        restart: unless-stopped
        command: >
          sh -c 'while true; do
                   find /uploads -type f -mtime +30 -delete;
                   find /uploads -mindepth 1 -type d -empty -delete;
                   sleep 3600;
                 done'
        volumes:
          - uploads:/uploads
    ```

    Only for a folder with nothing else in it (no settings, no database). Test it first with `-mmin +10`.

## Pitfalls

* **Published ports bypass the firewall.** Docker forwards traffic to containers around the server's own firewall rules, so `"8080:8080"` is open to the whole internet, whatever the firewall says. Always `"127.0.0.1:8080:8080"`, or no port at all for containers that only talk to each other. Check after every change; nothing should show `0.0.0.0:` or `[::]:` except the reverse proxy's 80 and 443:

    ```bash
    sudo docker ps --format '{{.Names}}\t{{.Ports}}'
    ```

* **`network_mode: host`** puts a container on the server's own network: it reaches everything on `127.0.0.1`, including the other apps. Never for apps. The one exception is the reverse proxy, which reaches all apps anyway (Caddy in an app's compose file, and the [lab portal](apps.md#lab-portal)'s Caddy).
* **The `docker` group is root access:** anyone who can run Docker can mount the whole disk into a container. Use `sudo`, and don't add anyone (or `sal`) to the group.
* **Missing folders for bind mounts** (`./data`) are created by Docker, owned by root: create them as `sal` first.
* **`docker compose down -v`** and `docker volume prune` delete the volumes, with the database. Only when removing an app.
* **Containers also bypass outgoing firewall rules:** a container can connect anywhere on the internet. Put databases on an `internal` network.
