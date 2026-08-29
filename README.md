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

Logs are also not backfilled: Alloy starts from the current position, so lines written
before it first ran, or while it was down, stay on disk and are reachable only through
`docker logs` until rotation discards them.

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

## Files

| File | What it is |
|---|---|
| `docker-compose.yml` | the two services |
| `loki-config.yaml` | storage, 30-day retention, level detection |
| `config.alloy` | container discovery, labels, level extraction, shipping |

Every non-obvious choice is commented at the line it affects rather than described here.
