# The HomeVerse app store

Every app a HomeVerse device can install, and nothing else. One folder per
app, three files in each: what the app is, how to run it, and what it looks
like.

```
store.yml                        this store's id, name and version
apps/<id>/manifest.yml           what the app is, and what it is allowed to do
apps/<id>/docker-compose.yml     how to run it, with every image pinned
apps/<id>/icon.svg
```

A device never builds anything from this repository. It reads these files,
pulls the exact images they name, and runs them.

## Why the images are pinned

Every image is written as `name:tag@sha256:…`. The tag is for a human reading
the file; the digest is what actually gets pulled. A release of this store is
therefore a fixed set of app versions: two devices installing the same app
from the same store release get byte-for-byte the same containers, today and
in a year.

Moving an app to a newer version is an edit to this repository, reviewed like
any other change, rather than something that happens quietly on a device
overnight.

## The rules

Every app here has to pass the checks in `.github/workflows/store.yml` before
a release goes out. They come from section 5 of the HomeVerse build plan, and
each one exists because of a way a device can be broken:

| Check | What it stops |
| --- | --- |
| the manifest matches its schema, and its id matches its folder | a device building a path to a file that is not there |
| every image pinned by digest | the same app arriving differently on two devices |
| the main service publishes only `127.0.0.1:${APP_HOST_PORT}` | an app port open to the network behind Caddy's back |
| other published ports only with `extra-ports`, and declared | the firewall and the app disagreeing about what is open |
| no `privileged`, host networking, `cap_add`, devices, or the Docker socket without the matching permission | an app with more of the machine than it asked for |
| bind mounts only under `${APP_DATA_DIR}` | an app reading or writing someone else's files |
| no `container_name`, everything `restart: unless-stopped` | two apps colliding, and apps that do not come back after a reboot |
| the main service has a healthcheck that passes with nobody touching a browser | a device that reports an app as starting forever |
| where an app offers an embedded database server and a single file, the single file is preset | the failure that broke Uptime Kuma when its first-run setup was interrupted |
| **arch-check**: every digest resolves for every processor the app claims | an app that installs on a laptop and fails on a Raspberry Pi |
| **upstream-check**: the project is not archived and has released within a year | shipping software nobody is going to patch — File Browser was archived in September 2026 |

The last two need the network, so they run here and never on a device.
Installing an app must not depend on a registry being reachable at that
moment — by then the digests are already decided.

## Adding an app

1. `apps/<id>/` with the three files.
2. `requirements.ramMb` is **measured**, not guessed: install it on a device,
   leave it alone, and read `docker stats`. Round up for headroom.
3. Open a pull request. The checks run on it.

## Using a different store

A HomeVerse device can be pointed at any repository shaped like this one, over
https. It is fetched, checked against every rule above, and marked
**Unverified source** — those rules are about what an app may do to a device,
so they do not relax because somebody chose the store themselves.
