# Immich on Cubeship

[Immich](https://immich.app) is a self-hosted photo and video backup: phone
apps that upload in the background, a web timeline, albums and sharing, and
search by faces, places and what is in the picture. This template installs it
on a Cubeship instance.

## What it creates

- **immich** — the Immich server `v3.2.0`: the web app, the API the phone apps
  talk to, and the background jobs that make thumbnails and transcode videos.
  It answers on the domain you choose and keeps every upload in a volume at
  `/data`.
- **machine-learning** — Immich's machine learning `v3.2.0`, on the CPU: smart
  search, face detection and recognition, and text in images. It has no
  domain, listens on port `3003` inside the instance, and keeps the models it
  downloads in a volume at `/cache`.
- **postgres** — Immich's own Postgres 14 with VectorChord, built from
  [`database/Dockerfile`](database/Dockerfile). It has no domain, listens on
  port `5432` inside the instance, and keeps its data in a volume at
  `/var/lib/postgresql/data`.
- **immich-redis** — a managed Redis 7.4, where Immich queues its jobs.

They are deployed in that order: the database, then machine learning, then the
server.

It needs Cubeship 0.7.0 or newer, and **an admin to install it**: the database
is built on the instance, and only admins build.

## Why the database is its own app, and built

Immich stores its search and face embeddings with the VectorChord extension.
The managed Postgres is the plain `postgres` image, which does not have it, so
the template runs the image Immich's own
[compose file](https://github.com/immich-app/immich/releases/download/v3.2.0/docker-compose.yml)
runs, `ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0`, as an
app with a volume. As its own container, Postgres makes Immich's user a
superuser, which Immich expects.

Compose starts that container with 128 MB of shared memory. Cubeship cannot
size it, and Docker's default of 64 MB is too little for the large queries a
phone's first full sync runs; Postgres then fails them with `could not resize
shared memory segment`. The Dockerfile adds
[one setting](database/postgresql.override.conf), through a file the image's
configuration already includes: Postgres takes that memory from System V
shared memory, which that limit does not cover.

The image refuses to start when its data is not on a local Linux filesystem
(ext4, xfs, btrfs or zfs). A volume is a directory on the server's own disk,
which on a VPS is one of those.

## What you are asked

| Input | What to give |
| --- | --- |
| Where Immich answers | A domain you control, pointed at your instance. The phone apps connect to it. |
| The password Immich uses for its database | Nothing — the instance generates it. |

## After installing

1. **Open `https://<your domain>` straight away.** Immich has no default
   account: the first person to sign up becomes the admin.
2. In the Immich app on your phone, enter `https://<your domain>` as the server
   URL and sign in.

Immich finds the machine learning app through `IMMICH_MACHINE_LEARNING_URL` on
the `immich` app. That is only the default of the setting under
Administration → Settings → Machine Learning: a URL saved there wins.

The first time a job needs a model, machine learning downloads it from Hugging
Face, so the server needs to reach the internet. Until then smart search and
faces show nothing, and a new library takes a while to index on a CPU.

## Reaching the data

Cubeship has no console into an app. Anything that needs a shell is done over
SSH on the machine the app runs on, with `docker exec`. For example, to reset
the admin's password with Immich's own command:

```bash
docker exec -it $(docker ps -qf name=cubeship-immich-production-immich) \
  immich-admin reset-admin-password
```

or to open the database:

```bash
docker exec -it $(docker ps -qf name=cubeship-immich-production-postgres) \
  psql -U postgres immich
```

The internal names follow the project, environment and app names you install
with; they are on each app's page in the dashboard.

## The volumes

Each of the three apps runs as one copy on the machine its volume is on, and a
deploy stops it for a few seconds. While the database or the server is
stopped, the web app and the phone apps cannot reach Immich.

**Uploads are the whole library.** The `/data` volume holds every original
photo and video, plus thumbnails and transcoded videos, which add 10–20% on
top. Size the server's disk for it.

Back up both the database volume and `/data`: the database holds the albums,
people, faces and which file is which asset, and restoring one without the
other loses that match. A volume backup stops the app for the whole copy, and
copies every file to an S3 store outside the instance — for a large library,
schedule it for when nobody is uploading, and expect it to take as long as
copying the library does. The `/cache` volume holds only models, which machine
learning downloads again.

Immich also dumps its own database into `/data/backups` every night at 2:00
and keeps the last 14 (Administration → Settings → Backup), so a backup of
`/data` carries a recent copy of the database with it.

## Resources

The server, machine learning and the database are each limited to 2 GiB of
memory; the server and machine learning to 2 CPUs, the database to 1. Immich
asks for at least 6 GB of memory and 2 cores for the whole stack, and
recommends 8 GB and 4 cores: a VPS with 8 GB is the smallest to install this
on. Machine learning unloads a model after five minutes unused, and raising
its `limits` is what helps a large first import.

## Updating

Change both `tag`s in `template.yaml` to the new release. Immich migrates its
database when the server starts, and there is no going back: back up the
database volume and `/data` first, and read the
[release notes](https://github.com/immich-app/immich/releases). Move the
database image in `database/Dockerfile` only to the one that release's
`docker-compose.yml` names, then release this repository and point the
`postgres` app's `ref` at the new release.
