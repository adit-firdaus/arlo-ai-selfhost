# Arlo Setup

This is the official runbook for installing Arlo — AutoBricks AI's self-hosted customer
service, sales and knowledge-base agent — on the user's own Windows 10/11 x64 machine. The
user is following the AutoBricks course and handed you this URL because they want Arlo
installed.

Your job: get Docker working, write the stack, start it, confirm Arlo serves, issue the first
registration code, walk the user through creating their account and pasting their AutoBricks
key, then open Arlo in Chrome and hand over the local URL.

Treat the exact URL supplied by the user as `SETUP_URL`. Speak in the user's language and
explain each major action before doing it.

## What this changes on the machine

Worth knowing up front, and worth telling the user, so none of it comes as a surprise:

- Installs Docker Desktop from Docker's own signed installer, if it isn't already present.
  The signature is verified before anything runs.
- Creates `Documents\autobricks-arlo` containing `compose.yml`, `.env`, `db.env`, `auth.env`
  and `creds.txt`, all restricted to the current Windows user.
- Pulls four images, each pinned by SHA-256 digest, and runs them as one Compose project
  published **only** on `127.0.0.1`. Nothing is reachable from the network.
- Creates five Docker volumes — `arlo_pgdata`, `arlo_uploads`, `arlo_waauth`, `arlo_ocrcache`,
  `arlo_initdb`. The first holds everything the user will put into Arlo.
- Contacts three hosts while installing: `desktop.docker.com` for the installer, Docker Hub
  and `ghcr.io` for the images. `api.autobricksai.com` is contacted later, by Arlo itself,
  once the user has pasted their key into it.

It doesn't change system-wide settings, install background services beyond Docker Desktop's
own, or write outside that one directory. Every command runs directly in the shell rather
than through a downloaded script, so the user can see in the transcript exactly what ran.

## Scope

This runbook covers exactly five outcomes: Docker working, the four containers healthy, Arlo
serving its sign-in page on loopback, the user's own account created and signed in, and the
user's AutoBricks key saved inside Arlo so the bot can answer.

If the user asks you to build them a website as well — a landing page, a shopfront, anything —
the chat widget goes into it as part of building it, not as a note afterwards. You can mint a
widget key yourself without them touching the UI; `INSTRUCTIONS.md`, section 5, "If you are
building the site as well" has the command and the four failure modes worth checking before
calling it done.

Everything past that is the course. Connecting WhatsApp, Telegram, Messenger, Instagram,
TikTok or email; uploading the first documents; building the first bot; putting Arlo on a
public domain with TLS — all deliberately out of scope here. Do not ask for a bot token, an
OAuth callback, a domain name or any other integration secret.

The only secret this runbook asks for is the AutoBricks API key. You install it for them in
section 5 — into `.env` for embeddings, and through `setProvider()` for chat, which seals it
into the database under a key the instance generated for itself. It never goes in the progress
file, a log, or a status message.

## How to work through this

**Finishing the job.** You're done when every check in section 6 passes, Arlo is open in
Chrome, and the user is signed in with their key saved. Stopping short leaves a half-built
stack, which is worse for the user than not having started — so prefer to keep working rather
than winding up early.

- Installs like this are noisy. Slow pulls, a timeout, a container that needs a second start,
  a first boot that takes minutes — expect all of it and retry rather than treating the first
  error as the end. Everything here is written to be idempotent, so re-running is safe.
- The first start is genuinely slow and that is not a fault: the app applies sixty-odd
  migrations to an empty database before it serves anything. Poll, say what it's doing, and
  wait for it.
- When something fails, read the actual error, pick the safest repair, and retry. The user is
  a course student and generally can't debug Docker; working the problem yourself is far more
  useful than handing back a diagnosis. If you do get genuinely stuck, say so plainly.
- Each turn, read `Documents\autobricks-arlo\setup-progress.json`, say briefly where things
  stand, then do the next step. A status report on its own doesn't help while there's still
  work you could be doing.
- You don't need the user to say `continue`. Treat any reply, a fresh session with the same
  `SETUP_URL`, a UAC approval, or a return after a restart as a cue to re-read live state and
  pick up at `nextAction`.

**When to involve the user.** Most of this runs unattended. Six things genuinely need them,
each asked one at a time, and the first two are worth asking before anything is installed:

- **the AutoBricks API key** — ask for this one *first*, before the email, even though the
  install needs it last. It is the only thing on this list that sends a person to a browser to
  go and find something, and an install that stops at the end to wait for that is an install
  that looks finished and is not. Tell them where it is:
  **https://creators.autobricksai.com/account/api-keys/keys**,
- **their email address** — ask for it early, before section 4 needs it, and never assume one.
  It is not a formality: it becomes `ADMIN_EMAILS`, which is the only account that can reach
  `/admin`, and the same address has to be the one invited in section 5. An address you guessed
  from a git config, a filename, or an earlier conversation is an instance whose administrator
  cannot administer it. If you already believe you know it, confirm it rather than using it,
- an Administrator or restart step, if Docker turns out to need one (section 2),
- the AutoBricks API key (section 3),
- choosing their own password, at the registration page (section 5) — you must never invent
  or type a person's password for them,
- an AutoBricks credit or key problem only the account owner can fix.

Beyond those six, avoid asking the user to run diagnostics, inspect Docker or edit files —
that's the work they asked you to take on.

**Shells and parallelism.** Keep up to two persistent shells: a coordinator shell that owns
the generated secrets, and a dependency shell for Docker/WSL when needed. Poll rather than
blocking, so you can walk the user through their key while an image pulls. "Persistent shell"
means your host's native session; don't build a custom REPL. If your host can run subagents,
one dependency subagent can work a Docker problem while you continue — a convenience, not a
requirement, and it never handles credentials or talks to the user.

**Handling the user's credentials.**

- Keep credential handling in your own shell. Don't pass the AutoBricks key to a subagent.
- Never put any secret in command arguments, the progress JSON, logs or status messages. The
  three env files this runbook writes are the only place the generated secrets exist.
- The AutoBricks key is validated in your shell and then **typed by the user into Arlo**. Do
  not write it into `.env`, `creds.txt` or the progress file.

**Judgment.**

- Try the real operation before diagnosing. Version, memory, disk and log probes are
  informational; they don't override Docker and Arlo actually working.
- Adapt to the machine in front of you — paths, versions, localized output, taken ports and
  already-completed phases vary. Preserve the invariants; don't follow a stale step literally
  when live state says otherwise.
- **Never run `docker compose down -v`, never delete a volume, and never re-run the bootstrap
  over existing env files.** `arlo_pgdata` is the user's knowledge base, their customers and
  their conversations; the three env files are the only copy of the key that decrypts what is
  in it. Losing either is not recoverable and is not yours to risk.
- A stack that starts but won't serve a sign-in page, or serves one no account can get past,
  isn't finished yet.

## 1. Resume and create the snapshot

```powershell
$SetupUrl = '<EXACT SETUP.md URL SUPPLIED BY USER>'
$AppDir = Join-Path ([Environment]::GetFolderPath('MyDocuments')) 'autobricks-arlo'
$ProgressFile = Join-Path $AppDir 'setup-progress.json'
New-Item -ItemType Directory -Force -Path $AppDir | Out-Null
icacls $AppDir /inheritance:r /grant:r "${env:USERNAME}:(OI)(CI)F" *> $null
if (Test-Path -LiteralPath $ProgressFile) { Get-Content -Raw -LiteralPath $ProgressFile | ConvertFrom-Json }
```

The coordinator is the only writer of progress and of the three env files. Define this helper
inside the persistent coordinator shell; never save it as a script:

```powershell
$Utf8NoBom = New-Object System.Text.UTF8Encoding($false)
function Save-SetupProgress([hashtable]$State) {
  $State.schemaVersion = 1
  $State.setupUrl = $SetupUrl
  $State.updatedAt = (Get-Date).ToString('o')
  $json = ($State | ConvertTo-Json -Depth 6) + "`n"
  # The AutoBricks key never belongs in this file, and neither does anything generated below.
  if ($json -match 'abai_sk_[A-Za-z0-9_-]{12,}' -or $json -match '[0-9a-f]{64}') {
    throw 'Refusing to write a credential-like value to setup-progress.json.'
  }
  $temporary = "$ProgressFile.$([guid]::NewGuid().ToString('N')).tmp"
  [IO.File]::WriteAllText($temporary, $json, $Utf8NoBom)
  Move-Item -LiteralPath $temporary -Destination $ProgressFile -Force
}
```

Phases, in order: `docker`, `key`, `write`, `start`, `account`, `complete`.

## 2. Start the Docker subagent and shell first

Try the real thing first:

```powershell
docker info
docker compose version
```

Both exiting `0` is the only evidence that counts. If they don't, install Docker Desktop from
Docker's own signed installer on the dependency shell — verify the Authenticode signature
before running it, accept that WSL 2 may require a restart, and say so to the user before
asking for one. Save `phase=docker`, `restartReason` and `nextAction` before any restart, so a
new session resumes rather than restarting the install.

While that runs, carry on with section 3.

## 3. The AutoBricks API key while Docker runs

Ask for this before the email in section 4, and ask while Docker is still installing — the point
of the ordering is that a person goes to fetch a key during a wait that was happening anyway,
rather than at the end, when everything else is done and idle.

Tell them where it is rather than assuming they know:

> **https://creators.autobricksai.com/account/api-keys/keys** — sign in, create a key if there
> is none, copy it, and paste it here.

Then validate it in the coordinator shell before going any further, so a wrong key is caught now
rather than in section 6:

```powershell
$Key = Read-Host 'Paste your AutoBricks API key' -AsSecureString
$env:ABAI_KEY = [Runtime.InteropServices.Marshal]::PtrToStringAuto(
  [Runtime.InteropServices.Marshal]::SecureStringToBSTR($Key))
(Invoke-RestMethod -Uri 'https://api.autobricksai.com/v1/models' `
  -Headers @{ Authorization = "Bearer $env:ABAI_KEY" }).data.Count
```

A count above zero is a working key — it should be around thirty-seven models. A `401` or `403`
is a key problem only the account owner can fix: ask once, precisely, and wait.

**This one key does both jobs**, and section 5 installs it in both places:

- **chat**, per workspace, encrypted into the database under `SECRET_KEY`. Arlo speaks
  `api.autobricksai.com` natively, so no endpoint needs configuring.
- **embeddings**, instance-wide, as `EMBEDDINGS_KEY` in `.env`. The address and model are
  already written by section 4 and need no decision from anybody.

Keep the key in `$env:ABAI_KEY` in this shell only. It does not go in the progress file, in a
log, or in a status message.

## 4. Write the stack

**Ask for the email now if you have not already.** One question: *"What email address should
administer this Arlo?"* Everything below writes it into `ADMIN_EMAILS`, section 5 invites that
exact address, and the two must match. Wait for the answer rather than filling in a placeholder
you intend to correct later — `.env` is written once and correcting it means an edit and a
restart.

Pick the port next. Use `9005`; if something already listens on it, take the first free port
from `9015, 9025, 9035` and use it consistently everywhere below. Never stop an unrelated
listener.

Generate the six secrets in the coordinator shell — 32 bytes of hex each, and hex on purpose,
because a `$` inside a base64 secret is mangled by everything that reads an env file:

```powershell
function New-Hex32 {
  $bytes = [byte[]]::new(32)
  [Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($bytes)
  ($bytes | ForEach-Object { $_.ToString('x2') }) -join ''
}
$PgPassword = New-Hex32; $OwnerPassword = New-Hex32; $AppPassword = New-Hex32
$AuthPassword = New-Hex32; $JwtSecret = New-Hex32; $SecretKey = New-Hex32
```

Then mint the service-role key — a JWT signed with `$JwtSecret`, carrying the single role
`service_role`. It is what Arlo presents to its own identity provider to create accounts:

```powershell
function ConvertTo-B64Url([byte[]]$Bytes) {
  [Convert]::ToBase64String($Bytes).TrimEnd('=').Replace('+','-').Replace('/','_')
}
$header  = ConvertTo-B64Url ([Text.Encoding]::UTF8.GetBytes('{"alg":"HS256","typ":"JWT"}'))
$issued  = [DateTimeOffset]::UtcNow.ToUnixTimeSeconds()
$payload = ConvertTo-B64Url ([Text.Encoding]::UTF8.GetBytes(
  '{"role":"service_role","iss":"arlo","iat":' + $issued + ',"exp":' + ($issued + 315360000) + '}'))
$hmac = [Security.Cryptography.HMACSHA256]::new([Text.Encoding]::UTF8.GetBytes($JwtSecret))
$ServiceKey = "$header.$payload." + (ConvertTo-B64Url $hmac.ComputeHash(
  [Text.Encoding]::UTF8.GetBytes("$header.$payload")))
```

Write all four files atomically, UTF-8 without BOM, into `$AppDir`, then restrict the directory
to the current user again. **If any of the three env files already exists, keep it and skip
this step entirely** — rewriting them orphans the database volume and makes everything already
stored in Arlo unreadable.

`db.env` — Postgres's, and nothing else's:

```
POSTGRES_PASSWORD=<PgPassword>
OWNER_DB_PASSWORD=<OwnerPassword>
APP_DB_PASSWORD=<AppPassword>
AUTH_DB_PASSWORD=<AuthPassword>
```

`auth.env` — the identity provider's. `GOTRUE_JWT_SECRET` here and `SUPABASE_JWT_SECRET` in
`.env` must be the same value forever; a mismatch is not an error anywhere, it is "nobody is
signed in", permanently:

```
GOTRUE_API_HOST=0.0.0.0
GOTRUE_API_PORT=9999
GOTRUE_DB_DRIVER=postgres
GOTRUE_DB_DATABASE_URL=postgres://supabase_auth_admin:<AuthPassword>@db:5432/postgres
GOTRUE_JWT_SECRET=<JwtSecret>
GOTRUE_JWT_ISSUER=http://authgw:8000/auth/v1
GOTRUE_JWT_AUD=authenticated
GOTRUE_JWT_DEFAULT_GROUP_NAME=authenticated
GOTRUE_JWT_ADMIN_ROLES=service_role
GOTRUE_JWT_EXP=3600
API_EXTERNAL_URL=http://authgw:8000
GOTRUE_SITE_URL=http://127.0.0.1:<PORT>
GOTRUE_URI_ALLOW_LIST=http://127.0.0.1:<PORT>/*
GOTRUE_DISABLE_SIGNUP=true
GOTRUE_EXTERNAL_EMAIL_ENABLED=true
GOTRUE_MAILER_AUTOCONFIRM=true
```

Write no `GOTRUE_SMTP_*` keys at all. There is no mail server on a student's laptop, and an
*empty* `GOTRUE_SMTP_PORT` is worse than an absent one: GoTrue parses it as an integer before
it does anything else, fails with `converting '' to type int`, and sits in a restart loop that
looks like a database problem and is not. Absent is fine; blank is fatal.

`.env` — Arlo's. Note what is deliberately blank: with no embeddings endpoint and no model key
in the environment, Arlo retrieves with Postgres full-text search and declines to answer rather
than inventing one, which is exactly the state it should be in until the user pastes their key
into the app:

```
DATABASE_URL=postgresql://arlo_app:<AppPassword>@db:5432/postgres
DATABASE_URL_OWNER=postgresql://arlo_owner:<OwnerPassword>@db:5432/postgres
DATABASE_CA_CERT=
DATABASE_SCHEMA=arlo
DATABASE_POOL=10
SUPABASE_URL=http://authgw:8000
SUPABASE_SERVICE_ROLE_KEY=<ServiceKey>
SUPABASE_JWT_SECRET=<JwtSecret>
OPENROUTER_API_KEY=
AUTOBRICKS_MODEL=autobricksai/gpt-4.1-mini
EMBEDDINGS_URL=https://api.autobricksai.com/v1/embeddings
EMBEDDINGS_MODEL=autobricksai/text-embedding-3-small
EMBEDDINGS_KEY=
EMBEDDINGS_DIMS=
SECRET_KEY=<SecretKey>
ADMIN_EMAILS=<the user's email address>
PUBLIC_URL=http://127.0.0.1:<PORT>
POLL_SECONDS=0
WEB_WORKERS=1
WA_AUTH_DIR=/opt/arlo/.wa-auth
OCR_CACHE_DIR=/opt/arlo/.ocr-cache
OCR_LANGS=eng
OCR_MAX_PAGES=20
```

`compose.yml` — four services and a one-shot that seeds the database's first-boot scripts out
of the Arlo image. Use the release-approved digests exactly as written; never resolve a
mutable tag during setup:

```yaml
name: arlo

services:
  seed:
    image: ghcr.io/adit-firdaus/arlo-ai-selfhost@sha256:24e0e4aeaf0852e35f5adc51f55f5269444754423b554ded8ca1769f777c090c
    user: root
    command: sh -c 'cp -a /opt/arlo/initdb/. /seed/'
    volumes: [initdb:/seed]
    restart: 'no'

  db:
    image: pgvector/pgvector@sha256:cf134a767f474095eeba57e0117be8e568e011a63f33fbf252f14c9b760f8e6f
    restart: unless-stopped
    env_file: [db.env]
    depends_on:
      seed: { condition: service_completed_successfully }
    volumes:
      - pgdata:/var/lib/postgresql/data
      - initdb:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U postgres -d postgres']
      interval: 5s
      timeout: 5s
      retries: 12

  auth:
    image: supabase/auth@sha256:1736a63078f5922b198c4cbe50f80ab9a2d3b54fe8b7b6cfb2e9dc5dbbc12c6b
    restart: unless-stopped
    env_file: [auth.env]
    depends_on:
      db: { condition: service_healthy }
    healthcheck:
      test: ['CMD', 'wget', '--no-verbose', '--tries=1', '--spider', 'http://localhost:9999/health']
      interval: 5s
      timeout: 5s
      retries: 12

  authgw:
    image: nginx@sha256:65645c7bb6a0661892a8b03b89d0743208a18dd2f3f17a54ef4b76fb8e2f2a10
    restart: unless-stopped
    depends_on:
      auth: { condition: service_healthy }
    expose: ['8000']
    command:
      - sh
      - -c
      - |
        cat > /etc/nginx/conf.d/default.conf <<'CONF'
        server {
          listen 8000;
          location /auth/v1/ {
            proxy_pass http://auth:9999/;
            proxy_set_header Host $$host;
            proxy_set_header X-Forwarded-For $$proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $$scheme;
          }
          location = /health { proxy_pass http://auth:9999/health; }
          location / { return 404; }
        }
        CONF
        exec nginx -g 'daemon off;'

  app:
    image: ghcr.io/adit-firdaus/arlo-ai-selfhost@sha256:24e0e4aeaf0852e35f5adc51f55f5269444754423b554ded8ca1769f777c090c
    restart: unless-stopped
    env_file: [.env]
    depends_on:
      db: { condition: service_healthy }
      authgw: { condition: service_started }
    volumes:
      - uploads:/opt/arlo/uploads
      - waauth:/opt/arlo/.wa-auth
      - ocrcache:/opt/arlo/.ocr-cache
    ports: ['127.0.0.1:<PORT>:9005']
    healthcheck:
      test: ['CMD', 'node', '-e', "fetch('http://127.0.0.1:9005/').then(r=>process.exit(r.status<500?0:1)).catch(()=>process.exit(1))"]
      interval: 30s
      timeout: 10s
      start_period: 300s
      retries: 3

volumes:
  pgdata:
  uploads:
  waauth:
  ocrcache:
  initdb:
```

## 5. Start it, and get the user their account

```powershell
Set-Location $AppDir
docker compose pull
docker compose up -d --wait --wait-timeout 600
```

`--wait` holds until every healthcheck passes. On the first run the app container applies
sixty-odd migrations before it serves; `docker compose logs -f app` shows them going in. If
`--wait` times out while the log is still applying migrations, that is patience, not failure —
poll again. If the app container restarts repeatedly, read its log: a migration that fails
stops the boot deliberately rather than serving against a half-built schema.

Then issue the first registration code. Registration is invitation-only the moment
`ADMIN_EMAILS` names anybody, and only an administrator can invite — the way out of that loop
is this command, which is the reason `scripts/` is in the image:

```powershell
docker compose exec app node scripts/invite.mjs <the user's email address>
```

It prints a code. Put the code and the local URL in `creds.txt` — never a password, and never
the AutoBricks key — restrict the file to the current user, then open Chrome at
`http://127.0.0.1:<PORT>/login?code=<the code>` — there is no `/register` route; the code is
what turns the sign-in page into a registration one — give the user their code, and ask them to
choose their own password. Wait for them. You must never type a password on a person's behalf.

### Install the key, now that there is a workspace to install it into

A workspace exists only once somebody has registered, which is why this happens here and not in
section 4. Both halves, in order:

```powershell
# 1. Embeddings, instance-wide. Write the key into .env and restart the app.
(Get-Content .env) -replace '^EMBEDDINGS_KEY=.*', "EMBEDDINGS_KEY=$env:ABAI_KEY" |
  Set-Content -Encoding utf8 .env
docker compose up -d --wait app

# 2. Chat, per workspace. setProvider seals the key into the database under SECRET_KEY —
#    the same path the Settings page uses, so nothing here is a special case.
docker compose exec -T -e ABAI_KEY=$env:ABAI_KEY app node --input-type=module -e 'const {setProvider}=await import("/opt/arlo/src/server/auth.ts");const {asSystem,one,close}=await import("/opt/arlo/src/server/db.ts");const ws=await asSystem(()=>one("SELECT id FROM workspaces ORDER BY id LIMIT 1"));await asSystem(()=>setProvider(ws.id,"autobricks",process.env.ABAI_KEY));console.log("key saved for workspace",ws.id);await close();'
```

The second command prints `key saved for workspace <id>`. Anything else — an import error, no
workspace row — means the account in the step above did not actually land; go back rather than
carrying on.

Confirm the account landed, without reading anything private:

```powershell
docker compose exec -T db psql -U postgres -d postgres -At -c "select count(*) from auth.users"
```

A count of `1` or more is the account. The key is already installed by the step above, so confirm rather than ask:

```powershell
docker compose exec -T db psql -U postgres -d postgres -At `
  -c "select llm_provider from arlo.workspaces limit 1"
```

`autobricks` is the answer you want. The key itself is encrypted in that table under
`SECRET_KEY` and neither you nor the psql output can read it, which is the intended behaviour.

## 6. Coordinator verification and finish

Run every check directly in the persistent coordinator shell. On any real failure, update the
snapshot, make the repair, retry the failed operation, and rerun the whole checklist. Don't
report the install complete while an item is pending — a half-verified handoff sends the user
away with a workspace that may not work.

Required successful outcomes:

- `[x]` `docker info` and `docker compose version` both exit `0`
- `[x]` `docker compose ps` shows `seed` exited `0` and `db`, `auth`, `authgw`, `app` running,
  with `db`, `auth` and `app` healthy and their restart counts stable across a recheck
- `[x]` `docker compose port app 9005` publishes on `127.0.0.1` and nothing else does
- `[x]` `select count(*) from arlo.schema_migrations` is greater than zero
- `[x]` `GET http://127.0.0.1:<PORT>/` returns a status under `500` and the page is Arlo's
  sign-in
- `[x]` The identity provider answers: `docker compose exec -T authgw wget -qO- http://auth:9999/health`
- `[x]` `select count(*) from auth.users` is at least `1` — the user's own account exists
- `[x]` `select llm_provider from arlo.workspaces limit 1` is `autobricks`, and
  `select autobricks_key_enc is not null from arlo.workspaces limit 1` is `t`
- `[x]` `EMBEDDINGS_KEY` in `.env` is not empty, and the app container has been restarted since
  it was written
- `[x]` `compose.yml`, `.env`, `db.env`, `auth.env` and `creds.txt` exist in `$AppDir` and are
  restricted to the current Windows user

Connectors, a public domain and TLS are intentionally not on this checklist — they are out of
scope here, so there is no need to test them or flag them as missing.

After all pass:

1. Update `setup-progress.json` to `phase=complete`, `status=complete`.
2. Close the dependency subagent if you opened one, and close both shells.
3. Clear `$env:ABAI_KEY` from the coordinator shell.
4. Open Chrome at `http://127.0.0.1:<PORT>`.
5. Report only: the local URL, the install directory, the recovery command (`docker compose up -d`
   from `$AppDir`), the backup line below, and one short sentence saying that uploading
   documents, building the first bot and connecting WhatsApp or Telegram are the next course
   steps, done from inside Arlo.

Tell the user this once, in plain words, as part of the handoff: **the three env files in that
directory are the only copy of the key that decrypts what Arlo stores. Copy that folder
somewhere safe. A `docker compose down -v` destroys the database.**

That is the end of this runbook.

## CDN release

Three of the four images are pinned above, by the index digest rather than a per-architecture
one, so the same line works on a student's amd64 laptop and on an arm64 machine.

`ghcr.io/adit-firdaus/arlo-ai-selfhost@sha256:24e0e4aeaf0852e35f5adc51f55f5269444754423b554ded8ca1769f777c090c` is the one still to fill, and it cannot be filled until the application
image is published. Two things, in this order:

1. Publish it. The build lives in the application repository, on the `af-public-compose`
   branch; the digest to pin is the one its run reports, or
   `docker buildx imagetools inspect ghcr.io/autobricks-ai/arlo-ai:main` afterwards.
2. **Make the `ghcr.io/autobricks-ai/arlo-ai` package public.** It belongs to a private
   repository, so it is private by default, and a student's `docker compose pull` fails with an
   authentication error that looks nothing like the permission problem it is. There is no API
   for this — it is Package settings → Change visibility, by hand.

Until both are done this runbook cannot complete on a machine that has never built the image,
and saying so here is cheaper than a student finding out at step 5.

The repository copy of this stack is
[adit-firdaus/arlo-ai-selfhost](https://github.com/adit-firdaus/arlo-ai-selfhost) —
`compose.yml` there is this same stack with its reasoning attached, and `bootstrap.sh` does
section 4 in one command for anyone who can clone it. Keep the two in step: a change to one is
a change to both.
