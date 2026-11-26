# Apps

The apps we host, and how we run each one. All follow [Setting up a server](server-setup.md#apps-in-opt) (a folder in `/opt`, owned by `sal`) and [Docker Compose](docker.md); the commands below are run in the app's folder, as `sal` (`sudo -u sal bash`), except those with `sudo`. Which server runs what: [Our servers](servers.md).

| App | What | For | Login | Data, and how long it is kept |
| --- | --- | --- | --- | --- |
| [Keycloak](#keycloak) | The login service | Everyone with an account | — | Accounts, until deleted |
| [Lab portal](#lab-portal) | The lab's start page, membership, and the login for lab apps without one | Lab members | Keycloak | Members, membership requests, a log of admin actions |
| [Whisper](#whisper) | Transcription of audio | Lab members | Lab portal | Audio and transcripts, ==TODO: deleted after N days== |
| [Scriptorium](#scriptorium) | ==TODO== | Lab members | ==TODO== | ==TODO== |
| [AmCAT](#amcat) | Storing, managing and sharing research data | Researchers, also outside the lab | Its own, through Keycloak | Research data, until the project deletes it |
| [AnnoTinder](#annotinder) | Annotating research data | Researchers, also outside the lab | Its own, through Keycloak | Research data and annotations, until the project deletes it |
| [PostgreSQL](#postgresql) | Databases for apps on other servers | Apps | — | The apps' data |
| [Elasticsearch](#elasticsearch) | Search indexes for apps on other servers | Apps | — | The apps' data |

The [privacy statement](policies/privacy.md) and the [acceptable use policy](policies/aup.md) apply to all of them.

## Keycloak

The login service, at `auth.societal-analytics.nl`: Keycloak with PostgreSQL, its own Caddy, and an email relay. Repository: [deploy-keycloak](https://github.com/vu-societal-analytics/deploy-keycloak), which is **private**: it holds the login flows and settings of every realm, the most sensitive part of our setup. The server reads it with a read-only deploy key, and builds its two images itself (Keycloak prepared for a quick start, and Caddy with a rate limit module):

```bash
# after adding the server's deploy key to the repository (INSTALL.md, "Files and settings")
sudo -u keycloak git clone ssh://git@ssh.github.com:443/vu-societal-analytics/deploy-keycloak.git /opt/keycloak && cd /opt/keycloak
sudo -u keycloak cp .env.example .env && sudo -u keycloak chmod 600 .env      # then fill it in
sudo docker compose up -d --build
```

Everything else (the admin hostname and its IP restriction, the realms, upgrades, recovery) is in the repository's INSTALL.md. The Keycloak folder is owned by its own system account `keycloak`, not by `sal`.

## Lab portal

The start page of the lab server, at `lab.societal-analytics.nl`, and the server's reverse proxy: Caddy, plus a small app that signs people in with Keycloak, handles lab membership (requests, invitations, approval by lab admins), and adds that login to the lab apps that have none. Which apps it serves, and who may use them, is set in `config/services.yml`; the lab admins in `config/admins.yml`. Repository: [deploy-lab-portal](https://github.com/vu-societal-analytics/deploy-lab-portal) ([compose.yml](https://github.com/vu-societal-analytics/deploy-lab-portal/blob/v0.1.0/compose.yml)).

```bash
mkdir -p config caddy && V=v0.1.0 && R=https://raw.githubusercontent.com/vu-societal-analytics/deploy-lab-portal/$V
curl -fsSLO $R/compose.yml
curl -fsSL -o .env $R/.env.example && chmod 600 .env                 # then fill it in
curl -fsSL -o config/services.yml $R/config/services.example.yml
curl -fsSL -o config/admins.yml $R/config/admins.example.yml
mkdir -p data backups && sudo chown 1000:1000 data backups caddy
sudo docker compose pull && sudo docker compose up -d
```

The Keycloak client, the first admin, and adding an app: the [README](https://github.com/vu-societal-analytics/deploy-lab-portal#readme). An app behind the portal is one entry in `config/services.yml`, e.g.:

```yaml
services:
  whisper:                       # https://whisper.lab.societal-analytics.nl
    name: Whisper
    description: Transcription of audio
    upstream: 127.0.0.1:9000     # the port the app publishes on 127.0.0.1
    access: members              # members, admins, or public (an app with its own login)
    mfa: required                # only with a second factor
    max_upload: 2.5GB
    timeout: 3000s
```

## Whisper

Transcription of audio, at `whisper.lab.societal-analytics.nl`, behind the lab portal. ==TODO: which image we use; its compose file goes in this documentation (`apps/whisper/compose.yml`), with a cleanup service for the audio and transcripts ([Docker Compose](docker.md)).==

## Scriptorium

At `scriptorium.lab.societal-analytics.nl`. ==TODO: what it is, its image and compose file.==

## AmCAT

The lab's platform for storing, managing and sharing research data, at `amcat.societal-analytics.nl`, with its own Caddy (the `caddy` profile of its compose file). ==TODO: the release and download commands, as for AnnoTinder below.== How AmCAT handles logins: [AmCAT](amcat.md).

## AnnoTinder

The lab's annotation tool, at `annotinder.societal-analytics.nl`. Repository: [ccs-amsterdam/annotinder](https://github.com/ccs-amsterdam/annotinder) ([compose.yml](https://github.com/ccs-amsterdam/annotinder/blob/v0.9.0/compose.yml)). Its compose file includes the database, its daily dump, and an optional Caddy, which we use: `COMPOSE_PROFILES=caddy` in `.env`.

```bash
V=v0.9.0 && R=https://raw.githubusercontent.com/ccs-amsterdam/annotinder/$V
curl -fsSLO $R/compose.yml
curl -fsSL -o .env $R/.env.example && chmod 600 .env                 # then fill it in
sudo docker compose pull && sudo docker compose up -d
```

## PostgreSQL

On the database server, for the apps on other servers. Our compose file: [compose.yml](apps/postgres/compose.yml), [env.example](apps/postgres/env.example). It dumps every database (and the roles) every day into `./backups`.

!!! warning "A draft"
    Not yet used on the server. ==TODO: TLS, and access only for the servers that need it (`pg_hba.conf`).==

```bash
R=https://vu-societal-analytics.github.io/societal-analytics-lab-infra/apps/postgres
curl -fsSLO $R/compose.yml
curl -fsSL -o .env $R/env.example && chmod 600 .env                  # then fill it in
sudo docker compose up -d
```

Each app gets its own database and role:

```bash
sudo docker compose exec db psql -U postgres -c "CREATE ROLE amcat LOGIN PASSWORD '<password>'" -c "CREATE DATABASE amcat OWNER amcat"
```

## Elasticsearch

On the database server, for the apps on other servers. Our compose file: [compose.yml](apps/elasticsearch/compose.yml), [env.example](apps/elasticsearch/env.example).

!!! warning "A draft"
    Not yet used on the server. ==TODO: TLS between the servers, and a user per app instead of `elastic`.==

```bash
R=https://vu-societal-analytics.github.io/societal-analytics-lab-infra/apps/elasticsearch
curl -fsSLO $R/compose.yml
curl -fsSL -o .env $R/env.example && chmod 600 .env                  # then fill it in
echo 'vm.max_map_count=1048576' | sudo tee /etc/sysctl.d/99-elasticsearch.conf && sudo sysctl --system
mkdir -p backups && sudo chown 1000:1000 backups
sudo docker compose up -d
```

Backups are Elasticsearch's own snapshots, in `./backups`. Once, after the first start, register that folder and schedule a daily snapshot, kept 7 days:

```bash
read -rsp 'Password of elastic: ' P && echo
es() { curl -s -u "elastic:$P" -H Content-Type:application/json "http://<internal address>:9200$1" "${@:2}"; echo; }
es /_snapshot/backups -X PUT -d '{"type":"fs","settings":{"location":"/backups"}}'
es /_slm/policy/daily -X PUT -d '{"schedule":"0 30 1 * * ?","name":"<daily-{now/d}>","repository":"backups","config":{"include_global_state":true},"retention":{"expire_after":"7d","min_count":1,"max_count":14}}'
es /_slm/policy/daily/_execute -X POST
```

Restoring an index, after deleting or closing it: `es /_snapshot/backups/<snapshot>/_restore -X POST -d '{"indices":"<index>"}'` (the snapshots: `es /_cat/snapshots/backups`).
