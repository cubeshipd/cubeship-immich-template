# The image Immich's v3.2.0 compose file runs. Compose gives it 128 MB of
# shared memory; Cubeship cannot size it, and Docker's 64 MB default is too
# little for the queries a mobile sync runs. The override moves Postgres'
# dynamic shared memory off /dev/shm. The image's config includes the file.
FROM ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0
COPY database/postgresql.override.conf /etc/postgresql/postgresql.override.conf
