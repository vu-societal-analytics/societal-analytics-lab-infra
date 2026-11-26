# FAQ

Why we set things up the way we do. The longer reasoning, and the options we dropped, are in the archived [Decisions](https://github.com/vu-societal-analytics/societal-analytics-lab-infra/blob/main/docs/archive/decisions.md).

## Why Docker Compose for everything?

One way of working for every app: its whole setup is one file you can read, its versions are pinned, and it runs isolated from the server and the other apps. Backups and retention work the same for every app, because every app keeps its data in the same places ([Docker Compose](docker.md)).

## Why /opt, and not /srv?

Both are defensible: by the Linux conventions, `/opt` is for added software and `/srv` for data a server serves, and an app's folder holds both. What matters is that it is one place, the same on every server: one folder per app, with everything needed to restore it, and one folder for the backup to cover. Splitting an app over two places would lose that.

## Why the `sal` account?

So that the apps' files belong to neither a person nor root. Files owned by a personal account break when that person leaves; files owned by root can only be changed with full root rights, and every mistake runs as root. With one system account without a login, every admin works on every app the same way (`sudo -u sal`, `sudoedit -u sal`), and the secrets in `.env` are readable only by `sal`. You could use root instead; we prefer not to.

## Why Docker with `sudo`, and not the `docker` group?

Anyone who can run Docker can mount the whole disk into a container, so the `docker` group is root access without a password and without logging. `sudo` asks, and logs.

## Why only ports on `127.0.0.1`?

Docker sends traffic to containers around the server's own firewall, so a port published on all addresses is open to the internet, whatever the firewall says. On `127.0.0.1`, only the reverse proxy on the same server can reach the app ([Pitfalls](docker.md#pitfalls)).

## Why database dumps, and not a backup of the volumes?

A copy of a running database's files can be inconsistent; a dump can't. The app dumps its own database (a service in its compose file), so the backup job doesn't need to know anything about the apps, and never runs app commands as root.

## Why restic and healthchecks.io?

restic encrypts on the server, before anything leaves it, only stores changes, and checks its own backups. healthchecks.io notices a backup that failed and also one that didn't run at all, without the server needing email credentials.

## Why Keycloak?

One login service for all our apps, with separate realms (logins) in one installation, a second factor, and the standard (OIDC) that our own software expects ([Authentication standard](auth.md)). It is widely used and maintained, and the lab's admins can reset a member's second factor. We compared others in the archived [Provider comparison](https://github.com/vu-societal-analytics/societal-analytics-lab-infra/blob/main/docs/archive/auth-providers.md).

## Why our own lab portal, and not oauth2-proxy?

The lab server needed a start page and a way to handle lab membership anyway, and that app could also do the login, with a few extra features. Who may use which app, and with or without a second factor, is set in a file on the server, which the portal turns into Caddy's configuration. It is a plain OIDC client of Keycloak, like AmCAT and AnnoTinder ([deploy-lab-portal](https://github.com/vu-societal-analytics/deploy-lab-portal#readme)).

## Why is everything under `lab.` signed in with one login?

So that members sign in once, and signing out ends it everywhere. The login cookie is valid for everything under `lab.societal-analytics.nl`, so only apps the lab runs on its own servers may get an address there, never a site hosted by someone else.

## Why are our images on GitHub, and public?

Our deployments are open source, like the rest of our software. GitHub Actions builds them from the repository and publishes them in the GitHub Container Registry (`ghcr.io`), so a server only downloads an image, and anyone can check what it contains. (Keycloak is the exception: its two images are thin layers on the official ones, built on the server in seconds.)

## Why is this documentation so short?

It describes what we do, for people who know how to run a server. The earlier, step-by-step version is in the [archive](https://github.com/vu-societal-analytics/societal-analytics-lab-infra/tree/main/docs/archive).
