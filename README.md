# NetDuke32 Pseudo-Server

A self-hosted, on-demand Duke Nukem 3D multiplayer host running headless in Docker, using [NetDuke32](https://voidpoint.io/StrikerTheHedgefox/eduke32-csrefactor) — a fork of EDuke32 dedicated to fixing Duke3D's old master/slave multiplayer netcode.

## Why "pseudo" server?

NetDuke32's multiplayer is built on Duke3D's original 1996 lockstep networking model. There is no true dedicated-server mode in this engine — confirmed directly by the current maintainer as of March 2026. The **host is always a full participating player**, with their own character physically present in the level, not invisible infrastructure sitting off to the side.

This repo runs that host headlessly — no monitor, no human watching or controlling it — but it's worth being upfront that this is a "pseudo" server, not a real dedicated one. A few practical consequences of this are documented in [Known quirks](#known-quirks-of-a-headless-host) below.

## What this is

A way to stand up a "server" on infrastructure reachable over the public internet — your own hardware behind a WAN IP, or cloud infrastructure on AWS, Azure, etc. This can make it easier for friends to connect, since a host with a static public IP gives everyone a stable address to point their client at, rather than relying on whoever happens to be hosting that night.

It's also a low-friction way to start NetDuke32 for multiplayer without needing to memorize the engine's command-line switches. The included `docker-compose.yml` exposes friendly environment variables for setting the game mode, player count, episode, and level.

## What this is not

Not a true dedicated server — NetDuke32's lockstep networking model doesn't support that. The "server" has to be started on-demand when you want to play, and each friend joins it from their own instance of NetDuke32 around the same time. It can't run 24/7 and silently accept new connections whenever someone shows up; starting a session still has to be coordinated with whoever's playing.

## What this actually solves

Running NetDuke32 fully headless (no display, no human input) turns out to be harder than it sounds. The most significant issue: under the usual headless-X approach (`Xvfb`), the host process **freezes solid on the very first rendered frame** — alive, burning 100% CPU, producing no further output, forever.

Root cause (confirmed via GDB): NetDuke32 calibrates its per-frame timing based on the display's reported refresh rate. Xvfb has no real display and reports an effective refresh rate of zero, which collapses that calibration to a literal `0` — producing an infinite `while (x < target) x += 0;` busy-wait.

**The fix:** run a real X server (`Xvnc`) instead of Xvfb. Xvnc is a genuine X server implementation and reports a real, configured refresh rate — nobody ever needs to actually connect a VNC viewer to it; it just needs to exist. This is baked into the image's `start.sh` and requires no action on your part.

## Requirements

- Docker + Docker Compose
- Your own legally-owned copy of Duke Nukem 3D (Atomic Edition): `DUKE3D.GRP` and `DUKE.RTS`. These are **not included** in this repo and cannot be — you'll need to copy them from your own GOG/Steam/retail install.
- A Linux or Windows NetDuke32 client for anyone connecting. See [Clients](#clients) below.

## Setup

```bash
git clone <this-repo>
cd netduke32-docker-server
mkdir -p gamedata
cp /path/to/your/DUKE3D.GRP /path/to/your/DUKE.RTS gamedata/
docker compose build
```

## Starting a session

```bash
docker compose up
```

This launches with sensible defaults: 2 players, Coop, Episode 1 Level 1 (Hollywood Holocaust). Wait for `Waiting for players...` in the logs before anyone connects.

`Ctrl-C` to stop. The container is removed automatically — it's deliberately **not** configured to auto-restart, since there's no benefit to leaving this running when nobody's playing (see [Known quirks](#known-quirks-of-a-headless-host) — there's no late-joining, so an always-on host doesn't actually buy you anything).

### Configuring a session

Settings are passed as environment variables at launch time — no need to edit `docker-compose.yml` directly:

| Variable | Default | Meaning |
|---|---|---|
| `PLAYERS` | `2` | Total players, **including the host** (see [Known quirks](#known-quirks-of-a-headless-host)) |
| `GAMEMODE` | `2` | `1` = Dukematch, `2` = Coop, `3` = Team Dukematch |
| `EPISODE` | `1` | `1`–`4` (`4` = The Birth, Atomic Edition only) |
| `LEVEL` | `1` | Level number within the episode |
| `HOSTNAME_INGAME` | `Server` | The host's displayed player name |

#### Examples

**4-player Coop, Episode 2 Level 3:**
```bash
PLAYERS=4 EPISODE=2 LEVEL=3 docker compose up
```

**Dukematch (deathmatch), 3 players, default level:**
```bash
PLAYERS=3 GAMEMODE=1 docker compose up
```

**Team Dukematch, 6 players, The Birth (episode 4) Level 1:**
```bash
PLAYERS=6 GAMEMODE=3 EPISODE=4 LEVEL=1 docker compose up
```

**Persisting your usual settings** — drop a `.env` file next to `docker-compose.yml` instead of typing variables every time:
```bash
# .env
PLAYERS=4
GAMEMODE=2
EPISODE=1
LEVEL=1
```
Then just `docker compose up` picks these up automatically.

To check exactly what command will be launched before committing to it:
```bash
PLAYERS=4 docker compose config
```

## Networking

The container publishes UDP `23513` via standard Docker bridge networking — no special network mode required.

**Firewall on the host machine:** no rule needed. Docker's own iptables chains handle the published port and bypass the host's firewall INPUT chain entirely (a known, accepted behavior of Docker — not specific to this project).

**Firewall on each client machine:** if a firewall is active (default on some distros, e.g. CachyOS ships ufw enabled by default), you'll need an explicit allow rule, or the client will silently never transmit a single packet:
```bash
sudo ufw allow from <host-ip> to any port 23513 proto udp
```

## Clients

Anyone connecting needs a NetDuke32 client and their own copy of `DUKE3D.GRP` / `DUKE.RTS`.

- **Linux:** build from source (see the `Dockerfile` in this repo for the exact build steps/patches), or via the AUR `netduke32` package.
- **Windows:** the [NukemNet Complete Fun Pack](https://nukemnet.com/downloads) bundles a working pre-built binary plus mods/usermaps — generally the easiest path for less technical friends.

### Connecting

**The join syntax matters and is easy to get wrong.** It is *not* `-net ip:port`:

```
netduke32 -nosetup -nologo -net <host-ip> -n0:<PLAYERS> -p23513
```

- `-n0:<PLAYERS>` must match the `PLAYERS` value the host was started with.
- `-p23513` must match the host's port.
- Episode/level/game mode are inherited from the host — don't pass `-v`/`-l`/`-c` on the client.

## Known quirks of a headless host

Because the host is a real, participating player rather than invisible infrastructure, a few things behave differently than you might expect from a typical game server. The "server" is the host/master for the lockstep networking — each real player connecting is a client/slave.

- **The host's character will consume a player slot.** Given everything that has already been stated, this might be obvious. The end result is that you have to +1 to the player count. E.g. if you want to play with 4 of your friends, you'll need to start the server with support for 5 total players.
- **The host's character does not respawn after dying.** In Dukematch, this means it's technically possible for someone to score a single kill against the host's character early on — but after that, the host stays dead and can't be fragged again for the rest of the match.
- **In Coop, level transitions don't strictly wait on the host.** The game does wait for player input before progressing to the next level, but if the host (whose character is unattended) doesn't provide any, the level eventually advances anyway after a timeout, without requiring host input.

Neither of the latter two is a bug in this setup specifically — they're consequences of running a real player slot with nobody actually playing it, which is the fundamental nature of NetDuke32's master/slave architecture (see [Why "pseudo" server?](#why-pseudo-server) above).

## Acknowledgments

- [EDuke32](https://voidpoint.io/EDuke32/eduke32) / [NetDuke32](https://voidpoint.io/StrikerTheHedgefox/eduke32-csrefactor) by StrikerTheHedgefox and contributors
- Build pinned to NY00123's maintained fork, tag `netduke32-r11589-post_v1.2.1-345003a8d` — also the build bundled in [NukemNet](https://nukemnet.com)'s Complete Fun Pack