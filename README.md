# Day 12: Docker Compose

A Flask web app with PostgreSQL and Redis, defined in one `compose.yaml`
and started with a single command.

Repo: https://github.com/YOUR-USERNAME/YOUR-REPO

## Architecture

```
Browser/curl --5000--> web (Flask) --> db (PostgreSQL, volume: pgdata)
                                  \--> redis (hit counter)
```

![architecture](evidence/architecture.png)

| Service | Role |
|---|---|
| web | Flask app. `/` increments a Redis counter, inserts a PostgreSQL row, shows both counts. `/health` returns `ok` |
| db | PostgreSQL 16, data stored in the named volume `pgdata` |
| redis | Redis 7, fast counter, no volume |

## Project structure

```
compose.yaml
.env.example      (template, committed)
.gitignore        (ignores .env)
app/
  app.py
  requirements.txt
  Dockerfile
evidence/
```

## Setup

```bash
git clone https://github.com/YOUR-USERNAME/YOUR-REPO.git
cd YOUR-REPO
cp .env.example .env        # then edit the password locally
docker compose config
docker compose up -d --build
docker compose ps
```

`.env` is ignored by Git. Only `.env.example` is committed.

## Test

```bash
curl http://localhost:5000/          # Redis hits: N | Postgres visits: N
curl http://localhost:5000/
curl http://localhost:5000/health    # ok
docker compose logs --tail=50 web
```

Observed: [paste your real output, both counters increased on repeated calls]

![app](evidence/app-response.png)
![status](evidence/compose-status.png)

## Persistence test

```bash
docker compose down
docker compose up -d
curl http://localhost:5000/
```

Observed: the PostgreSQL visits count continued after down/up because `pgdata`
is a named volume. [add your real numbers]
Redis has no volume, so its counter can reset.

![persistence](evidence/persistence-test.png)

## Restart test

```bash
docker kill $(docker compose ps -q web)
docker compose ps
curl http://localhost:5000/
```

Observed: [write what really happened, e.g. web restarted and answered again]

![restart](evidence/restart-test.png)

## Troubleshooting log

| Problem | Evidence | Root cause | Fix |
|---|---|---|---|
| Image build failed | `No matching distribution found for psycopg-binary==2.9.9` | Wrong package name in `requirements.txt` | Use `psycopg2-binary==2.9.9` |
| `docker kill` permission denied | `permission denied ... docker.sock`, then `requires at least 1 argument` | `sudo` applied only to the outer command, so the inner `docker compose ps -q web` returned nothing | `sudo docker kill $(sudo docker compose ps -q web)` |
| web stayed down after kill | `docker compose ps` listed only db and redis | [your cause, e.g. container created without the restart policy] | [your fix, e.g. `docker compose up -d --force-recreate web`] |
| `git restore --staged` failed | `could not resolve 'HEAD'` | Repo had no commits yet | `git rm --cached <file>` |

## Reflection

- **What does each service do?** web serves requests, db keeps permanent visit data, redis keeps a fast counter.
- **How does web find PostgreSQL and Redis?** Compose creates a project network with DNS, so the service names `db` and `redis` resolve. `localhost` would point back to the web container.
- **What proves the database was ready before the app started?** `docker compose ps` showed db and redis as `healthy`, and web uses `depends_on` with `condition: service_healthy`.
- **Why did PostgreSQL data persist while Redis hits could reset?** PostgreSQL stores data in the named volume `pgdata`. Redis has no volume.
- **What would I change before production?** Strong secrets from a secrets manager, pinned image versions, a production server such as gunicorn instead of the Flask dev server, a Redis volume if needed, database backups, and monitoring.

## Commands cheat sheet

```bash
docker compose config            # validate
docker compose up -d --build     # build and start
docker compose ps                # status and health
docker compose logs -f web       # follow logs
docker compose down              # remove containers and network, keep volume
docker compose down -v           # also delete volumes and data (careful)
```

## Stop

```bash
docker compose down
```
