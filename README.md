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
| `sign_in` | How to sign in the first time, one sentence, shown under the installed app on the Apps page and when its install finishes: who to sign in as and with what ("the password Amahi-kai made for it", for a generated one), or what its first page asks for. Every app here has one (this repo's CI checks); older Amahi-kai versions don't show it |
| `logo` | An `https` link (optional). Logos of this repo's own go in `logos/`, linked through jsDelivr: `https://cdn.jsdelivr.net/gh/CatDogBark/amahi-kai-apps@main/logos/<file>`. A changed logo gets a new file name (`bitshare-2.svg`): browsers keep a logo for a week. It must stand out on Amahi-kai's dark cards: at least half of it at 3:1 contrast with the card, which this repo's CI measures (`script/check-logos apps/*.yml`). For a logo made for light pages, use its icon set's light version (dashboard-icons names some `-light`, others `-dark`) |
| `releases` | The release notes for a version, `{version}` being the tag's leading number (`2.5.5` for `2.5.5-rootless`): the Apps page's What's new link (optional, `https`) |
| `image` | `name:tag@sha256:digest`: the exact image, pinned, on Docker Hub or `ghcr.io`, public |
| `run_as` | `app`: the container runs as the app's own user (`--user`). `image`: it starts as root and switches to the app's user itself (linuxserver.io images, given `PUID`/`PGID`) |
| `web_port` | The host port of the app's web page, one of `ports`'s `host`: where Open goes, and what's announced on the LAN (optional) |
| `web_tls` | `true` if that page is HTTPS, with the app's own certificate: Open and the LAN announcement use `https` (optional; format 2) |
| `memory` | Memory limit, like `512m` or `2g` (default `1g`) |
| `ports` | `host` (1024–65535, not the server's own, and no other app's here), `container`, `protocol` (`tcp` or `udp`, default `tcp`), and `label` for the Apps page (the web port is labelled "web"). `host` is the port the app gets when it's free; if not, it gets the next free one at install, and keeps it |
| `folders` | `name` (a folder under `/var/lib/amahi-kai/apps/<id>/`, owned by the app's user) and `path` in the container; `backup: false` leaves it out of the copy taken before an update (caches, downloads). `kind: dataset` puts it on a ZFS pool instead, in a dataset of the app's own (`<pool>/apps/<id>`) on the pool chosen at install, snapshotted before an update: the data that grows. It's `required` (the app can't be installed without a pool) unless `required: false`, when the app must run without it. The default, `kind: state`, is the system disk: what the app needs to run (format 3) |
| `environment` | Plain settings, one line each; `{{uid}}`, `{{gid}}` and `{{timezone}}` are filled in at install. Names are like `TZ` (capitals, digits, `_`), or, from format 4, as the app spells them (letters of either case, digits, `_` and `-`, starting with a letter: PiGallery2's `Media-folder`) |
| `secrets` | `env` and `label`: generated at install, passed as that environment variable, shown to admins. From format 4 a setting can also hold one inside its value as `{{secret:<env>}}` (a password inside a JSON list of users, say); secrets are letters and digits, so they fit inside quotes |
| `writes_shares` | `true` if the app may be given shares to write into (default `false`: shares are read only, at `/shares/<name>`). Shares Greyhole pools are always read only |
| `requires` | The catalog format it needs (default 1); see Formats |

## Ports

The host ports the apps ask for, so a new one can take a port that's free. No two apps may ask for
the same port, and this table must be the manifests' ports: this repo's CI checks both with
`script/ports --check README.md`, and `script/ports` prints the table. A NAS still gives an app the
next free port when its own is taken there (by something else on that NAS), and the app keeps it.
Ports under 1024, and 3000 (Amahi-kai's own page), are refused by the manifests' check.

<!-- ports -->
| Port | App | For |
| --- | --- | --- |
| 2222 | Gitea | Git over SSH |
| 3001 | Uptime Kuma | web page |
| 3300 | Gitea | web page |
| 8096 | Jellyfin | web page |
| 8098 | PiGallery2 | web page |
| 8443 | bitShare | web page (HTTPS) |
| 8484 | bitTube | web page |
| 8880 | Vaultwarden | web page |
| 9091 | Transmission | web page |
| 51413 | Transmission | peers (TCP and UDP) |
<!-- /ports -->

## Adding an app

1. Add `apps/<id>.yml` (and its logo in `logos/`, if it's this repo's own), with host ports no
   other app asks for, and add them to the Ports table (`script/ports` prints it).
2. Check it as a server will, from a checkout of Amahi-kai beside this one:

   ```bash
   ruby --disable-gems ../Amahi-kai/libexec/amahi-helper --check-catalog apps
   ```

3. Open a pull request. Its CI runs that same check, checks the logos linked from here, and that every logo
   stands out on the dark cards.
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

A NAS knows the catalog formats up to the one its Amahi-kai was built with (format 3 in
Amahi-kai 1.0: format 2 adds `web_tls`, format 3 folders' `kind` and `required`; format 4, in the
release after 1.0.3, environment names as the app spells them and secrets inside values). A
manifest that needs a newer one (`requires: 3`) is listed on older NASes with "Needs a newer
Amahi-kai: run System Update first" instead of Install or Update. Raise `requires` whenever a
manifest uses a field older Amahi-kai versions don't know.
