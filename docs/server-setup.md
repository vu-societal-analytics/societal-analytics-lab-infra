# Setting up a server

The same on every server, physical or VM. A VM on VU SciCloud is created as on [University VMs](vm.md), which also has a script for the first steps.

## The basics

* **Ubuntu LTS,** with automatic security updates (`unattended-upgrades`).
* **A personal admin account per person,** with `sudo` and SSH keys. No root logins and no passwords over SSH.
* **Firewall:** only ports 80 and 443 open to everyone, SSH only from the VU network.
* **Docker Engine** with the compose plugin, from [Docker's own repository](https://docs.docker.com/engine/install/ubuntu/). Admins run it with `sudo`; nobody is in the `docker` group.

??? note "How we do it"
    On a VM, [`setup-vm.sh`](install/setup-vm.sh) does all of the above ([University VMs](vm.md)). By hand, the SSH part is:

    ```bash
    printf 'PermitRootLogin no\nPasswordAuthentication no\nKbdInteractiveAuthentication no\n' | sudo tee /etc/ssh/sshd_config.d/00-hardening.conf
    sudo sshd -t && sudo systemctl restart ssh
    ```

    Test a new login in a second terminal before closing the first.

## Apps in /opt

Every app gets a folder `/opt/<app>` with its `compose.yml`, its settings (`.env`) and its data, and runs with Docker Compose ([Docker Compose](docker.md)). The folders are owned by a system account, `sal`, without a login; admins are in its group.

Once per server, create `sal` and add each admin to its group:

```console
$ sudo useradd --system --home-dir /nonexistent --no-create-home --shell /usr/sbin/nologin sal
$ sudo usermod -aG sal jane
$ id sal
uid=999(sal) gid=999(sal) groups=999(sal)
```

For each app, e.g. AnnoTinder:

```console
$ sudo mkdir /opt/annotinder
$ sudo chown sal:sal /opt/annotinder && sudo chmod 750 /opt/annotinder
$ cd /opt/annotinder
$ sudo -u sal curl -fsSLO https://raw.githubusercontent.com/ccs-amsterdam/annotinder/v0.9.0/compose.yml
$ sudo -u sal curl -fsSL -o .env https://raw.githubusercontent.com/ccs-amsterdam/annotinder/v0.9.0/.env.example
$ sudo -u sal chmod 600 .env
$ sudoedit -u sal .env
$ ls -la
total 20
drwxr-x--- 2 sal  sal  4096 Nov 26 10:49 .
drwxr-xr-x 4 root root 4096 Nov 26 10:49 ..
-rw------- 1 sal  sal  4763 Nov 26 10:49 .env
-rw-r--r-- 1 sal  sal  5644 Nov 26 10:49 compose.yml
$ sudo docker compose pull && sudo docker compose up -d
```

* **`750` on the folder:** admins (the `sal` group) can look inside, other accounts can't. **`600` on `.env`:** only `sal` reads the secrets; admins use `sudo`.
* **Work as `sal`:** `sudo -u sal <command>`, and edit with `sudoedit -u sal <file>` (your own editor, saved as `sal`). Plain `sudo` inside `/opt/<app>` leaves files owned by root.
* **Docker with `sudo`:** `sudo docker compose ...`, from the app's folder.
* **Folders the app mounts** (e.g. `data/`): create them as `sal` before the first start (`sudo -u sal mkdir data`); Docker would create them owned by root.

??? note "Why `sal`, and not root or your own account?"
    Files owned by a personal account break when that person leaves; files owned by root can only be changed with full root rights. With one system account, every admin works on every app the same way, and nothing in `/opt` needs root except Docker itself. `sal` is not in the `docker` group, since that is effectively root access. More: [FAQ](faq.md#why-the-sal-account).

**Updating an app:** change the version in `compose.yml` (or download the compose file of the new version), then `sudo docker compose pull && sudo docker compose up -d`. **Removing one:** `sudo docker compose down -v` (this deletes its volumes, so its database too), then `sudo rm -r /opt/<app>`.

## A reverse proxy

Only the reverse proxy (Caddy, with certificates from Let's Encrypt) is reachable from outside; apps listen on `127.0.0.1` or on internal Docker networks. Where it comes from depends on the server:

* **A server with one app** (Keycloak, AmCAT, AnnoTinder): the Caddy that comes with the app's compose file.
* **The lab server,** with many apps: the [lab portal](apps.md#lab-portal), which is Caddy plus a small app that adds the lab's login to apps that have none.

## Backups

Every server runs the same nightly backup: [deploy-backup](https://github.com/vu-societal-analytics/deploy-backup), in `/opt/backup`. It backs up all of `/opt` with restic, encrypted, after checking that every app's database dumps (`/opt/<app>/backups`) are recent, and reports to healthchecks.io, which emails us when a backup failed or didn't run.

```bash
sudo git clone https://github.com/vu-societal-analytics/deploy-backup.git /opt/backup
sudo /opt/backup/setup.sh https://hc-ping.com/<the server's check>
```

Then store the encryption password (`sudo cat /etc/restic/password`) in the lab's password manager. Checks, restores and settings: the [deploy-backup README](https://github.com/vu-societal-analytics/deploy-backup#readme).

So everything needed to restore an app is in `/opt/<app>`: its compose file and `.env`, its files in `data/`, and its database dumps in `backups/`. Databases are dumped by the app itself ([Docker Compose](docker.md#a-database-with-daily-dumps)). The server backups of VU IT are the last resort. ==TODO: for now our backups are stored on the server itself.==
