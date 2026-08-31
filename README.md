# Server Saver Bot v2

Slash-command moderation bot with versioned backup/restore and anti-nuke protection.

## ⚠️ First step: rotate your token

If your old bot token was ever hardcoded in a shared file, rotate it in the
[Discord Developer Portal](https://discord.com/developers/applications) →
your app → **Bot** → **Reset Token** before running this.

## Setup

1. Install dependencies:
   ```
   pip install -r requirements.txt
   ```
2. Copy `.env.example` to `.env` and fill in your token:
   ```
   cp .env.example .env
   ```
3. Load it before running — either via your host's dashboard, or add this
   to the very top of `main.py`:
   ```python
   from dotenv import load_dotenv
   load_dotenv()
   ```
4. In the Developer Portal, under **Bot**, enable **Server Members Intent**
   and **Message Content Intent**.
5. When inviting the bot, grant the `applications.commands` scope in
   addition to `bot` so slash commands register.
6. Run it:
   ```
   python main.py
   ```
   Slash commands sync globally on startup — the very first sync (or any
   time you add/change a command) can take up to an hour to appear for
   users, since Discord caches global command lists client-side. This is
   normal and not something the bot controls; it settles down once it's
   been running a while.

## Project layout

- `main.py` — bot entrypoint, event listeners (welcome message, word
  filter, anti-nuke triggers)
- `storage.py` — SQLite layer (`serversaver.db`, created automatically):
  guild config, admin whitelist, bot whitelist, versioned backups (last 5
  kept per server), anti-nuke trigger history
- `backup.py` — snapshot/restore: categories, channels, permission
  overwrites, roles, server name, icon, banner, `@everyone` permissions
- `anti_nuke.py` — detection logic and response (strip dangerous roles,
  log, DM owner)
- `cogs/moderation.py` — `/kick /ban /unban /warn /bans`
- `cogs/backup_cog.py` — `/backup /restore /backups /exportbackup /importbackup`
- `cogs/admin_cog.py` — `/setlogchannel /setwelcomechannel /antinuke
  /whitelist /botwhitelist`
- `cogs/dashboard.py` — `/status`
- `cogs/help_cog.py` — `/help`

## Commands

**Backup & restore**
- `/backup` — snapshot the server (admin only)
- `/restore` — shows what a restore will do and asks for confirmation via
  buttons before doing anything (admin only)
- `/backups` — list saved backups with IDs and timestamps (admin only)
- `/exportbackup [backup_id]` — download a backup as a `.json` file to keep
  yourself, back up externally, or move to another server (admin only)
- `/importbackup file:` — attach a previously exported `.json` file to load
  it in as a new backup for this server. Works across servers too — export
  from one, import into another, then `/restore` there (admin only)

**Config**
- `/setlogchannel #channel` — where mod actions and anti-nuke triggers get
  logged
- `/setwelcomechannel #channel` — where new-member welcomes are posted
- `/antinuke sensitivity` — choose `strict` / `balanced` / `lenient` (see
  table below); defaults to `strict`
- `/antinuke status` — show the current sensitivity level and its exact
  thresholds
- `/whitelist add|remove|list` — exempt trusted admins from anti-nuke
  checks (the server owner is always exempt automatically)
- `/botwhitelist add|remove|list` — allow specific bots to join without
  being auto-kicked

**Moderation**
- `/kick /ban /unban /warn /bans`

**Dashboard**
- `/status` — uptime, member count, latest backup, log/welcome channel
  config, whitelist sizes, current sensitivity, and the last 5 anti-nuke
  triggers

**Help**
- `/help` — lists every command above, grouped by category (ephemeral,
  only visible to the person who ran it)

## Anti-nuke coverage

All of these strip the actor's dangerous roles (administrator, manage
channels/guild/roles, ban/kick members, manage webhooks), kick them from
the server, post to the log channel, and DM every member with
Administrator permission. The server owner and anyone on `/whitelist` are
always exempt.

Thresholds depend on the server's sensitivity level (`/antinuke
sensitivity`), since how fast this should react is a real tradeoff:
faster reaction limits damage from an actual attack, but reacts to a
single isolated deletion too — so anyone doing routine admin work should
be on `/whitelist` regardless of which level you pick.

| Trigger | strict | balanced | lenient |
|---|---|---|---|
| Channel deletions | 1 (immediate) | 2 within 5s | 3 within 10s |
| Bans | 1 (immediate) | 2 within 5s | 3 within 10s |
| Kicks | 1 (immediate) | 2 within 5s | 3 within 10s |
| Role deletions | 1 (immediate) | 2 within 5s | 3 within 10s |
| Role creations | 3 within 10s | 4 within 10s | 5 within 15s |

These are always immediate, regardless of sensitivity:
| Trigger | Reaction |
|---|---|
| Any role granted a dangerous permission it didn't have | reverts the permission change, then locks down the actor |
| `@everyone` granted a dangerous permission | reverts the permission change, then locks down the actor |
| Unwhitelisted bot joins | the bot is kicked on join |

**Note on permission-escalation detection:** this catches a role or
`@everyone` *gaining* a dangerous permission it didn't have before. It
reverts the change immediately, then strips the actor's own dangerous
roles — so even if they escalated a role, they lose the access needed to
use it.

## Restore behavior

`/restore` now also restores server name, icon, banner, and `@everyone`
permissions, in addition to categories/channels/roles/overwrites — asked
via a button confirmation, not a typed command, so there's no risk of a
stray message triggering it.

**Still not restorable:** message history, and member-specific (as
opposed to role-based) permission overwrites, since member IDs and old
messages aren't things the bot can reliably recreate.

## Notes on the anti-nuke design

This is burst/heuristic detection, not a guarantee. It's a strong
deterrent against a compromised account or malicious bot going on a fast
rampage, but it can't stop a determined attacker with `administrator`
who moves slowly enough to stay under the thresholds. Pair this with:
- Requiring 2FA for moderation in Discord's own server settings
- Keeping the number of members with `administrator` small
- Reviewing `/whitelist` periodically
