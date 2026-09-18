# Standalone stack, as it ships

Runs the Agent from the published images, for the deployment topology where Home Assistant itself
runs in Docker and add-ons are not an option. This is a production target, not a development
scaffold. See [ADR-0004](../../docs/adr/0004-internal-command-channel.md).

The development loop lives in the repository root, `compose.yml` behind a `Makefile`, and is a
different thing: it mounts the working copy into the runtime and watches it. Here the images are
built ahead of time, the TypeScript is compiled, the PHP dependencies are installed at build time,
and nothing from the host is mounted, so what runs is the artifact.

## Nothing is published yet

**The image tags in `compose.yml` resolve nowhere.** No image has been pushed to `ghcr.io` and no
version tag has been cut, so `docker compose pull` fails and a plain `docker compose up` fails with
it. That is the honest state of this milestone: the packaging is written and the pipeline that
would publish it is run by nothing in this branch.

What works today is building the three images from this checkout, which is one extra file:

```bash
cd deploy/standalone
cp .env.example .env   # then fill it in, see below
docker compose -f compose.yml -f compose.build.yml up -d --build
docker compose logs -f
curl localhost:8080/health
```

`compose.build.yml` adds one build context per service and changes nothing else, so what it builds
is tagged exactly as what would have been pulled and every other setting is the shipped one. The
rest of this file reads the same either way, and the day the tags resolve the second `-f` is what
drops off.

## Installing it, with Home Assistant in Docker

`compose.yml` and `.env.example` are the whole install. Copy the pair into a directory of its own,
anywhere, and fill in the copy: neither file points back into this repository, which is why the
build contexts are in a separate file rather than in the one people take away.

```bash
mkdir -p /opt/domely && cd /opt/domely
# put compose.yml and .env.example here, then
cp .env.example .env
```

Three values need a decision. Everything else in `.env.example` has a default that suits a house
and says on itself what it does.

### `HA_URL`, the address of Home Assistant

As reachable **from inside this stack's containers**, which is not the same address your browser
uses. Never `localhost`: inside a container that is the container.

| Where Home Assistant runs | What `HA_URL` is |
| --- | --- |
| A container on this same Docker host | `http://<the host's LAN address>:8123`. The container name works instead only if you attach this stack to Home Assistant's network |
| Another machine on the LAN | `http://<that machine's address>:8123` |

Whichever it is, `docker compose logs adapter` says which of the two went wrong on a first start:
a connection that never arrives is the address, and a connection that is answered and then refused
is the token.

### `HA_TOKEN`, a long-lived access token

Home Assistant issues these itself, and shows each one exactly once:

1. Open Home Assistant and click your own name at the bottom of the sidebar.
2. Open the **Security** tab.
3. Scroll to **Long-lived access tokens** and click **Create token**.
4. Name it something you will recognise in a year, `Domely` for instance.
5. Copy what it shows into `HA_TOKEN` in `.env`, in one go. It is never shown again, and a token
   you lost is revoked and replaced rather than recovered.

The token carries the permissions of the account that created it, so create it under an account
that may do what Domely is going to publish and no more. Deleting it in that same screen is what
cuts this Agent off from the house, immediately and without touching anything here.

**This is not a development shortcut.** Only an add-on is handed a token by the supervisor, and a
large part of the Home Assistant community runs Home Assistant in Docker and cannot install add-ons
at all. The long-lived token path is permanent, and the adapter is not allowed to care which of the
two it got: one resolver in `agent/adapter/src/config.ts` decides, and nothing downstream asks.

### `ADAPTER_SECRET`, the shared secret between the two processes

The internal command channel between the API layer and the adapter (ADR-0004). Both processes read
this one variable, neither starts without it, and compose refuses to bring the stack up at all when
it is unset. Generate one and paste it:

```bash
openssl rand -hex 32
```

`.env` is gitignored and holds a credential with full access to your home. It never enters the
repo, a commit, a screenshot or a log line. Only the adapter is given `HA_TOKEN`: the API layer has
no business holding a credential to the house.

### Pairing

`CLOUD_URL` is what turns a local Agent into a reachable one. Leave it empty and the Agent mirrors
state as it always has and serves nothing publicly. Set it, start the stack, and the adapter prints
a pairing code in the log for an administrator to redeem in the app. `CORS_ORIGINS` has to name the
same cloud or the app gets nothing back from this Agent, for the reason below.

After pairing, the app proposes the house: every lamp, switch and thermostat Home Assistant already
has an area for, read straight from the mirror and grouped by that area, ticked and ready to publish
in one tap. Nothing is published until the administrator confirms it, and a candidate Home Assistant
placed in no area is offered too, unticked, under a group of its own.

## Three services and two volumes

| Service | What it is | Exposed |
| --- | --- | --- |
| `adapter` | Node, holds the WebSocket to Home Assistant | `8099` on the compose network only |
| `api` | Laravel on FrankenPHP, the public API | `127.0.0.1:8080`, loopback only |
| `tunnel` | `cloudflared`, the way in from outside | Nothing. It dials out |

The adapter's port is its internal command channel (ADR-0004), which is how the API layer executes
a service call. It is in no `ports:` mapping, so it is reachable by service name from the other
container and from nothing outside this stack, and `ADAPTER_SECRET` is required on every request
because anything else on that Docker network could otherwise drive the house. The add-on topology
keeps the tighter default and binds it to `127.0.0.1`; here it binds `0.0.0.0` because the two
processes are two containers.

`API_PORT` moves the published port, `API_THREADS` sizes FrankenPHP's thread pool: one is held per
open event-stream connection from M4 onwards, so it is the number of phones in the house rather
than a constant. See [ADR-0005](../../docs/adr/0005-realtime-event-stream.md).

**`CORS_ORIGINS` has to be set or the app does not work.** The app is served by Domely Cloud and
this Agent answers on its own tunnel hostname, so every call the app makes is cross-origin and the
browser discards every answer from an origin this list does not name. Empty is the default because
an Agent that has not been told where its app is served from should let no page anywhere read the
house, and it is the reason a stack that is paired, healthy and answering `curl` perfectly can
still show nothing at all in a browser. Set it to the cloud's own address, comma separated if
there is more than one, and never `*`: this hostname is public and the token lives in the app's
storage.

The rest of what the API layer gained in M4 has defaults that are right for a house and is
documented in `.env.example`: `EVENTS_RETENTION` and the three `EVENT_STREAM_*` values, which the
adapter's side of the push (`API_URL`, `EVENT_PUSH_TIMEOUT_MS`) feeds. `AUTOMATION_CAP` joined them
in M9, on the same terms: a default that is right for a house, documented there, and reaching the
container because it is documented as tunable.

**`API_URL` is more than the push's address since M9.** An automation's action goes to the API
layer through it too, because the engine runs in the adapter and deliberately does not call Home
Assistant itself ([ADR-0019](../../docs/adr/0019-the-automation-engine.md)). So a blank one is not
a dropped state change that the next read repairs: no automation can act at all, and a house that
quietly does none of what its owner was told it would is the failure worth reading the log for. The
adapter says so on every attempt rather than once at startup.

Loopback is deliberate. Every API route except `/health` requires a valid JWT, with no bypass in
any environment, and the way in from outside is the tunnel rather than a published port.
Publishing it more widely, or putting a reverse proxy in front, adds a way in that the tunnel's
allow-list does not cover.

The `tunnel` service terminates at the `api` service and nothing else. Its ingress lives on
Cloudflare's side, written by the cloud when the home was paired (#34), so this stack holds no
credential that could rewrite it, and it is allow-list shaped: one rule for this home's hostname
ending in a catch-all that refuses. Home Assistant and everything else are excluded by
construction rather than by someone remembering to exclude them. Three paths on this home's own
hostname are refused ahead of the rule that serves: `GET /health`, because it answers without a
token by design; `/internal`, both directions the adapter and the API layer speak over, which the
shared secret authenticates and no caller outside this stack has any business reaching; and
`/ingress`, the page Home Assistant's sidebar opens in the other topology, which is admitted by where the connection
came from rather than by a token ([ADR-0010](../../docs/adr/0010-ingress-is-a-deep-link.md)). One
cloud writes the same ingress for both topologies, so that third rule is here too, refusing a route
that would have refused the caller itself.

The admin API is not one of the refusals. The administrator reaches the admin screens through the
same app everybody else uses, served by the cloud, and those screens call `api/admin/*` here with
an administrator token that the Agent checks against the role in its grant set
([ADR-0009](../../docs/adr/0009-where-the-admin-ui-lives.md)). So it goes over the tunnel like
every other route, and the token and the role check are what stand in front of it.

It joins the `api` container's network namespace, which is how `cloudflared` reaches the API layer
at `127.0.0.1:8080`: the same address it has in the add-on, where the three processes share one
container anyway. One cloud provisions the ingress for both topologies from one
`CLOUD_TUNNEL_SERVICE`, so that address has to mean the same thing in each.

The cost is that `api` has to be **recreated** and never merely restarted. `docker compose up`
recreates `tunnel` along with it; `docker compose restart api`, `docker restart api` and a
restart-policy restart after a crash do not, and they leave `tunnel` running in a network
namespace that no longer exists, where `cloudflared` retries forever instead of exiting. That is
why `run-tunnel.sh` watches `TUNNEL_HEALTH_URL` and exits after a minute of silence: the exit is
what lets `restart: unless-stopped` rejoin the live namespace without anybody noticing first.

It raises nothing until this Agent is paired: the credentials arrive with pairing and it waits for
them, so a stack that has never been paired is publicly reachable at no address at all. A tunnel
that is down takes nothing else with it, `cloudflared` reconnects on its own, and the LAN keeps
working throughout. See [`agent/tunnel/README.md`](../../agent/tunnel/README.md).

The adapter and the API mount one named volume at `/var/lib/domely`, holding the shared SQLite
file: the adapter writes the state mirror and the API owns the publish configuration. A second
volume at `/var/lib/domely-tunnel` holds the tunnel credentials, and it is the only one `tunnel`
mounts. That split is deliberate. `tunnel` is the one process in this stack reachable from the
internet, and `domely.sqlite` holds the agent token in plaintext, the grant set and every Home
Assistant entity ID, which is the mapping principle 1 keeps inside the Agent; all three containers
run as uid 1000, so `:ro` on the data volume would not have kept it out of that file. `cloudflared`
needs exactly one file, and `TUNNEL_CREDENTIALS_PATH` names it in one place for all three services.

All three containers running as uid 1000 is also what keeps either volume free of a file another
service cannot read or write. Back up `agent-data` and nothing else: everything in it except the
publish configuration is reconstructible from Home Assistant, and the credentials on the other
volume come back with a re-pair. See [`docs/data-model.md`](../../docs/data-model.md).

**Losing `agent-tunnel` while `agent-data` survives strands this home, quietly.** The adapter
writes that file once, at the moment it pairs, and nothing rewrites it afterwards: the identity in
`agent-data` is what stops pairing running a second time. So an Agent that keeps its database and
loses its credentials stays paired, never asks for them again, and leaves `tunnel` sitting on
`this Agent is not paired yet` for as long as it runs, with no error anywhere and the LAN still
working. That is the case to watch when the volume is renamed, moved to another host, or recreated
by anything that prunes volumes. If the file still exists on a volume of its own, put it back
before starting the stack:

```sh
docker run --rm -v <the old volume>:/from -v standalone_agent-tunnel:/to alpine \
  cp -a /from/tunnel-credentials.json /to/tunnel-credentials.json
```

If it is gone, the only way back is `docker compose exec api php artisan domely:unpair` and
redeeming a fresh code at the terminal, and everybody loses access until that is done.

Both volume names are prefixed with the compose project, which is the name of the directory the
stack runs from: `standalone_agent-data` in this checkout, `domely_agent-data` under `/opt/domely`.
`docker compose config --volumes` says which, and `docker volume ls` confirms it.

## Upgrading

Each image is pinned to an exact version rather than to `latest`, so an upgrade is an edit somebody
made on purpose and never something a restart does by surprise. Put the new version in the three
tags in `compose.yml`, then:

```bash
docker compose pull
docker compose up -d
docker compose logs -f
curl localhost:8080/health
```

**The volumes are not touched, and that is the whole point.** `agent-data` holds `domely.sqlite`,
and `domely.sqlite` holds `devices` and `rooms`: the publish configuration, and the device UUIDs
every share in the cloud is written against. Keep that volume across an upgrade and every share
keeps working, every name and every room stays where the administrator put it, and nobody in the
house has to be told anything. `docker compose down` leaves named volumes alone. `docker compose
down -v` deletes them, which costs a re-pair and a re-publish for everybody, and there is no
version of that which is quick.

The API layer applies its migrations on every start, before FrankenPHP is exec'd, so a release that
changes the schema needs nothing but the new image. `domely:mirror-table` runs with them and
re-creates the state mirror when that cache is gone, which the migration alone cannot do once its
batch is recorded. A failing migration stops the container rather than being served over, and so
does a failing `domely:channel-check`: the internal channel is versioned, and two images from
different releases is exactly the case that check exists for. That check waits up to
`CHANNEL_CHECK_WAIT` seconds (10 by default) for the adapter, because nothing orders the two
containers. An adapter that stays silent for the whole window does not stop the start: the API
serves the mirror it has, and every command answers `503` until the adapter is up.

Pull all three images or none. They are released together and versioned together, and the channel
check is what turns a mismatched adapter and API into a container that stopped rather than into a
house that half works.

Rolling back is the same edit in reverse. A migration the older image does not know about is the
one thing that does not roll back with it, so a release whose changelog mentions one is a release
to take a copy of the volume before:

```bash
docker compose stop
docker run --rm -v standalone_agent-data:/from -v "$PWD":/to alpine \
  tar czf /to/agent-data-backup.tgz -C /from .
docker compose up -d
```

The same publish configuration comes out as readable JSON, onto the host rather than into the
volume it is a copy of:

```bash
docker compose exec -T api php artisan domely:export > publish.json
```

`domely:import` reads that back without ever reassigning a UUID, which is what makes it a restore
rather than a fresh publish, and it names every device whose entity is not in this Home Assistant
instead of quietly leaving the family with devices that cannot work. See
[`agent/api/README.md`](../../agent/api/README.md).

## What differs between the two topologies

Both are production targets, both run the same three processes, and neither is a lesser version of
the other. What actually differs is short, and everything not in this table is the same code
behaving the same way.

| | Add-on, on HAOS | Standalone, this stack |
| --- | --- | --- |
| How Home Assistant is reached | `http://supervisor/core`, with the `SUPERVISOR_TOKEN` the supervisor injects. The operator creates no token and can revoke none | `HA_URL` plus a long-lived access token, created by hand and revoked by hand |
| Which Home Assistant | The one it is installed in, always. There is no way to point it elsewhere, by design | Any Home Assistant `HA_URL` reaches, including one on another machine |
| Packaging | One container under s6-overlay, supervising the three processes inside it | Three containers on a compose network |
| Reaching the admin screens | A sidebar item, which opens one page the Agent serves that deep-links into the app (ADR-0010) | Open the app the cloud serves and go to the admin screens. There is no sidebar to put anything in |
| Ingress | `GET /ingress` is that page, served only to the supervisor's own address | Not used. The route is in the image and answers `404` to everything that is not the supervisor, and the tunnel refuses it at the edge as well |
| Where data lives | `/data`, the one directory the supervisor keeps across an update and captures in a backup | Two named volumes, `agent-data` and `agent-tunnel` |
| Backups | Home Assistant's own, with the add-on's `backup: hot` hook | Yours to take. `agent-data` is the one that matters |
| Updating | The store offers it when the version in `config.yaml` moves, and the supervisor does the rest | Edit the three tags, `docker compose pull`, `docker compose up -d` |
| The internal channel | `127.0.0.1:8099`: one network namespace, so loopback is enough | `adapter:8099` on the compose network, bound `0.0.0.0` and in no `ports:` mapping |

The credential difference is the one the code is not allowed to notice.
[ADR-0004](../../docs/adr/0004-internal-command-channel.md) has the comparison it was decided from,
and `agent/adapter/src/config.ts` is the one resolver that decides: everything downstream is handed
`{ url, token, source }` and never asks which topology it is in.

## The images

Three images, one per process, each a multi-arch manifest covering `linux/amd64` and `linux/arm64`:

| Image | From |
| --- | --- |
| `ghcr.io/raymonbb/domely-adapter` | [`agent/adapter/Dockerfile`](../../agent/adapter/Dockerfile) |
| `ghcr.io/raymonbb/domely-api` | [`agent/api/Dockerfile`](../../agent/api/Dockerfile) |
| `ghcr.io/raymonbb/domely-tunnel` | [`agent/tunnel/Dockerfile`](../../agent/tunnel/Dockerfile) |

The tag is the version in the repository root's [`VERSION`](../../VERSION), which is the single
source: [`CHANGELOG.md`](../../CHANGELOG.md), `deploy/addon/config.yaml` and the three tags in
`compose.yml` all have to agree with it.

The adapter's Dockerfile is multi-stage. The first stage installs the full dependency tree,
compiles `src/` to `dist/` and then prunes the tree to production only. The second copies just
those two results, so the toolchain, the TypeScript and the dev dependencies never reach the image
that ships. It runs as `node`, the unprivileged user the base image already provides. The adapter
writes no files of its own, so beyond the shared volume it needs nothing it owns.

The API's has the same shape: composer resolves the dependencies in a build stage, on the same PHP
version that will run them, and the final stage copies `/app` and nothing else, so composer, the
dev dependencies and the test suite stay out of the image. It runs as `domely`, uid 1000, and
writes only `storage/` and the shared volume. It serves plain HTTP on `:8080`, with no hostname, so
Caddy asks for no certificate and needs no privileged port: TLS terminates at the tunnel.

The tunnel's is the odd one out: it builds nothing. It puts Cloudflare's own `cloudflared` binary,
at a pinned version, on Alpine, because the image Cloudflare ships is distroless and has no shell
to wait for pairing with, and runs as a uid that could not read the `0600` credentials file the
adapter writes.

`restart: unless-stopped` covers what the adapter cannot: it survives a Home Assistant restart by
itself (see the adapter README), and a reboot of the host needs no hand on it either. `docker
stop` is a clean shutdown, the adapter catches `SIGTERM`, closes the socket and exits 0 well
inside the grace period.

Logs go to stdout and stderr, unbuffered and one line per event, so `docker compose logs -f` is
the whole interface. Nothing is written to a file inside the container.

### Building for another architecture

`docker compose -f compose.yml -f compose.build.yml build` builds for the architecture it is
running on: Apple Silicon and a Raspberry Pi are arm64, a Proxmox host is usually amd64. Both at
once needs a builder that is not the default one, and is what the release pipeline does:

```bash
docker buildx create --name domely --driver docker-container --use
docker buildx build --platform linux/amd64,linux/arm64 --build-arg BUILD_VERSION=0.1.4 -t ghcr.io/raymonbb/domely-adapter:0.1.4 agent/adapter
docker buildx build --platform linux/amd64,linux/arm64 --build-arg BUILD_VERSION=0.1.4 -t ghcr.io/raymonbb/domely-api:0.1.4 agent/api
docker buildx build --platform linux/amd64,linux/arm64 --build-arg BUILD_VERSION=0.1.4 -t ghcr.io/raymonbb/domely-tunnel:0.1.4 agent/tunnel
```

Those are run from the repository root, and without `--push` they build and go nowhere, which is
the only form of them this milestone has run. The build argument is what the container answers
with when asked which release it is (#322); leaving it out builds an image that says `dev`.

## Status

Week 5, packaging. The stack is still the adapter, the API and the tunnel, and every route but
`/health` and the adapter's own push needs a valid token.

**Nothing is published.** No image has been pushed and no version tag has been cut, so the tags in
`compose.yml` are what the first release will produce rather than something anybody can pull today.
Building locally is the whole story until then, and the section at the top says how.

**This stack serves no app.** There is nothing to open in a browser here beyond `/health`, and
nothing to build before starting it: the PWA is served by Domely Cloud, which is a deployment of
its own ([`deploy/cloud`](../cloud/README.md)), and the app talks to this Agent over the tunnel
from there. That is also why `CORS_ORIGINS` exists at all, and the order that matters is the
cloud's: its frontend is built before its stack is started.

What has never been exercised against Cloudflare is the tunnel actually coming up, because that
needs an account: everything up to the `cloudflared` invocation is covered by
`agent/tunnel/test.sh`, and [`docs/remote-access.md`](../../docs/remote-access.md) lists the three
things a single run against a real account would settle, the event stream through the hop
included.
