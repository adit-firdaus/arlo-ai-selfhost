# Arlo, self-hosted

Four containers, one command, no accounts anywhere. Your documents, your customers'
conversations and your customers' data stay on your machine.

```sh
git clone https://github.com/adit-firdaus/arlo-ai-selfhost.git arlo
cd arlo
./bootstrap.sh https://arlo.example.com you@example.com
docker compose up -d --wait
```

The application image is pulled, not built: this repository holds the stack, not Arlo's
source. `ARLO_IMAGE` in your shell overrides which image that is, and a real installation
should pin a digest rather than follow the `main` tag.

**While `ghcr.io/autobricks-ai/arlo-ai` is still a private package, that pull fails** with an
authentication error rather than a permission one. Until it is published, point the stack at an
image you have built yourself from the application repository:

```sh
ARLO_IMAGE=arlo:local docker compose up -d --wait
```

Everything else in this repository is exercised by that — the stack was verified end to end
this way, on empty volumes: the seed service, the roles, all eighty migrations, and a first
registration code.

Then point a TLS proxy at `127.0.0.1:9005` — [below](#the-proxy-in-front) — and open the
address you gave. Everything else is configuration you can do later, from inside the app.

## What is running

| Container | What it is | Reachable from |
|---|---|---|
| `seed` | one-shot: copies the database's first-boot scripts out of the app image, then exits | — |
| `db` | Postgres 17 with pgvector and pg_trgm | `127.0.0.1:54341`, for backups |
| `auth` | GoTrue — the same identity provider Supabase runs | the compose network only |
| `authgw` | eight lines of nginx, stripping the `/auth/v1` prefix GoTrue does not serve | the compose network only |
| `app` | Arlo itself, two workers, migrations applied at boot | `127.0.0.1:9005` |

The database's first-boot scripts — the roles, the two extensions and the schema GoTrue owns
— ride inside the application image, and `seed` copies them into a volume that Postgres runs
on its first boot. That is why this repository needs no copy of them: there is exactly one
`00-roles.sql` in the world, in the image, and it is the same file the application's own test
suite runs against.

Data lives in four named volumes: `pgdata`, `uploads`, `waauth`, `ocrcache`. `docker compose
down` keeps all four; `docker compose down -v` destroys them, which is the only command here
that loses anything.

## The proxy in front

This stack deliberately does not bind 80 or 443 — on a box that already serves something,
that is an outage rather than a deploy. Caddy, if you have nothing:

```
arlo.example.com {
    reverse_proxy 127.0.0.1:9005
}
```

The address must match `PUBLIC_URL` in `.env` exactly, scheme included: OAuth redirects, the
widget snippet and every webhook address are built from it, and a mismatch fails at the far
end, where the message is somebody else's.

## The first account

Registration is invitation-only the moment `ADMIN_EMAILS` names anybody, and only an
administrator can invite — but an administrator is somebody who has already registered. The
way out of that loop is a shell:

```sh
docker compose exec app node scripts/invite.mjs you@example.com
```

It prints a code that admits that address once. Register with it at `/register`, and every
later invitation comes from the admin page.

## Turning the answering on

Arlo runs with no model key and is honest about it — retrieval falls back to Postgres
full-text search, and the bot says it cannot answer rather than inventing one. To have it
answer, put an OpenRouter key in `.env` and restart:

```sh
sed -i 's|^OPENROUTER_API_KEY=.*|OPENROUTER_API_KEY=sk-or-v1-…|' .env
docker compose up -d app
```

Embeddings come off the same key and the same bill — `EMBEDDINGS_URL` is already filled in.

## Upgrading

```sh
docker compose pull app && docker compose up -d --wait app
```

New migrations apply themselves at boot, before the first worker serves. If one fails the
container stays down and says why: a worker answering against a schema that did not catch up
is the worse outcome.

## Backups

The database is the whole product; the volumes beside it are a cache and a WhatsApp session.

```sh
docker compose exec -T db pg_dump -U postgres -Fc postgres > arlo-$(date +%F).dump
```

Back up `.env` with it, somewhere else. `SECRET_KEY` in that file is what every stored
connector token is encrypted with: with the dump and without the key, the rows are still
there and none of them can be read.

## Going further

`INSTRUCTIONS.md` is the full setup: a public address, the model key and embeddings, the
knowledge base, all eight channels with their exact redirect and webhook URLs, invitations,
backups, upgrades, and a table of what each symptom actually means.

## The course runbook

`SETUP.md` beside this file is the same stack written for an agent to install on a course
student's Windows machine, step by step, with the image digests pinned and nothing assumed
beyond Docker. Change the stack here and change it there; they are one thing described twice.

## What this is not

Hexcore's own instance at `arlo.autobricks.ai` does not run this stack — it is a systemd unit
against hosted Supabase, behind a Traefik that was already there. That deployment can rebuild
in place and wants to. This one assumes a machine with Docker on it and nothing else, which is
the right assumption for everybody who is not us.

Arlo's source is not here either. This repository is the deployment: four services, the
secrets that wire them together, and the runbook. The image it pulls is built from the private
application repository.
