# Arlo, self-hosted: the full setup

Everything, from an empty machine to a bot answering customers on the channel they turned up
on. Section 1 is the install and takes about ten minutes, most of it waiting; the rest is
configuration you can do in any order, on any day after.

Work down it as written — every section assumes the ones above it.

`SETUP.md` beside this file is the *other* audience: an agent installing the basic stack on a
course student's laptop, scoped to "it runs and you can sign in". If that is what you want,
start there and come back here.

Two rules that decide most of what follows.

**`PUBLIC_URL` is the address people type, and everything is built from it.** OAuth redirects,
webhook addresses, the widget snippet and the links in a customer's email all come out of that
one variable. If it is wrong, each of them is wrong in a different place, and the error surfaces
at the far end — in Meta's dialog, in Telegram's API, in somebody's inbox — where it reads as a
platform problem rather than a typo in your `.env`.

**Half the channels are push.** Messenger, Instagram, TikTok and the WhatsApp Cloud API are the
platform calling *you*: they need a hostname their servers resolve over HTTPS. `127.0.0.1` is
not one and neither is a laptop behind NAT. Baileys WhatsApp, Telegram polling and the mailbox
connectors are the other direction and work from anywhere. Nothing is broken if you skip
section 2 — you just get the pull half.

---

## 1. Install it

Arlo runs in Docker, so the install is the same four commands on Windows, macOS and Linux.
What differs is only what you install first and which terminal you type them into.

### What you need first

| | Install | Type the commands in |
|---|---|---|
| **Windows 10/11** | [Docker Desktop](https://www.docker.com/products/docker-desktop/) with the **WSL 2** backend (its installer sets this up), plus [Git for Windows](https://git-scm.com/download/win) | **Git Bash** or a **WSL** shell — *not* PowerShell or CMD |
| **macOS 12+** | [Docker Desktop](https://www.docker.com/products/docker-desktop/) (Apple Silicon or Intel), or OrbStack | Terminal |
| **Linux** | Docker Engine + the Compose plugin — `docker-ce` and `docker-compose-plugin` from [Docker's repository](https://docs.docker.com/engine/install/), not the distribution's older `docker.io` | Any shell |

Docker Engine 24 or newer, with Compose v2.20+ — `docker compose version` tells you. The stack
uses `--wait` and a run-once service that Postgres waits for, and older Compose has neither.
Give Docker about 4GB of memory and 2GB of disk.

**Windows: it must be Git Bash or WSL.** `bootstrap.sh` is a shell script and PowerShell cannot
run it. Both of those shells ship the four things it needs — `openssl`, `date`, `tr`, `printf` —
and Git Bash comes with Git for Windows, so there is nothing else to install.

**Linux: put yourself in the `docker` group** or every command below needs `sudo`:

```sh
sudo usermod -aG docker $USER    # then log out and back in
```

### Ask for three things, in this order

Collect all three before running anything. The install then goes from nothing to a working
instance without stopping to wait for a person — which is the whole point of asking now.

**1. The AutoBricks API key.** First, because it is the only one that sends somebody to a
browser:

> **https://creators.autobricksai.com/account/api-keys/keys** — sign in, create a key if there
> is none, copy it.

It is not needed until section 3, well after the install. Ask for it anyway: a key fetched now
costs a minute of waiting at the start, and a key fetched later costs a finished install sitting
idle while somebody goes looking for a login. Without it Arlo runs but declines to answer, which
is a working instance that does nothing anyone wanted.

**2. The administrator's email address.** Not a formality and not a placeholder: it becomes
`ADMIN_EMAILS`, the only account that can reach `/admin`, and the same address has to be the one
invited in step 4 below. If you are setting Arlo up on somebody else's behalf, it is *their*
address, not yours — and if you think you already know it, confirm it rather than using it.

**3. The address this instance answers on.** `http://127.0.0.1:9005` for a laptop, or a real
`https://` hostname if there is one. It becomes `PUBLIC_URL`, and OAuth redirects, webhook
addresses and widget links are all built from it. Deciding now is free; changing it later means
editing three lines across two files and restarting.

### The four commands

```sh
# 1. Get the files
git clone https://github.com/adit-firdaus/arlo-ai-selfhost.git arlo
cd arlo

# 2. Generate every secret. Once, and only once — see the warnings below.
#    Replace the email with the real address of whoever will administer this instance.
./bootstrap.sh http://127.0.0.1:9005 their.real.address@company.com

# 3. Start it
docker compose up -d --wait

# 4. Issue the first registration code, for that same address
docker compose exec app node scripts/invite.mjs their.real.address@company.com
```

That last command prints a code and the address to use it at:

```
  ARLO-XXXX-XXXX
Register at http://127.0.0.1:9005/login?code=ARLO-XXXX-XXXX
```

Open it, choose your own password, and you are in. There is no separate registration page — the
code is what turns sign-in into registration.

### Four things about those four commands

The first `docker compose up` pulls about 120MB and then applies eighty migrations to an empty
database before it serves anything, so give it a few minutes; `docker compose logs -f app` shows
them going in. `bootstrap.sh` refuses to run twice, and that refusal is protecting you —
rerolling the secrets orphans the database volume and makes everything already stored in it
unreadable. `docker compose down` is safe, while `docker compose down -v` destroys the database;
it is the only command in this file that loses anything. And until you finish section 3, Arlo
runs but declines to answer rather than inventing anything, which is the designed behaviour and
not a broken install.

### When the install itself will not go

Docker problems look like Arlo problems and are not. In rough order of how often they happen:

| What you see | What to do |
|---|---|
| `docker: command not found` | Docker is not installed, or on Windows you are in a shell that cannot see it — reopen Git Bash after installing Docker Desktop |
| `Cannot connect to the Docker daemon` | Docker Desktop is not running (Windows, macOS) — start it and wait for the whale to settle. On Linux: `sudo systemctl start docker` |
| `permission denied ... /var/run/docker.sock` | Linux, and you are not in the `docker` group — see above, and log out and back in for it to take |
| `WSL 2 installation is incomplete` | Windows — run `wsl --install` in an admin PowerShell, reboot, then start Docker Desktop again |
| Docker Desktop will not start at all | Windows — virtualization is off in the BIOS/UEFI. Enable Intel VT-x or AMD-V. Task Manager → Performance → CPU shows "Virtualization: Enabled" when it is on |
| `./bootstrap.sh: bad interpreter` or `\r: command not found` | The file was saved with Windows line endings. `git clone` does not do this; an editor did. Re-clone, or run `sed -i 's/\r$//' bootstrap.sh` |
| `bind: address already in use` on 9005 | Something else holds the port. Stop it, or change both `ports:` in `compose.yml` and the port in `PUBLIC_URL` |
| `--wait` times out while the log still scrolls migrations | Not a failure. The first boot is slow — run `docker compose up -d --wait` again and it will pass |
| The app container restarts over and over | Read `docker compose logs app`. A failed migration stops the boot deliberately rather than serving against a half-built schema |
| Everything is very slow, or containers are killed | Docker has too little memory. Docker Desktop → Settings → Resources, give it 4GB |
| `no matching manifest for ...` | Not possible with this image — it is published for `linux/amd64` and `linux/arm64`, so Apple Silicon needs no Rosetta and no `--platform` flag |

## 2. A public address

Skip this if Arlo is only ever going to answer you on the machine it runs on — a laptop
demonstration needs none of it. Everything in section 5 that a platform has to *call*, however,
starts here.

Arlo publishes on `127.0.0.1:9005` and terminates no TLS of its own. Put a proxy in front.
Caddy, if you have nothing:

```
arlo.example.com {
    reverse_proxy 127.0.0.1:9005
}
```

Then set the address in **all three** places that must agree, and restart:

Open the two files in any editor and change three lines:

```
.env       PUBLIC_URL=https://arlo.example.com
auth.env   GOTRUE_SITE_URL=https://arlo.example.com
auth.env   GOTRUE_URI_ALLOW_LIST=https://arlo.example.com/*
```

Then restart:

```sh
docker compose up -d --wait
```

`https://`, no trailing slash, and the hostname a browser actually reaches. A mismatch between
`PUBLIC_URL` and `GOTRUE_SITE_URL` does not fail loudly: sign-in redirects land somewhere the
allow-list refuses, and the symptom is a login that returns to the login page.

Check it from outside your own network before going on — a proxy that works on the box and not
from the internet fails every push channel below, several hours later, silently.

## 3. The model key

This is the key collected in section 1 — if it was not, fetch it now from
**https://creators.autobricksai.com/account/api-keys/keys** and expect to wait while somebody
signs in.

One key does both halves. `bootstrap.sh` has already pointed the instance at AutoBricks — `EMBEDDINGS_URL` and
`EMBEDDINGS_MODEL` are filled in, and chat needs no endpoint at all because Arlo speaks
`api.autobricksai.com` natively. Only the key itself is missing.

**Chat** is per workspace, so it goes in the app: **Settings → the model provider section**,
paste, save. It is encrypted into the database under `SECRET_KEY`, which is why two workspaces
on one instance can pay their own way.

**Embeddings** are instance-wide. Put the same key in `.env` and restart:

```
EMBEDDINGS_KEY=abai_sk_live_…
```

```sh
docker compose up -d --wait app
```

Both are worth doing. Without the chat key the bot declines to answer; without the embeddings
key retrieval is keyword-only, so a question phrased unlike the document that answers it will
miss.

**Why that model and not the better one.** `EMBEDDINGS_MODEL` is
`autobricksai/text-embedding-3-small`, which returns 1536-wide vectors — exactly what
`migrations/003_kb.sql` declares. `text-embedding-3-large` returns 3072 and Arlo refuses it
rather than letting pgvector fail mid-transaction. Changing the width later is not a config
edit: it means a schema change and re-embedding everything you have stored.

**If you would rather use OpenRouter**, it works the same way — put your key in
`OPENROUTER_API_KEY`, set `EMBEDDINGS_URL=https://openrouter.ai/api/v1/embeddings` and
`EMBEDDINGS_MODEL=openai/text-embedding-3-small`, and leave the AutoBricks provider unset. The
two are independent: chat on one, embeddings on the other, if that is what your billing wants.

## 4. The knowledge base

Upload the documents first and connect a channel second. A bot with nothing to cite hands every
conversation to a person, which is correct behaviour and a poor demonstration.

- **Documents.** PDF, Word, text, spreadsheets. A scanned PDF has no text layer, so Arlo reads
  the pixels with Tesseract locally — `OCR_LANGS=eng` by default, `eng+chi_sim` for two, blank
  to switch it off. The first scan downloads about 15MB of language data into the `ocrcache`
  volume; it costs a few seconds a page and nothing leaves the machine.
- **A website.** The crawler refuses every host by default rather than becoming an open proxy
  into whatever your server can reach. Name the ones it may fetch:

  ```
  SCRAPE_ALLOW_HOSTS=example.com,www.example.com,docs.example.com
  ```

  ```sh
  docker compose up -d --wait app
  ```

  Subdomains are not implied. List each one.

## 5. The channels

Each of these is a registration you make against your own instance. Set the variables in `.env`
and `docker compose up -d --wait app` — the connectors page reads them at boot, and a blank pair
is a channel that reads "not set up" rather than one that half-works.

### Web widget — nothing to configure

Already on. **Bots → your bot → the widget snippet**, paste it into the page before `</body>`:

```html
<script src="https://arlo.example.com/api/widget.js?key=YOUR_WIDGET_KEY" defer></script>
```

One tag. No build step, no npm package, no CORS to configure. The script reads its own `src` for
the key and the origin, mounts into a shadow root so the page's CSS and the widget's never meet,
and polls `/api/chat` every two seconds while open.

**A workspace can have several**, one per site — named, revocable on its own, and recognisable
when a question arrives. Add them under **Connectors → Web chat**.

#### If you are building the site as well

Someone who asks for a website *and* has Arlo running wants the two joined up, not a snippet
pasted into a handover note. Do it for them, in this order, and do not stop to ask for a key —
you can mint one:

**1. Create the widget and take its key.** Named after the site, so the workspace can tell its
sites apart later:

```sh
docker compose exec -T app node --input-type=module -e 'const {add}=await import("/opt/arlo/src/server/channels/web.ts");const {asSystem,one,close}=await import("/opt/arlo/src/server/db.ts");const ws=await asSystem(()=>one("SELECT id FROM workspaces ORDER BY id LIMIT 1"));const w=await asSystem(()=>add(ws.id,"THE SITE NAME"));console.log(JSON.stringify({widget:w.id,key:w.key}));await close();'
```

It prints the id and the key. That is the same row the Connectors page would have created —
`add()` is the function behind that button — so nothing here is a special case to be undone
later.

**2. Put the tag in the page**, before `</body>`, with that key and this instance's
`PUBLIC_URL` as the origin. Not `127.0.0.1` unless the site is only ever opened on this machine:
the script calls back to whatever origin it was loaded from, so a laptop address in a page
served anywhere else is a widget that loads and never answers.

**3. Prove it answers before saying it works.** Load the page, open the chat, ask something the
site itself claims — a price, an opening time, a date from its own table — and read the reply.
The failure modes are quiet and specific:

| What comes back | What it means |
|---|---|
| `Unknown widget key` in a grey system line | the key in the tag does not match the row you just made |
| The launcher never appears | the script did not load — check the origin in `src` and the browser console |
| *"I couldn't find information about…"* | the widget works and the **knowledge base is empty of that**. Section 4 is the fix, not the tag |
| *"A person is picking this up"* | also working — the bot escalated rather than inventing. Correct, and worth explaining to whoever is watching |

That third row is the one to expect on a site you have just written: the bot answers from the
workspace's documents, and a brand-new site is not in them. Upload the page's own content — the
courses, the prices, the FAQ — and ask again before concluding anything is broken.

**4. Hand over the key with the site.** It sits in the page source, so it is public by design,
but it is also the thing to delete if the site is ever taken down.

### Telegram — works anywhere

Talk to `@BotFather`, `/newbot`, paste the token into the connector card. Arlo tries a webhook
first and falls back to polling by itself. On a laptop or behind NAT, skip the attempt that
cannot succeed:

```
TELEGRAM_POLLING=1
POLL_SECONDS=60
```

`POLL_SECONDS=0` disables the mail sweep, reminders and the Telegram long poll together — it is
how a process declares it does no background work at all, so leave it at 60 anywhere you want
either.

**Restart the app after connecting a bot**, and after each further one:

```sh
docker compose restart app
```

Not superstition, and not optional. Connecting a bot happens inside an HTTP request, which in
production is a forked web worker — and the polling loop has to run in the cluster primary,
because two processes polling one bot each get the updates the other never sees. The worker
hands the job across, the hand-off is launched without being awaited, and a failed hand-off is
discarded silently. The result is a bot that connects cleanly, whose connector card says
"receiving by polling", and which receives nothing at all until the next restart. Arlo's own
source names this exact symptom as a bug it fixed once; it still reproduces. Twenty seconds of
restart is the whole remedy.

### WhatsApp, the Baileys way — works anywhere

Drives a real handset over a socket, needs no public address and no Meta account. Press Connect,
scan the QR with the phone. Credentials land in the `waauth` volume, one directory per number.

Two things bite. **Never run two processes against one session** — the same credentials opened
twice is what WhatsApp closes the connection over, and you lose both. And it is an unofficial
client: it is the fastest way to a working number and the least durable one.

### WhatsApp, the official Cloud API — needs a public address

Meta's own, reached through Kapso, and push. Both variables blank switches it off:

```
KAPSO_API_KEY=
KAPSO_WEBHOOK_SECRET=
```

Webhook URL at Kapso: `{PUBLIC_URL}/api/whatsapp/cloud`. The secret is what proves an arriving
body is really theirs, checked as an HMAC over the raw bytes. The key is the whole Kapso
account rather than one number, and every workspace here becomes a customer under it.
`KAPSO_BILLING_MODE=customer_managed` — the default — keeps a business's messaging costs on
their own Meta account, which is the honest setting; `partner_managed` moves them onto yours and
is a pricing decision, not a configuration one.

### Messenger, Instagram and TikTok — need a public address

Three OAuth connectors, one app each, registered once for the instance; a workspace then
consents per account. **The three apps are genuinely separate products and their credentials
cannot be substituted for one another** — Instagram Login is not Facebook Login, and TikTok
Business Messaging is not TikTok Shop. Using the wrong pair fails at the consent screen with no
useful message.

| | Redirect URI | Webhook | Variables |
|---|---|---|---|
| Messenger | `{PUBLIC_URL}/api/oauth/facebook` | `{PUBLIC_URL}/api/facebook/webhook` | `FB_APP_ID`, `FB_APP_SECRET`, `FB_VERIFY_TOKEN` |
| Instagram | `{PUBLIC_URL}/api/oauth/instagram` | `{PUBLIC_URL}/api/instagram/webhook` | `INSTAGRAM_APP_ID`, `INSTAGRAM_APP_SECRET`, `INSTAGRAM_VERIFY_TOKEN` |
| TikTok | `{PUBLIC_URL}/api/oauth/tiktok` | `{PUBLIC_URL}/api/tiktok/webhook` | `TIKTOK_CLIENT_KEY`, `TIKTOK_CLIENT_SECRET` |

The verify token is a string you choose; the platform echoes it back once, when the webhook is
first subscribed. The redirect URI is matched character for character — scheme, host, path, and
no trailing slash.

**The Meta trap, which costs an afternoon every time.** Meta needs *two* subscriptions and only
one of them is Arlo's job. Arlo subscribes the page when you consent
(`POST /{page-id}/subscribed_apps`). The **app-level** subscription — `POST
/{app-id}/subscriptions`, which fields the app wants at all — is dashboard work nobody does,
and with zero fields subscribed Meta sends nothing. The callback verifies, the dashboard turns
green, the page-level call reports success, and no message ever arrives. Subscribe the fields
at the app level too: `messages`, `message_deliveries`, `message_echoes`, `message_reads`,
`standby`, `messaging_handovers` for Messenger; `messages`, `message_reactions`,
`messaging_seen` for Instagram.

TikTok delivers inbound only to business accounts outside the EEA, Switzerland, the UK and the
US. That is TikTok's rule, not a setting — a connector that consents cleanly and stays quiet is
usually this.

### Email — works anywhere

Gmail and Outlook, by OAuth. Register one app per provider and put the pair in `.env`:

```
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
OUTLOOK_CLIENT_ID=
OUTLOOK_CLIENT_SECRET=
```

Redirect URI `{PUBLIC_URL}/api/oauth/gmail` and `{PUBLIC_URL}/api/oauth/outlook` — `gmail`,
not `google`, because the redirect is built from the connector's own name
(`src/server/oauth.ts`) and Google's console matches it character for character. A workspace can bring
its own app instead of using the instance's. Mailboxes are swept on the `POLL_SECONDS` timer,
so a `0` there is a mailbox that never gets read.

## 6. People

Registration is invitation-only the moment `ADMIN_EMAILS` names anybody. The first code has to
come from a shell, because every later one comes from an administrator and the first
administrator does not exist yet:

```sh
docker compose exec app node scripts/invite.mjs you@example.com
docker compose exec app node scripts/invite.mjs someone@example.com --days 30 --note "ops"
```

After that, **Admin → invitations**. A code admits one address once and expires in a fortnight
unless you say `--forever`.

Optional: Cloudflare Turnstile on the public waitlist form — `TURNSTILE_SITE_KEY` and
`TURNSTILE_SECRET_KEY`. It is the only thing in this app that loads a script from another
origin, and the content-security-policy is widened for that one page because of it.

## 7. Backups

The database is the product. The volumes beside it are a cache and a phone session.

```sh
docker compose exec -T db pg_dump -U postgres -Fc postgres > arlo-$(date +%F).dump
```

Back up `.env`, `db.env` and `auth.env` with it, **somewhere else**. `SECRET_KEY` in `.env` is
what every stored connector token and every workspace's model key is encrypted with: with the
dump and without that file, the rows are all still there and not one of them can be read.

Restoring onto a fresh stack: bootstrap it, stop the app, restore the dump, put the *old* three
env files back, start it.

```sh
docker compose stop app
docker compose exec -T db pg_restore -U postgres -d postgres --clean --if-exists < arlo-2026-09-16.dump
docker compose up -d --wait app
```

## 8. Upgrading

```sh
docker compose pull app && docker compose up -d --wait app
```

New migrations apply themselves at boot, before the first worker serves. If one fails the
container stays down and says why in `docker compose logs app` — a worker answering against a
schema that did not catch up is the worse outcome, so that is deliberate. Roll back by pinning
the previous digest in `ARLO_IMAGE` and starting again; note that a migration already applied is
not undone by running an older image.

## 9. When something is wrong

| What you see | What it actually is |
|---|---|
| Sign-in returns to the sign-in page | `PUBLIC_URL` and `GOTRUE_SITE_URL` disagree, or the allow-list does not cover the address |
| "Nobody is signed in", nothing in any log | `SUPABASE_JWT_SECRET` in `.env` and `GOTRUE_JWT_SECRET` in `auth.env` are not the same value. They are verified locally; a mismatch is silent by construction |
| `auth` restarting, `converting '' to type int` | A blank `GOTRUE_SMTP_*` value. GoTrue parses the port before anything else — absent is fine, empty is fatal |
| App container restarting on a fresh install | Read the log: a failed migration stops the boot on purpose |
| First start takes minutes | It is applying eighty migrations to an empty database. Expected, once |
| Consent fails immediately, no message | The redirect URI does not match exactly — scheme, host, trailing slash |
| A connector connects, then silence | Push channel on an address the platform cannot reach, or the Meta app-level subscription in section 5 |
| Telegram connected, card says "receiving by polling", nothing arrives | The polling loop was started from a web worker and never reached the cluster primary. `docker compose restart app`. Queued messages are not lost — Telegram holds them and delivers the moment the loop starts. `select poll_offset from arlo.telegram_bots` reading NULL is the tell: not one batch has ever been collected |
| Bot answers "I cannot answer that" to everything | No model key, or no documents to cite. Both are the honest failure, not a bug |
| Answers miss documents that obviously match | Embeddings are off — section 3 |
| A crawl fetches nothing | The host is not in `SCRAPE_ALLOW_HOSTS`, subdomains included |
| `fetch failed` on a connector that has correct credentials — and only *sometimes* | Node's address-family race. It gives the first family 250ms, and Telegram, Meta, Google and the model gateway are all 180-250ms away, so a late-but-fine handshake is discarded. `bootstrap.sh` sets `NODE_OPTIONS=--network-family-autoselection-attempt-timeout=3000` for this; if your `.env` predates that line, add it and restart the app |
| Telegram's Connect form says `fetch failed` | The same race — the token is validated with a live `getMe` before anything is stored, so the failure is the call, not the token. Also set `TELEGRAM_POLLING=1` whenever `PUBLIC_URL` is loopback: a webhook Telegram cannot reach fails on every connect attempt before polling takes over |
| The bot stops answering a thread even after the model key is set | That conversation was handed to a person — `status = human`, and the bot deliberately stays out of a thread somebody took over. Start a fresh one (the widget keeps `arlo.visitor` in `localStorage`; clearing site data starts a new conversation) rather than retrying in the same thread |

Logs, for any of it:

```sh
docker compose logs -f app
docker compose logs auth --tail 50
docker compose ps
```

## What is not here

TLS certificates, DNS, firewalling and the machine itself are yours. So is the decision to put
this on the public internet at all: Arlo holds a company's internal documents and its customers'
conversations, and a stack that is loopback-only behind a proxy you control is a smaller
problem than one that is not.
