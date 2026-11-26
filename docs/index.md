# Lab infrastructure

How the Societal Analytics Lab (VU Amsterdam) hosts its tools: the login service, the lab's own research software ([AmCAT](amcat.md), [AnnoTinder](annotinder.md)), and tools for lab members, such as transcription. This describes what we do, not how to do it in general: we assume you know your way around a Linux server.

| Page | What |
| --- | --- |
| [Our servers](servers.md) | Which servers we have, and what runs where |
| [Setting up a server](server-setup.md) | Admin accounts, firewall, Docker, `/opt`, backups: the same on every server |
| [Docker Compose](docker.md) | The settings every app uses, and the pitfalls |
| [Apps](apps.md) | Each app: what it is, and the commands to run it |
| [FAQ](faq.md) | Why we do it this way |

## In short

* **Every app runs with Docker Compose,** in its own folder `/opt/<app>`, with everything needed to restore it in that folder.
* **Only HTTPS is public.** Each server has one reverse proxy (Caddy) in front of its apps; apps only listen on the server itself.
* **One login service,** Keycloak at `auth.societal-analytics.nl`, for everything that needs a login. Apps without a login of their own sit behind the [lab portal](apps.md#lab-portal).
* **Backups:** every server backs up `/opt` every night, encrypted, with the database dumps the apps make themselves. VU IT's server backups are the last resort.
* **Everything is open source,** our deployments included: [github.com/vu-societal-analytics](https://github.com/vu-societal-analytics).

## Open points

* **Backups off the server:** our nightly backups are still stored on each server itself. ==TODO: agree with VU IT where they may go, and which servers VU IT backs up, how often, and for how long.==
* **Monitoring:** ==TODO: the lab's healthchecks.io account, for the backup reports.==
* **The database server:** ==TODO: what it stores, and how the other servers connect to it (TLS, which servers).==
* **Whisper and Scriptorium:** ==TODO: their compose files ([Apps](apps.md#whisper)).==
* **Keycloak's address:** ==TODO: move Keycloak to `auth.societal-analytics.nl`.==
* **SSH:** ==TODO: allow SSH only from the VU network on every server ([University VMs](vm.md#ssh-only-from-the-vu-network)).==
* **Privacy statement and acceptable use policy:** drafts, to be reviewed by the VU privacy officer.

The documentation before November 2026, with the background of many choices, is in the [archive](https://github.com/vu-societal-analytics/societal-analytics-lab-infra/tree/main/docs/archive).
