# Amahi-kai apps

The app catalog for [Amahi-kai](https://github.com/CatDogBark/Amahi-kai): one manifest per app in
`apps/<id>.yml`. Every Amahi-kai NAS fetches this repo's `main` branch every 6 hours (and when its
Apps page's **Check now** is pressed), so a new app, or a new version of one, reaches every NAS
without an Amahi-kai update. Nothing updates on its own: each app's row then offers **Update**.

The NAS's root helper checks every manifest before using it, the same way it checks an install,
and skips the ones that don't pass (the Apps page names them). This repo's CI runs those same
checks on every pull request, so a manifest a NAS would refuse can't be merged.

## A manifest

```yaml
name: Gitea                       # what the Apps page shows, one line each
description: Your own Git server, like a small GitHub, for code and its history.
category: development
logo: https://cdn.jsdelivr.net/gh/walkxcode/dashboard-icons/png/gitea.png   # https, optional
releases: https://github.com/go-gitea/gitea/releases/tag/v{version}         # https, optional
image: gitea/gitea:1.27.3-rootless@sha256:…   # pinned: tag and digest
run_as: app                       # app (its own user, app-<id>) or image (the image's own)
web_port: 3300                    # one of its host ports: Open goes there
memory: 1g                        # a limit, like 512m or 2g (1g if left out)
ports:
  - { host: 3300, container: 3000 }
  - { host: 2222, container: 2222, label: Git over SSH }
folders:                          # its data, kept in /var/lib/amahi-kai/apps/<id>/<name>
  - { name: data, path: /var/lib/gitea }
  - { name: cache, path: /cache, backup: false }   # not copied before an update
environment:
  TZ: "{{timezone}}"              # {{uid}}, {{gid}} and {{timezone}} are filled in at install
secrets:                          # generated at install, shown to admins
  - { env: ADMIN_TOKEN, label: Admin page token }
writes_shares: false              # true lets it be given shares to write to
requires: 1                       # the catalog format it needs (1 if left out)
```

Host ports must be 1024 or above and not the NAS's own; no two apps here may list the same one.
Logos of this repo's own go in `logos/` and are linked through jsDelivr
(`https://cdn.jsdelivr.net/gh/CatDogBark/amahi-kai-apps@main/logos/<file>`).

## Updating an app

From an Amahi-kai checkout beside this one:

```bash
script/app-versions --catalog ../amahi-kai-apps/apps
```

```bash
script/app-versions --catalog ../amahi-kai-apps/apps --update gitea
```

The first lists newer versions with their release notes; the second writes the newest tag and
digest into `apps/gitea.yml`. Read the notes, then open a pull request here.

## Formats

A NAS knows the catalog formats up to the one its Amahi-kai was built with (format 1 today). A
manifest that needs a newer one (`requires: 2`) is listed on older NASes with "Needs a newer
Amahi-kai: run System Update first" instead of Install or Update. Raise `requires` whenever a
manifest uses a field older Amahi-kai versions don't know.
