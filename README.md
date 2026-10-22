<img src="docs/logo.png" alt="SafeSync" width="160" align="right">

# SafeSync

Player data sync between the servers of a network (Paper or Folia, behind Velocity or BungeeCord): the inventory,
ender chest, health, XP and more follow the player from one server to the next, **never duplicated, never lost**.
MySQL or MariaDB only: **no Redis, nothing on the proxy**.

The proxy connects a player to the new server *before* it disconnects them from the old one, so the new server can
read the data before the old one has written it. That is where items get duplicated (die then disconnect at once,
server restart, quick back and forth) or whole inventories disappear (save too slow under load). Every such case
below has a defense in SafeSync and a bot test that fails when the defense is removed.

> **Status:** version 1.0.0 is finished and tested, not released yet. Free edition: SpigotMC, CurseForge and
> [Releases](../../releases/latest); full edition: Payhip (links when published).

## Editions

Duplication and loss safety is never limited in the free edition.

| | Free | Full (12.99 USD) |
|---|---|---|
| Servers | 2 synced together (one group), plus any number of standalone servers (alone in their group) | unlimited, unlimited groups |
| Data | inventory, ender chest, armor and off-hand, health, hunger, XP | + potion effects, game mode, flight, statistics, advancements, location (off by default), plugin data (off by default) |
| Duplication and loss safety, crash recovery, conflicts | yes | yes |
| First install: local data of the other servers kept | yes | yes |
| Time-stamped history and restore | no | yes |
| Import from HuskSync and MySQL Inventory Bridge | no | yes |
| Floodgate account linking | no | yes |
| Transfer log, PlaceholderAPI | no | yes |

## Never duplicated, never lost

| # | What goes wrong | Defense |
|---|---|---|
| 1 | The new server reads before the old one has written (Velocity, BungeeCord) | checkout in the database: one owner per player and group, taken by a conditional update; the new server waits (player frozen) until the old one has written and released |
| 2 | Die, then disconnect at once | the save is the real state after death; on the next server the player is alive, never a second death (no second drop if `keepInventory` differs) |
| 3 | Server restart or stop | every player is saved and released before the database closes; the stop waits for the writes (then the local journal) |
| 4 | Save too slow under load | saves run off the game thread, one queue per player; the next server waits for that version, never an older one |
| 5 | Server crash | heartbeat every 5 s; after 20 s of silence the checkout is stale and the player loads elsewhere, from the last save |
| 6 | Same player on two servers at once (unstable proxy) | a single checkout: the second server freezes, then disconnects without writing |
| 7 | Item on the cursor, in the crafting grid, open container | given back to the inventory before the save |
| 8 | Picking up, dropping, opening a chest, moving while loading | everything is cancelled until the data is applied |
| 9 | Database down | joining: frozen then disconnected, never an empty inventory written; leaving: kept in a local journal on disk, written when the database is back, checkout not released meanwhile |
| 10 | Late write from a server that lost the checkout | refused by the version condition, kept as a **conflict** |
| 11 | Item from a newer Minecraft version (mixed versions) | the older server does not load it (frozen, then disconnected with a message); newer servers read older data |

A server that cannot write its heartbeat for 10 s **freezes its own players** and keeps their state in its local
journal: the other servers will soon consider them free, and they must not keep playing on data that is about to be
taken back. When the database answers again, players whose checkout is still held are unfrozen.

**What a crash can cost: at worst the play since the last regular save (2 minutes by default, `autosave-seconds`).**
Never an item in double, never the whole inventory. Tested by killing the server (`kill -9`) at each step of a
player's departure: after the capture, after the encoding, before the database commit, after the release. The
guarantee is per player: an item handed to another player during those last minutes is back with both after a crash,
since each one gets their own last save.

A save the database refuses for good (over 8 MB, or larger than the server's `max_allowed_packet`) is not retried:
it is kept in `plugins/SafeSync/pending/` as a `.refused` file, logged once, and on departure the player's checkout
is released so they get their last save back instead of staying stuck.

### Conflicts

Any refused write, any checkout taken back from a silent server and any outdated local journal entry is kept as a
conflict (player, server, time, content): listed at startup and by `/safesync conflicts`. `/safesync conflict <id> view`
shows the difference with the current state, `apply` replaces the whole player state with it (now if the player is
online on this network, at their next join otherwise), `discard` drops it. Applying keeps the replaced state as a
conflict too: nothing is ever overwritten without a copy. Applying an old state after the player kept playing is an
admin decision.

### First install

When a player without data in the database joins, their local data on that server becomes the data of the group.
The local data they have on the **other** servers of the group is not overwritten in silence: on their first visit
there it is kept as a snapshot (`LOCAL`), restorable (full edition) or visible as a conflict (free edition).

## What follows the player

| Data | Config key | Edition |
|---|---|---|
| Inventory, armor, off-hand | `sync.inventory` | free |
| Ender chest | `sync.ender-chest` | free |
| Health, max health | `sync.health` | free |
| Hunger, saturation | `sync.hunger` | free |
| Experience | `sync.experience` | free |
| Potion effects (time left) | `sync.potion-effects` | full |
| Game mode | `sync.game-mode` | full |
| Flight | `sync.flight` | full |
| Statistics | `sync.statistics` | full |
| Advancements (no chat announcement, no second reward) | `sync.advancements` | full |
| Location (mirrored worlds only; missing world: ignored) | `sync.location`, off by default | full |
| Plugin data stored on the player (PDC) | `sync.persistent-data`, off by default | full |

Items are kept with Paper's own versioned format: components, custom names, lore, enchantments, plugin data, full
shulker boxes and bundles come back byte for byte.

A server that does not sync a data (different config, or free edition in a group of full servers) **keeps the value
written by the others** instead of wiping it, and does not apply it: going through a "smaller" server never loses
effects, statistics or advancements.

**Money is not synced:** leave it to your economy plugin (most have their own network option).

## Groups

Servers with the same `group` share the player data. A server in another group (creative, lobby) keeps its own. A
player has one set of data per group.

## History and restore (full edition)

- Each snapshot keeps its cause (quit, autosave, death, world change, stop, restore, import, link), the server and the
  time. Retention: `history.keep-last` (20) per player, plus one per day for `history.keep-days` (7). Conflicts are
  never pruned.
- `/safesync history <player> [group]`: time-stamped list in chat, chest menu on Java (click: read-only preview of the
  inventory and ender chest, shift-click: restore), form on Bedrock.
- `/safesync restore <player> <id>`: shows the snapshot, then `confirm`. Applied now on the player's server if they
  are online, at their next join otherwise; the replaced state is kept.
- Transfer log, one file per day in `plugins/SafeSync/logs/`: load, save, conflict, restore, with version, cause,
  servers and freeze time.

## Import from HuskSync or MySQL Inventory Bridge (full edition)

`/safesync import husksync|mib [--dry-run] [--overwrite]`, with the source database in the `import:` section of the
config. The source database is opened read only: SafeSync never writes there. A player who already has SafeSync data
is skipped unless `--overwrite`. Run it with no player online on the group; a report lists imported, skipped and failed
players.

- **HuskSync** (tested with 4.1): inventory, ender chest, health, hunger, XP, effects, game mode, flight, advancements.
- **MySQL Inventory Bridge** (tested with 2.4.1, table `mib-table`, `meb_inventory` by default): inventory and armor,
  the only data it syncs. Items are read by the server and upgraded to its Minecraft version. Its "modded items"
  mode (ProtocolLib) is not read.
- **MySQL Player Data Bridge** is not supported: its format cannot be checked without buying it.

## Floodgate account linking (full edition)

When a Bedrock player links their account to a Java account, Floodgate gives them the Java UUID and their Bedrock data
stays under the old one. On their first join under the Java UUID, SafeSync merges once: if one side is empty, the
other is taken as is; otherwise the items of the Bedrock account go to a claim box (`/safesync claim`: one item per
click, a chest menu on Java, a form on Bedrock, refused when the inventory is full, nothing on the ground), and the
rest (health, XP...) keeps the Java value. The Bedrock data is kept as a snapshot.

## Login plugins

- **AuthMe** (on the server): a player who has not typed their password yet is not loaded and holds no checkout;
  loading starts at their login, after AuthMe has restored their state. A player who leaves before logging in writes
  nothing. Works with the pre-join dialog of AuthMe on Paper too.
- **LibreLogin on the proxy**: nothing to do, it never touches the servers' data; Bedrock players (Floodgate) go
  straight to the lobby. LibreLogin installed on a Paper server moves players to a void world until they log in: if
  you sync `location`, use it on the proxy instead.

## Commands

| Command | Role | Permission |
|---|---|---|
| `/safesync status` | servers, heartbeats, checkouts held, local journal, conflicts | `safesync.admin` (op) |
| `/safesync conflicts` | list conflicts | `safesync.admin` |
| `/safesync conflict <id> [view\|apply\|discard]` | see, apply or drop a conflict | `safesync.admin` |
| `/safesync unlock <player> [confirm]` | release a stuck checkout (says who holds it first) | `safesync.admin` |
| `/safesync history <player> [group]` | history (full) | `safesync.admin` |
| `/safesync restore <player> <id> [confirm]` | restore a snapshot (full) | `safesync.admin` |
| `/safesync import husksync\|mib [--dry-run] [--overwrite]` | import (full) | `safesync.admin` |
| `/safesync claim` | claim box after account linking (full) | `safesync.use` (everyone) |
| `/safesync reload`, `/safesync version` | admin | `safesync.admin` |

## PlaceholderAPI (full edition)

- `%safesync_state%`: `ACTIVE`, `WAITING` (loading), `LEAVING`, `FENCED` (database lost), empty when not synced
- `%safesync_last_save%`: seconds since the last save of the player, `-` if none

## Configuration

```yaml
server-name: server1      # different on each server
group: default            # same on the servers that share the data
storage:                  # the same MySQL or MariaDB for every server
  host: localhost
  port: 3306
  database: safesync
  user: safesync
  password: ''
autosave-seconds: 120     # a crash loses at most the play since the last save
lock-timeout-seconds: 10  # arriving player waits at most this long for the previous server, then is disconnected
heartbeat-seconds: 5
heartbeat-stale-seconds: 20  # a server silent this long is considered down
frozen-commands: []       # commands allowed while the data loads
```

Plus the `sync:` section (table above), `history:` and `import:` (full edition). Every key is commented in the
generated `config.yml`.

## Setup

1. Create a database and a user on your MySQL or MariaDB server.
2. Put the jar in `plugins/` of each server of the group, start once.
3. In `plugins/SafeSync/config.yml`: the same `storage` and `group` everywhere, a different `server-name` on each
   server.
4. Restart. The log says `Syncing player data as server '...' in group '...'`.

## Requirements

- Paper or Folia 1.21.4 to 26.x, Java 21+. Tested on Paper 1.21.11, Paper 26.3 and Folia 26.3, behind Velocity and
  BungeeCord, and on a network mixing 1.21.11 and 26.3.
- MySQL or MariaDB.
- Optional: Floodgate (account linking, Bedrock admin forms), PlaceholderAPI.
- Not compatible with per-world inventory plugins (Multiverse-Inventories, PerWorldInventory): each would overwrite
  the other's inventories. SafeSync warns at startup.
- Not synced: money, unlocked maps, inventories of other plugins (backpacks...).
- Items given by a kit or join plugin while the player's data is loading are replaced by the loaded data: give them
  a moment after the join. Advancements and effects that only exist in a datapack are dropped when the player saves
  on a server without that datapack.
- No license server.

## Support

Bug reports and questions: [issues](../../issues).
