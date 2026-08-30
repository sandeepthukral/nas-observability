# nas-observability

One place to read the logs of **every container on the NAS**, whatever repo or compose
project it came from.

- **Alloy** asks the Docker daemon for each container's log stream and tags it.
- **Loki** stores it, searchable, for 30 days.
- **Grafana** shows it. Grafana is not in this repo -- see [Where Grafana is](#where-grafana-is).

Containers keep logging exactly as they do now, to Docker's `json-file` driver, so
`sudo docker logs <container>` still works and is still the fallback when this stack is
the thing that is down.

## Running it

```
cd /volume1/docker/nas-observability
sudo docker compose up -d
```

`sudo` because docker on the Synology needs it. `alphaess-net` must already exist, which
it does whenever the alphaess-collector stack has been started at least once.

Then open Grafana -> Dashboards -> **NAS** -> **Container Logs**.

## Being logged

**A new repo does nothing.** No Grafana, no Loki config, no network join, no compose edit,
no code change. Start the container and its lines are in the dashboard within about 15
seconds, tagged with its compose project name.

That holds because Alloy does not reach containers over the network -- it asks the Docker
daemon for the logs of every container the daemon knows about. Running on this NAS is the
entire qualification.

Two things can still leave a container out, and they are worth knowing because both are
silent:

- **It writes to a logfile inside itself instead of to stdout/stderr.** Nothing reaches
  Docker, so nothing reaches here. `sudo docker logs <container>` being empty is the tell.
- **Its compose sets a different log driver.** Check with:
  ```
  sudo docker inspect -f '{{.HostConfig.LogConfig.Type}}' <container>
  ```
  `json-file` or `local` are collected. `none`, `syslog` and `gelf` are not.

**The first start replays history, and it can be a lot of it.** On an empty positions
file Alloy reads each container's whole retained `json-file` log, not just new lines --
so a container with no rotation cap hands over months at once. Loki refuses anything
older than `reject_old_samples_max_age` and Alloy logs a `400` per rejected batch, which
looks alarming and is not: the valid entries in each batch are still stored, and it
drains in about a minute. After that the positions file makes restarts resume cleanly.

What *is* lost is anything older than the 30-day retention window, and anything written
while Alloy is down and rotated away before it comes back. `docker logs` remains the
only route to those.

## Where Grafana is

Grafana, InfluxDB and the `alphaess-net` network are owned by **alphaess-collector**. This
repo declares that network `external: true` and defines none of its own, per that repo's
DEPLOY.md, "Sharing the stack with another project".

So the two halves of this feature live apart, deliberately -- Grafana provisioning cannot
be owned by two repos at once:

| Piece | Repo |
|---|---|
| `loki` and `alloy` services, their configs | here |
| Loki datasource, Container Logs dashboard, error alert rule | alphaess-collector, under `grafana/` |

Practical consequence: changing a label in `config.alloy` may require a matching change to
the dashboard queries in the other repo, and nothing checks that for you.

## Changing a config file

`loki-config.yaml` and `config.alloy` are bind mounts, and **`docker compose up -d` will
not pick up an edit to either.** Compose diffs the mount *spec*, not the file behind it,
so an identical spec means no recreate and the container keeps running with the old
config already read into memory. Nothing reports this — the file on disk is right, the
running service disagrees, and `up -d` prints nothing to say so.

```
cd /volume1/docker/nas-observability
git pull
sudo docker compose up -d --force-recreate loki    # or alloy
```

Then confirm against the running process rather than the file:

```
sudo docker compose exec loki wget -qO- http://localhost:3100/config | grep -A1 reject_old
```

Expect `30d` for the current retention setting. Loki normalises durations, so `720h` in
the file reads back as `30d` — that is the same value, not a setting that failed to
apply.

A restart also puts a short burst into the error dashboard: Alloy logs `No such
container` for the container that just went away, `empty ring` while Loki forms its ring,
and `error sending batch, will retry` for anything buffered meanwhile. All three clear
themselves within a few seconds.

## Querying Loki by hand

Alloy's image is distroless and has no shell tools, so `exec` into **loki** instead --
its own image ships `wget`, which is also what its healthcheck uses:

```
cd /volume1/docker/nas-observability
sudo docker compose exec loki wget -qO- 'http://localhost:3100/loki/api/v1/label/project/values'
```

That one is the health check worth knowing: it lists every compose project currently
reaching Loki. A project missing from it is one whose log driver needs checking, per
"Being logged" above.

## Files

| File | What it is |
|---|---|
| `docker-compose.yml` | the two services |
| `loki-config.yaml` | storage, 30-day retention, level detection |
| `config.alloy` | container discovery, labels, level extraction, shipping |

Every non-obvious choice is commented at the line it affects rather than described here.
