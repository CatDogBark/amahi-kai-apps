# Amahi-kai apps

The app catalog for [Amahi-kai](https://github.com/CatDogBark/Amahi-kai): one manifest per app in
`apps/<id>.yml`. Every Amahi-kai NAS fetches this repo's `main` branch every 6 hours (and when its
Apps page's **Check now** is pressed), so a new app, or a new version of one, reaches every NAS
without an Amahi-kai update. Nothing updates on its own: each app's row then offers **Update**.

The NAS's root helper checks every manifest before using it, the same way it checks an install,
and skips the ones that don't pass (the Apps page names them). This repo's CI runs those same
checks on every pull request, so a manifest a NAS would refuse can't be merged.

## Making an app

The wiki's [Making Apps](https://amahi-kai.com/wiki/making-apps) is the guide: what Amahi-kai does
with an app, what its image needs, and how to try it the way a server runs it. This README is
the reference.

## A manifest

`apps/<id>.yml`, where `<id>` is the app's name in lowercase letters and digits (2 to 24). Here's
Gitea's:

```yaml
name: Gitea
description: Your own Git server, like a small GitHub, for code and its history.
category: development
logo: https://cdn.jsdelivr.net/gh/walkxcode/dashboard-icons/png/gitea.png
releases: https://github.com/go-gitea/gitea/releases/tag/v{version}
image: gitea/gitea:1.27.3-rootless@sha256:1c17ecaead42eb3b5391553d8708103a4beb0e86edf5b9ebc1eb269c318845f2
run_as: app
web_port: 3300
memory: 1g
ports:
  - { host: 3300, container: 3000 }
  - { host: 2222, container: 2222, label: Git over SSH }
folders:
  - { name: data, path: /var/lib/gitea }
  - { name: config, path: /etc/gitea }
environment:
  TZ: "{{timezone}}"
```

| Field | Meaning |
| --- | --- |
| `name`, `description`, `category` | What the Apps page shows, one line each |
| `logo` | An `https` link (optional). Logos of this repo's own go in `logos/`, linked through jsDelivr: `https://cdn.jsdelivr.net/gh/CatDogBark/amahi-kai-apps@main/logos/<file>`. A changed logo gets a new file name (`bitshare-2.svg`): browsers keep a logo for a week |
| `releases` | The release notes for a version, `{version}` being the tag's leading number (`2.5.5` for `2.5.5-rootless`): the Apps page's What's new link (optional, `https`) |
| `image` | `name:tag@sha256:digest`: the exact image, pinned, on Docker Hub or `ghcr.io`, public |
| `run_as` | `app`: the container runs as the app's own user (`--user`). `image`: it starts as root and switches to the app's user itself (linuxserver.io images, given `PUID`/`PGID`) |
| `web_port` | The host port of the app's web page, one of `ports`'s `host`: where Open goes, and what's announced on the LAN (optional) |
| `web_tls` | `true` if that page is HTTPS, with the app's own certificate: Open and the LAN announcement use `https` (optional; format 2) |
| `memory` | Memory limit, like `512m` or `2g` (default `1g`) |
| `ports` | `host` (1024–65535, not the server's own, and no other app's here), `container`, `protocol` (`tcp` or `udp`, default `tcp`), and `label` for the Apps page (the web port is labelled "web"). `host` is the port the app gets when it's free; if not, it gets the next free one at install, and keeps it |
| `folders` | `name` (a folder under `/var/lib/amahi-kai/apps/<id>/`, owned by the app's user) and `path` in the container; `backup: false` leaves it out of the copy taken before an update (caches, downloads) |
| `environment` | Plain settings, one line each; `{{uid}}`, `{{gid}}` and `{{timezone}}` are filled in at install |
| `secrets` | `env` and `label`: generated at install, passed as that environment variable, shown to admins |
| `writes_shares` | `true` if the app may be given shares to write into (default `false`: shares are read only, at `/shares/<name>`). Shares Greyhole pools are always read only |
| `requires` | The catalog format it needs (default 1); see Formats |

## Adding an app

1. Add `apps/<id>.yml` (and its logo in `logos/`, if it's this repo's own).
2. Check it as a server will, from a checkout of Amahi-kai beside this one:

   ```bash
   ruby --disable-gems ../Amahi-kai/libexec/amahi-helper --check-catalog apps
   ```

3. Open a pull request. Its CI runs that same check, and checks the logos linked from here.
   What a reviewer looks for besides: the image comes from the app's own project (or a
   well-known packager), it's pinned to a release, not `latest`, it runs as an ordinary user
   where it can, and it was tried the way a server runs it (see the guide).

Within 6 hours of the merge every server lists it (or at once, with Check now on its Apps page).

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

A NAS knows the catalog formats up to the one its Amahi-kai was built with (format 2 today:
format 2 adds `web_tls`). A
manifest that needs a newer one (`requires: 2`) is listed on older NASes with "Needs a newer
Amahi-kai: run System Update first" instead of Install or Update. Raise `requires` whenever a
manifest uses a field older Amahi-kai versions don't know.
