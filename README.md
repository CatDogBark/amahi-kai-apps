# Amahi-kai apps

The app catalog for [Amahi-kai](https://github.com/CatDogBark/Amahi-kai): one manifest per app in
`apps/<id>.yml`, and its logo in `logos/`. Every Amahi-kai NAS fetches this repo's newest **signed
version** every 6 hours (and when its Apps page's **Check now** is pressed), so a new app, or a new
version of one, reaches every NAS without an Amahi-kai update. Nothing updates on its own: each app's row then offers
**Update**. A signed version is a `catalog-N` tag signed with one of Amahi-kai's release keys
([Signed versions](#signed-versions)); a NAS (Amahi-kai 1.8.0 and later) never fetches `main`
itself.

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
| `logo` | An `https` link, for Amahi-kai before 1.8.5 (optional). From 1.8.5 a NAS shows the app's logo from **`logos/<id>.png`** (or `.jpg`, `.webp`, `.svg`; 256 KB at most), which comes with the signed version: it never loads a logo from a link. Every app has one (this repo's CI checks). It must stand out on Amahi-kai's dark cards: at least half of it at 3:1 contrast with the card, which this repo's CI measures (`script/check-logos apps/*.yml`). For a logo made for light pages, use its icon set's light version (dashboard-icons names some `-light`, others `-dark`) |
| `releases` | The release notes for a version, `{version}` being the tag's leading number (`2.5.5` for `2.5.5-rootless`): the Apps page's What's new link (optional, `https`) |
| `image` | `name:tag@sha256:digest`: the image, pinned, on Docker Hub or `ghcr.io`, public. It's the version a new install starts from; Amahi-kai 1.8.5 and later also offer newer releases of the same image, a tag of the same shape (`1.27.3-rootless` finds `28.3.0-rootless`, not `nightly-rootless`), and newer builds of the one an app runs ([Updating an app](#updating-an-app)) |
| `run_as` | `app`: the container runs as the app's own user (`--user`). `image`: it starts as root and switches to the app's user itself (linuxserver.io images, given `PUID`/`PGID`) |
| `web_port` | The host port of the app's web page, one of `ports`'s `host`: where Open goes, and what's announced on the LAN (optional) |
| `web_tls` | `true` if that page is HTTPS, with the app's own certificate: Open and the LAN announcement use `https` (optional; format 2) |
| `access` | `tailscale` for an app reached only from devices on the NAS's Tailscale network, over HTTPS: its ports are published on the NAS itself only, and its `web_port` (plain HTTP on TCP, not `web_tls`) is served on the tailnet with Tailscale's certificate, at `https://<the NAS's name on the tailnet>:<its port>`. It installs once the NAS's Tailscale is connected with MagicDNS and HTTPS Certificates on, and isn't announced on the LAN. Default `lan` (optional; format 5) |
| `network` | What the app reaches out to: `internet` (the default: the internet, not the LAN's devices, the Tailscale network or other private addresses), `lan` (the internet and the LAN's devices too: cameras, a monitor checking the network's machines) or `none` (nothing past the NAS; it still asks the NAS's DNS). Each app is on a network of its own, so none reaches another app, or the NAS's own services. Ask for `lan` only when the app needs it: the Apps page says so on its card. Amahi-kai before 1.6.2 ignores it and lets an app reach everything, as it always did (optional) |
| `capabilities` | The kernel capabilities the app gets back; every container starts with none. An app that runs as its own user (`run_as: app`) needs none: leave it out. An image that starts as root and switches to the app's user itself (`run_as: image`) lists what its setup takes (Transmission: `[CHOWN, DAC_OVERRIDE, SETUID, SETGID]`), from Docker's default set but `NET_RAW` and `MKNOD`, each once. Amahi-kai before 1.6.3 ignores it and gives Docker's set (optional) |
| `memory` | Memory limit, like `512m` or `2g` (default `1g`) |
| `ports` | `host` (1024–65535, not the server's own, and no other app's here), `container`, `protocol` (`tcp` or `udp`, default `tcp`), and `label` for the Apps page (the web port is labelled "web"). `host` is the port the app gets when it's free; if not, it gets the next free one at install, and keeps it |
| `folders` | `name` (a folder under `/var/lib/amahi-kai/apps/<id>/`, owned by the app's user) and `path` in the container; `backup: false` leaves it out of the copy taken before an update (caches, downloads). `kind: dataset` puts it on a ZFS pool instead, in a dataset of the app's own (`<pool>/apps/<id>`) on the pool chosen at install, snapshotted before an update: the data that grows. It's `required` (the app can't be installed without a pool) unless `required: false`, when the app must run without it. The default, `kind: state`, is the system disk: what the app needs to run (format 3) |
| `environment` | Plain settings, one line each; `{{uid}}`, `{{gid}}` and `{{timezone}}` are filled in at install. Names are like `TZ` (capitals, digits, `_`), or, from format 4, as the app spells them (letters of either case, digits, `_` and `-`, starting with a letter: PiGallery2's `Media-folder`). From format 5, `{{tailscale_url}}` is the address of an app with `access: tailscale` (Vaultwarden's `DOMAIN`) |
| `secrets` | `env` and `label`: generated at install, passed as that environment variable, shown to admins. From format 4 a setting can also hold one inside its value as `{{secret:<env>}}` (a password inside a JSON list of users, say); secrets are letters and digits, so they fit inside quotes. From format 5, `hash: argon2` gives the container an Argon2id hash of the secret (PHC form) while admins are shown the secret itself (Vaultwarden's `ADMIN_TOKEN`) |
| `shares` | `true` if the app works with files in shares (a media server, a photo gallery): Install asks which shares to give it, read only, at `/shares/<name>`. An app without it (or `writes_shares`) installs without being asked and gets none; older Amahi-kai versions ask for every app (optional) |
| `writes_shares` | `true` if the app may be given shares to write into (default `false`: shares are read only, at `/shares/<name>`); it says `shares` too. Shares Greyhole pools are always read only |
| `ping` | The path of the image's `ping` when it's marked to need raw sockets (`setcap cap_net_raw`), which apps don't get, so it doesn't run (optional). Amahi-kai mounts a copy of that file without the mark in its place, which pings the way Docker lets any container; older Amahi-kai ignores it |
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
| 8880 | Vaultwarden | web page |
| 9091 | Transmission | web page |
| 51413 | Transmission | peers (TCP and UDP) |
<!-- /ports -->

## Logos

An app's logo is `logos/<id>.png` (or `.jpg`, `.webp`, `.svg`), 256 KB at most. It's part of each
signed version, so a NAS gets it with the manifests, checked by the same signature, and serves it
itself (Amahi-kai 1.8.5 and later). The logos of Gitea, Jellyfin, PiGallery2, Transmission, Uptime
Kuma and Vaultwarden are from [dashboard-icons](https://github.com/homarr-labs/dashboard-icons),
under the Apache License 2.0 ([`logos/LICENSE-dashboard-icons`](logos/LICENSE-dashboard-icons));
each is its project's trademark.

## Adding an app

1. Add `apps/<id>.yml` and its logo, `logos/<id>.png` (or `.jpg`, `.webp`, `.svg`), with host
   ports no other app asks for, and add them to the Ports table (`script/ports` prints it).
2. Check it as a server will, from a checkout of Amahi-kai beside this one:

   ```bash
   ruby --disable-gems ../Amahi-kai/libexec/amahi-helper --check-catalog apps
   ```

3. Open a pull request (pull requests come from collaborators: see [CONTRIBUTING.md](CONTRIBUTING.md)). Its CI runs that same check, checks that every app has its logo, a picture
   of its type, and that every logo stands out on the dark cards.
   What a reviewer looks for besides: the image comes from the app's own project (or a
   well-known packager), it's pinned to a release, not `latest`, it runs as an ordinary user
   where it can, and it was tried the way a server runs it (see the guide).

Once it's merged and in a [signed version](#signed-versions), every server lists it within 6 hours
(or at once, with Check now on its Apps page).

## Updating an app

A new version of an app needs nothing here. A NAS on Amahi-kai 1.8.5 or later checks each
installed app's image every 6 hours for a newer tag of the same shape as its manifest's, and a
newer build of the tag it runs, and offers it (a new major version with a word to read what's new
first); nothing updates by itself. What a manifest is signed for is what the app may do: its
image's repository, ports, folders, settings and what it reaches. Change it when those change.

Older NASes offer only the version a manifest names, and a new install starts from it, so raising
it now and then still helps. From an Amahi-kai checkout beside this one:

```bash
script/app-versions --catalog ../amahi-kai-apps/apps
```

```bash
script/app-versions --catalog ../amahi-kai-apps/apps --update gitea
```

The first lists newer versions with their release notes; the second writes the newest tag and
digest into `apps/gitea.yml`. Read the notes, then open a pull request here. Older servers offer
the update once it's in a [signed version](#signed-versions).

## Signed versions

Whoever could change this repo's `main` would otherwise decide which image every NAS installs. So
a NAS takes only a **`catalog-N` tag** (N counts up from 1, no leading zero) signed with one of
Amahi-kai's release keys (the keys in its
[`config/allowed_signers`](https://github.com/CatDogBark/Amahi-kai/blob/main/config/allowed_signers),
the same ones that sign its releases). It takes the highest N, and never one lower than the
version it has. A tag that isn't signed with a release key, is a plain tag, or calls itself by
another name inside is refused, and the NAS keeps the catalog it has. Its Apps page names the
version it has and the key that signed it.

After merging into `main`, the maintainer signs the next version on the merge commit, with a
release key (it asks for the key's passphrase), checks it, and pushes it:

```bash
git fetch origin
```

```bash
git -c gpg.format=ssh -c user.signingkey=$HOME/.ssh/<release key> tag -s catalog-<N> origin/main -m "catalog-<N> — what changed"
```

```bash
git -c gpg.format=ssh -c gpg.ssh.allowedSignersFile=<Amahi-kai's config/allowed_signers> -c gpg.minTrustLevel=fully tag -v catalog-<N>
```

```bash
git push origin catalog-<N>
```

The **Signed version** workflow then checks the pushed tag as a NAS will: its name, its signature,
that it's on `main`, that it's the highest, and every manifest in it. Never reuse or delete a
version: a NAS that has `catalog-7` refuses `catalog-6`. A version of Amahi-kai that needs a new
manifest field ships after the version of the catalog that uses it (`requires` keeps older ones
from misreading it).

## Formats

A NAS knows the catalog formats up to the one its Amahi-kai was built with (format 3 in
Amahi-kai 1.0: format 2 adds `web_tls`, format 3 folders' `kind` and `required`; format 4, in the
release after 1.0.3, environment names as the app spells them and secrets inside values; format 5,
in the release after 1.2.0, `access: tailscale`, `{{tailscale_url}}` and a secret's `hash`). A
manifest that needs a newer one (`requires: 3`) is listed on older NASes with "Needs a newer
Amahi-kai: run System Update first" instead of Install or Update. Raise `requires` whenever a
manifest uses a field older Amahi-kai versions don't know.
