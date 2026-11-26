# Societal Analytics Lab: infrastructure documentation

The documentation of the Societal Analytics Lab's infrastructure: its servers, how they are set up and secured, our standards for apps and backups, and how our own software handles logins. Published at [vu-societal-analytics.github.io/societal-analytics-lab-infra](https://vu-societal-analytics.github.io/societal-analytics-lab-infra/).

The deployments themselves are in their own repositories:

* [deploy-keycloak](https://github.com/vu-societal-analytics/deploy-keycloak): the lab's login service (Keycloak).
* [deploy-backup](https://github.com/vu-societal-analytics/deploy-backup): the nightly backup of every server.
* [deploy-lab-portal](https://github.com/vu-societal-analytics/deploy-lab-portal): the lab server's start page, membership and login (Caddy and a small app).

Compose files for apps without a repository of their own are in `docs/apps/<app>/`: the site serves them as plain files, for download.

The documentation before November 2026 is in `docs/archive/`: kept for its background, not on the site.

## Editing

Change the files in `docs/` (Markdown), and the menu in `mkdocs.yml`. Keep it short: what we do, with the reasons in folded notes or the FAQ. To preview:

```bash
python -m venv venv
venv/bin/pip install -r requirements.txt
venv/bin/mkdocs serve --livereload
```

Check before committing that the site builds without warnings (`venv/bin/mkdocs build --strict`), and that no secrets, IP addresses or admin hostnames are in it.
