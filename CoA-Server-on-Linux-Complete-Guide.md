# Conquest of Azeroth (CoA) Private Server on Linux — The Complete Newbie Guide

**From zero to a populated Azeroth: Distrobox + Wine + CoA Repack + CoA Bots v1.3**

*Battle-tested on Bazzite 44 (Fedora Atomic), September 2026. Works on any Linux distro with Distrobox.*

---

## Table of Contents

0. [What you're building (mental model)](#0-what-youre-building)
1. [Requirements & Downloads](#1-requirements--downloads)
2. [Directory layout](#2-directory-layout)
3. [The container](#3-the-container-distrobox--wine)
4. [Know your repack (read before running)](#4-know-your-repack)
5. [First launch](#5-first-launch)
6. [Troubleshooting MySQL (the two classic failures)](#6-troubleshooting-mysql)
7. [Playing without bots](#7-playing-without-bots)
8. [Installing the bots](#8-installing-co-bots-v13)
9. [First bots boot (the long one)](#9-first-bots-boot)
10. [Troubleshooting: stuck on "connecting"](#10-troubleshooting-stuck-on-connecting)
11. [Living with bots](#11-living-with-bots)
12. [Daily operations cheat sheet](#12-daily-operations-cheat-sheet)
13. [Appendix: error → fix quick table](#13-appendix-error--fix-quick-table)

---

## 0. What you're building

You will run a World of Warcraft **private server** (Conquest of Azeroth, an AzerothCore 3.3.5a mod) on your own PC, then connect the game client to `127.0.0.1` (yourself). AI playerbots populate the world.

Four moving parts:

| Component | What it is | In this repack |
|---|---|---|
| **worldserver.exe** | The game world (NPCs, quests, spells) | `Server/Core/worldserver.exe` + a newer one in `CoA-Bots/` once bots are installed |
| **authserver.exe** | Login server (accounts, realm list) | `Server/Core/authserver.exe` |
| **MySQL 8.4** | Database (already populated — no importing!) | `Server/mysql/` |
| **manage.py / coa_bots.py** | Python supervisor that starts/stops everything in order | `Server/Scripts/`, `Server/CoA-Bots/` |

**Key facts about this specific repack (main-20260919-b3717c137):**
- Fully **64-bit** → no 32-bit/Wine-wow64 problems
- World database **already populated** → the scary 20-minute first import of older guides does **not** exist here
- A GM account **`local` / `local`** (level 3) already exists
- Bundles its **own Windows Python** (`Runtime/python/python.exe`) — you never install Python or MySQL yourself
- Every `.bat` file is a thin wrapper around a Python supervisor — we call the Python directly

**Ports:** MySQL `3307` · auth `3724` · world `8085` · GM console `3443`

**Why Distrobox?** The repack is Windows software; **Wine** runs it. On immutable distros (Bazzite/SteamOS) you can't install system packages, so Distrobox provides a normal Fedora container for Wine. On any other distro, Distrobox still gives you a clean, disposable environment — and the host stays untouched.

---

## 1. Requirements & Downloads

### Hardware
- **RAM:** 16 GB minimum (bots default 200); **32 GB comfortable** (1000+ bots)
- **Disk:** ~60 GB free (repack ~25 GB + client ~20 GB + bots + backups)
- CPU: anything modern; first bots boot is CPU-heavy for ~30–50 min

### Downloads (from the CoA Discord release channels)

| What | Link | Notes |
|---|---|---|
| **Server repack (with update included)** | `https://drive.proton.me/urls/TYZZH1H62W#BC2SXFZCYQBc` | Pick "update included" — skips a whole step |
| Base client | `https://drive.proton.me/urls/TSVCQXCB2W#zm2YuJ2qpDwj` |  |
| Client patch (**mandatory**) | `https://drive.proton.me/urls/ZHFXKRGEN4#F54Z6NvX7Nmq` | "js-3067-client" — apply ON TOP of the base client |
| CoA Bots v1.3 **full package** | `https://drive.proton.me/urls/PC8C71BMN4#IBtnVVERXr17` | Only the full package for new installs |

> The bots version must match the repack release (`main-20260919-b3717c137`). The zip name says it. Never mix versions.

**Verify the bots zip after downloading** (catches corrupted downloads):
```bash
sha256sum CoA-Bots-v1.3-main-20260919-b3717c137.zip
# MUST equal: 299D38A8DCF8B0ED51247648DF8F57E5905BE27C4FD402811C3A2B694E27D1C4
```

### ⚠️ Before extracting
**Never run the server from NTFS or exFAT** (external drives, Windows partitions). MySQL data corrupts. Use your Linux home directory (ext4/btrfs). Copy archives onto it first.

---

## 2. Directory layout

```bash
mkdir -p ~/Games/coa-server/Server ~/Games/coa-server/client
```

Extract:
- **Repack** → into `~/Games/coa-server/Server/`
- **Base client** → into `~/Games/coa-server/client/`
- **Client patch** → **on top of** the same `client/` folder (overwrite when asked)

Final structure (names/casing matter):

```
~/Games/coa-server/
├── Server/                  ← THE "repack folder" (every bots guide means this)
│   ├── Core/                ← authserver.exe, worldserver.exe, Logs/
│   ├── mysql/               ← Windows MySQL 8.4 (bin, data, logs, my.ini)
│   ├── Runtime/python/      ← bundled Windows Python
│   ├── Scripts/manage.py    ← the supervisor
│   ├── Settings/            ← .conf templates, database.json, repack.json
│   ├── Data/                ← game data (DBC, maps)
│   ├── Start_All_Server.bat, Stop_All_Server.bat, ...
│   └── (later) CoA-Bots/    ← bots live here
├── client/                  ← Ascension.exe, Data/, Interface/, realmlist.wtf
└── wine-prefix/             ← created automatically later
```

Sanity check:
```bash
ls ~/Games/coa-server/Server/      # you should see Core, mysql, Settings, Scripts...
ls ~/Games/coa-server/client/      # Ascension.exe, Data, Interface...
```

---

## 3. The container (Distrobox + Wine)

On the host terminal (works on Bazzite and any distro with distrobox installed):

```bash
distrobox create --name coa-server --image registry.fedoraproject.org/fedora-toolbox:44
distrobox enter coa-server
```

Your prompt changes to something like `📦[user@coa-server ~]$` — you're inside the box. The home folder is **shared** with the host, so `~/Games/coa-server` is the same place everywhere.

Install the few things we need (deliberately short list — the repack brings its own Python/MySQL):

```bash
sudo dnf install -y wine nc
```

**Wine prefix** (a fake Windows just for the server — never mix with your Lutris game prefixes):

```bash
export WINEPREFIX=$HOME/Games/coa-server/wine-prefix
export WINEARCH=win64
wineboot --init
winecfg          # set "Windows Version" = Windows 10, OK
```

> **Why fedora-toolbox:44 is safe here:** older guides warn about 32-bit worldserver crashing on Fedora 44. This repack is 100% 64-bit (`libcrypto-3-x64.dll`, README says 64-bit Windows) — the problem doesn't apply.

---

## 4. Know your repack

**Rule #1 of this whole guide: read a launcher before running it.** Every failure we hit was diagnosed this way.

```bash
cat ~/Games/coa-server/Server/Start_All_Server.bat
cat ~/Games/coa-server/Server/Scripts/manage.py | head -60
```

You'll find: every `.bat` calls `Runtime\python\python.exe -B Scripts\manage.py <action>`. Useful actions: `start-all`, `start-mysql`, `start-auth`, `start-world`, `stop-all`, `status`.

The supervisor: regenerates configs for your actual path on every start, starts MySQL → auth → world in order, checks ports, sets the realmlist to `127.0.0.1:8085` automatically, and shuts everything down gracefully. **Never run the .exe files directly.**

---

## 5. First launch

Inside the container:

```bash
export WINEPREFIX=$HOME/Games/coa-server/wine-prefix
export WINEESYNC=1 WINEFSYNC=1 WINEDEBUG=-all
cd ~/Games/coa-server/Server
wine Runtime/python/python.exe -B Scripts/manage.py start-all
```

Success looks like:

```
Starting MySQL...
Starting authserver...
Starting worldserver and its bug-report relay...
Worldserver and automatic bug reporting are ready.
Use Stop_All_Server.bat to shut down this repack safely.
```

Verify (from any terminal — the container shares the host network):

```bash
nc -z 127.0.0.1 3307 && echo DB-OK
nc -z 127.0.0.1 3724 && echo Auth-OK
nc -z 127.0.0.1 8085 && echo World-OK
```

**If MySQL failed here → go to [Section 6](#6-troubleshooting-mysql).** (It failed for us. It will probably fail for you. It's fine.)**

---

## 6. Troubleshooting MySQL

### Failure 1 — "Can't create UNDO tablespace innodb_undo_001 since '.\undo_001' already exists"

**Symptom:** `ERROR: mysql stopped during startup. Check its log.` — and in `mysql/logs/mysql-error.log`:
```
[ERROR] [MY-013039] [InnoDB] Can't create UNDO tablespace innodb_undo_001 since '.\undo_001' already exists.
[ERROR] [MY-010334] [Server] Failed to initialize DD Storage Engine
```

**Cause:** The database folder was created on Windows. After moving it to a Linux filesystem, InnoDB's internal catalog loses track of its "undo tablespaces" (disposable transaction scratch files), tries to create fresh ones, and finds the old files in the way.

**Fix** (undo files are scratch space — your actual game data is in other files and is NOT touched):

```bash
wineserver -k
nc -z 127.0.0.1 3307 && echo "STILL RUNNING" || echo "clean"   # want: clean

cd ~/Games/coa-server/Server/mysql/data
mkdir -p ~/Games/coa-server/old-undo-backup/      # OUTSIDE the data dir — see Failure 2!
mv undo_001 undo_002 undo_*_trunc.log ~/Games/coa-server/old-undo-backup/ 2>/dev/null
ls | grep -i undo    # should print nothing
```

Then relaunch `start-all`. MySQL recreates clean undo files automatically.

### Failure 2 — "Multiple files found for the same tablespace ID"  ⚠️ caused by fixing Failure 1 wrong

**Symptom:**
```
[ERROR] [MY-012209] [InnoDB] Multiple files found for the same tablespace ID:
[ERROR] Tablespace ID: 4294967278 = ['undo-backup\undo_002', 'undo_002']
```

**Cause:** You kept the old undo files in a backup folder **inside** `mysql/data/`. InnoDB scans subdirectories of its data directory — it found two files claiming the same internal ID.

**Fix:** move backups **completely outside** `mysql/data/`, e.g. to `~/Games/coa-server/old-undo-backup/` (as shown above). Never keep stray InnoDB files anywhere under `data/`.

### Failure 3 — "'Windows aio' returned OS error 0. Cannot continue operation"  ⭐ the big one

**Symptom:** MySQL starts fine ("ready for connections"), then **7 seconds later** — right when the worldserver begins querying the big game-data table:
```
[ERROR] [MY-012646] [InnoDB] File .\acore_world\item_template.ibd: 'Windows aio' returned OS error 0. Cannot continue operation
[ERROR] [MY-012981] [InnoDB] Cannot continue operation.
```
The worldserver then dies with "Lost connection to MySQL server during query".

**Cause:** MySQL on Windows uses asynchronous file I/O ("Windows AIO", overlapped I/O). Wine implements this unreliably — the operation returns garbage and InnoDB aborts. **Not** a RAM problem (check `free -h` — you'll see plenty free).

**Fix:** force MySQL's synchronous I/O path with `innodb_use_native_aio=0`.

*Step 1 — confirm the flag is accepted on this build (it is, but verify in 30 s):*
```bash
cd ~/Games/coa-server/Server/mysql
wine bin/mysqld.exe --defaults-file="$HOME/Games/coa-server/Server/mysql/my.ini" --innodb-use-native-aio=0 --no-monitor
```
> **Silence ≠ failure!** `my.ini` contains `log-error=...mysql-error.log`, so ALL output goes to the log file, not your terminal. Watch it from a second terminal:
> ```bash
> tail -20 ~/Games/coa-server/Server/mysql/logs/mysql-error.log
> ```
> `ready for connections` = flag works. Ctrl+C the mysqld, wait for `Shutdown complete`.
> If instead you get `unknown option '--innodb-use-native-aio'` → this build doesn't support it; ask for help (Plan B: native Linux MySQL on the same data dir).

*Step 2 — make it permanent.* The supervisor **rewrites `my.ini` on every start**, so patch the script that generates it (one line; backup first):

```bash
cp ~/Games/coa-server/Server/Scripts/manage.py ~/Games/coa-server/Server/Scripts/manage.py.bak

sed -i "s|innodb_buffer_pool_size=256M\\\\nskip-log-bin|innodb_buffer_pool_size=256M\\\\nskip-log-bin\\\\ninnodb_use_native_aio=0|" ~/Games/coa-server/Server/Scripts/manage.py

grep -n "innodb_use_native_aio" ~/Games/coa-server/Server/Scripts/manage.py   # should show the new line
```

*Step 3 — relaunch:*
```bash
cd ~/Games/coa-server/Server
wine Runtime/python/python.exe -B Scripts/manage.py start-all
grep "innodb_use_native_aio" mysql/my.ini    # confirm the config took effect
```

> First boot after a crash does InnoDB crash-recovery — a few extra log lines and a slower start. Normal.

### Other MySQL notes
- After **any** unclean shutdown, the undo ghost (Failure 1) may return — same 30-second fix.
- Always stop with `stop-all` (Section 12), never by closing terminals.

---

## 7. Playing without bots

### Client setup (Lutris — on the Bazzite host, NOT in the container)

1. Lutris → **+** → Add locally installed game
2. Runner: **Wine** (latest GE-Proton / Wine-GE), enable **DXVK + Esync + Fsync**, Windows 10 mode
3. Executable: `/home/<user>/Games/coa-server/client/Ascension.exe`
4. Working directory: `/home/<user>/Games/coa-server/client`
5. Let Lutris create its **own** prefix — never point it at `wine-prefix`

### Check the realmlist (before first login)
```bash
cat ~/Games/coa-server/client/Data/enUS/realmlist.wtf
# should contain: set realmlist "127.0.0.1"
```

### Log in
- Account **`local`** / password **`local`** → GM level 3
- In-game: `.gm on` to verify

### GM console without a game client (handy)
The RA console is already enabled (login `local`/`local`):
```bash
nc 127.0.0.1 3443
# then type: account create myuser mypassword
# GM an account:  account set gmlevel myuser 3 -1     (AzerothCore: -1 = all realms)
```

### If the realm shows "offline"
The server's `acore_auth.realmlist` table is authoritative. Fix with the server running:
```bash
cd ~/Games/coa-server/Server
mysql --defaults-file=mysql/admin-client.ini acore_auth -e \
  "UPDATE realmlist SET address='127.0.0.1', localAddress='127.0.0.1', port=8085 WHERE id=1;"
```
(Normally `manage.py` does this automatically at every MySQL start.)

---

## 8. Installing CoA Bots v1.3

### Step 0 — Backup your world (you have characters now)
```bash
cd ~/Games/coa-server
tar -czf ~/coa-server-backup-$(date +%F).tar.gz Server/
```

### Step 1 — Stop everything
```bash
distrobox enter coa-server
cd ~/Games/coa-server/Server
wine Runtime/python/python.exe -B Scripts/manage.py stop-all
# wait for "All services from this repack are stopped."
ps aux | grep -iE "worldserver|authserver|mysqld" | grep -v grep   # must be empty
```

### Step 2 — Verify + extract into the repack folder
```bash
sha256sum "<wherever-you-keep-it>/CoA-Bots-v1.3-main-20260919-b3717c137.zip"
# MUST equal 299D38A8DCF8B0ED51247648DF8F57E5905BE27C4FD402811C3A2B694E27D1C4

cd ~/Games/coa-server/Server
unzip "<wherever-you-keep-it>/CoA-Bots-v1.3-main-20260919-b3717c137.zip"
ls -d CoA-Bots && ls ~/Games/coa-server/Server/ | grep -E "Start_All_Server|CoA-Bots"
# CoA-Bots must sit NEXT TO Start_All_Server.bat
```

### Step 3 — Install the client addon (the only client change)
```bash
mkdir -p ~/Games/coa-server/client/Interface/AddOns
cp -r ~/Games/coa-server/Server/CoA-Bots/Client/Interface/AddOns/UnBot \
      ~/Games/coa-server/client/Interface/AddOns/
```
In game: `/unbot` opens the panel, `/unbot lang en` sets English.

### Step 4 — ⚠️ The trap: Installer-Bots.bat uses PowerShell; Wine's PowerShell is a fake

`CoA-Bots/Installer-Bots.bat` calls `powershell.exe -File Installer-Bots.ps1` — and **Wine ships a PowerShell stub that does absolutely nothing** (you'll see `fixme:powershell:wmain stub` and an instant "Press any key"). Running it appears to work and installs nothing.

**Rule #2: never run a CoA-Bots .bat under Wine without reading it first.** `cat` any `.bat`/`.ps1` before running. The fix: translate the installer to bash.

**Install the native MySQL client** (for fast SQL imports):
```bash
sudo dnf install -y mariadb
```

**Create the translated installer** — paste this whole block in the container:

```bash
cat > ~/Games/coa-server/Server/install-bots.sh <<'SCRIPT'
#!/bin/bash
# Hand-translated from CoA-Bots/Installer-Bots.ps1 (Wine has no working powershell)
set -e
cd "$(dirname "$0")"
BOTS=CoA-Bots
PY="wine Runtime/python/python.exe"

echo "==> Checking nothing is running"
if nc -z 127.0.0.1 8085 2>/dev/null; then echo "ERROR: worldserver is running. Run stop-all first."; exit 1; fi

echo "==> Release check"
$PY -B $BOTS/coa_bots.py check

echo "==> Bot settings"
CONF=$BOTS/Core/configs/modules/playerbots.conf
if [ -f "$CONF" ]; then
  echo "    keeping existing playerbots.conf"
else
  PORT=$(grep -o '"mysqlPort": *[0-9]*' Settings/repack.json | grep -o '[0-9]*')
  PASS=$(grep -o '"appPassword": *"[^"]*"' Settings/database.json | cut -d'"' -f4)
  sed -e "s/<MYSQL_PORT>/$PORT/g" -e "s/<APP_PASSWORD>/$PASS/g" "$CONF.dist" > "$CONF"
  echo "    created playerbots.conf (port $PORT)"
fi
sed -i -E 's/^[[:space:]]*Playerbots\.Updates\.EnableDatabases[[:space:]]*=[[:space:]]*1[[:space:]]*$/Playerbots.Updates.EnableDatabases = 0/' "$CONF"

echo "==> Starting MySQL"
$PY -B Scripts/manage.py start-mysql

MY() { mysql --defaults-file=mysql/admin-client.ini --default-character-set=utf8mb4 -N -B -e "$1" $2; }

echo "==> Creating bot database"
MY 'CREATE DATABASE IF NOT EXISTS `acore_playerbots` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci'
for host in $(MY "SELECT host FROM mysql.user WHERE user = 'acore'"); do
  MY "GRANT ALL PRIVILEGES ON \`acore_playerbots\`.* TO 'acore'@'$host'"
done
MY 'CREATE TABLE IF NOT EXISTS `coa_bots_installed` (`file` VARCHAR(255) NOT NULL PRIMARY KEY, `created_table` VARCHAR(128) NULL, `db` VARCHAR(64) NULL, `applied` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP)' acore_playerbots

import_folder() {  # $1 folder, $2 db
  local dir="$BOTS/Database/$1"
  [ -d "$dir" ] || return 0
  echo "==> Importing $1 into $2"
  for f in "$dir"/*.sql; do
    [ -e "$f" ] || continue
    local key="$1/$(basename "$f")"
    local done_n
    done_n=$(MY "SELECT COUNT(*) FROM coa_bots_installed WHERE file = '$key'" acore_playerbots)
    if [ "$done_n" -ge 1 ]; then echo "    skip $key (already done)"; continue; fi
    local table=""
    if [[ "$1" == */base ]]; then
      table=$(head -80 "$f" | grep -i 'CREATE TABLE' | head -1 | grep -oE '`[A-Za-z0-9_]+`' | head -1 | tr -d '`')
      if [ -n "$table" ]; then
        local exists
        exists=$(MY "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = '$2' AND table_name = '$table'")
        if [ "$exists" -ge 1 ]; then
          MY "INSERT IGNORE INTO coa_bots_installed (file, created_table, db) VALUES ('$key', '$table', '$2')" acore_playerbots
          echo "    kept existing table $table"
          continue
        fi
      fi
    fi
    echo "    importing $(basename "$f") ..."
    mysql --defaults-file=mysql/admin-client.ini --default-character-set=utf8mb4 "$2" < "$f"
    if [ -n "$table" ]; then
      MY "INSERT IGNORE INTO coa_bots_installed (file, created_table, db) VALUES ('$key', '$table', '$2')" acore_playerbots
    else
      MY "INSERT IGNORE INTO coa_bots_installed (file) VALUES ('$key')" acore_playerbots
    fi
    echo "    OK $(basename "$f")"
  done
}

import_folder "playerbots/base"    acore_playerbots
import_folder "playerbots/updates" acore_playerbots
import_folder "playerbots/custom"  acore_playerbots
import_folder "world/updates"      acore_world
import_folder "characters/base"    acore_characters
import_folder "world/base"         acore_world

echo ""
echo "CoA Bots installed. No repack file was changed."
SCRIPT
```

Run it (idempotent — safe to re-run; ~2–10 min, the ~70 MB travel-nodes file is the long one):

```bash
cd ~/Games/coa-server/Server
bash install-bots.sh
```

Ends with `CoA Bots installed. No repack file was changed.`

> **If it fails at "Starting MySQL"** with a tablespace error → see Section 6 (usually: old undo backup folder still inside `mysql/data/` — move it out).

---

## 9. First bots boot

```bash
cd ~/Games/coa-server/Server
wine Runtime/python/python.exe -B CoA-Bots/coa_bots.py start-all
```

**Know what's normal:**
- `MySQL is already running.` → fine (the installer left it up).
- `ERROR: world is still starting; check its log...` → the **launcher's patience ran out**, not the server's. The worldserver keeps working in the background. Ignore and monitor.
- `fixme:` spam → harmless Wine chatter.

**The first bots boot takes 30–50 MINUTES.** The server is creating ~155 bot accounts and ~3,200 characters in the database. It looks frozen. It is not.

### Watch progress (the best meter)
```bash
cd ~/Games/coa-server/Server
mysql --defaults-file=mysql/admin-client.ini -N -e \
  "SELECT (SELECT COUNT(*) FROM acore_auth.account) AS accounts,
          (SELECT COUNT(*) FROM acore_characters.characters) AS chars"
```
Run every couple of minutes: `accounts` climbs to ~155, `chars` to ~3,200+, then:

```bash
nc -z 127.0.0.1 8085 && echo World-OK
```

### Where the logs are (surprise: NOT in CoA-Bots/)
The bots worldserver reuses the **repack's** log folder:
```bash
tail -f ~/Games/coa-server/Server/Core/Logs/world-console.log   # main console
tail -f ~/Games/coa-server/Server/Core/Logs/CoaBots.log         # bots module
tail -f ~/Games/coa-server/Server/Core/Logs/Chat.log            # bot conversations :)
```
Harmless noise you will see: `spelldifficulty_dbc` warnings (custom CoA spells), `SpellScript::_Validate ... will not be added` (a few custom spell hooks skipped), `UTF-8 to WStr` conversion notes.

### Optional housekeeping (once, while the server is stopped)
Silence a config warning seen at every login:
```bash
echo "AscensionCompat.RulesetLoginDefault = 1" >> ~/Games/coa-server/Server/CoA-Bots/Core/configs/worldserver.conf
```

---

## 10. Troubleshooting: stuck on "connecting"

**Symptom:** client shows a Lua error like ``[string "?"]:1: '=' expected near 'Glue Screen: true'`` and hangs on "connecting". Realm list works.

**Cause:** the **bot login storm**. Right after world start, thousands of bot characters synchronize into the world one by one. The worldserver's session handling is saturated and your client's handshake times out. (Auth server is unaffected — you'll see `User 'LOCAL' successfully authenticated` in `Core/Logs/Auth.log` and nothing at all about you in the world log.)

**Diagnosis:**
```bash
tail -30 ~/Games/coa-server/Server/Core/Logs/Auth.log        # auth side (works)
tail -30 ~/Games/coa-server/Server/Core/Logs/world-console.log  # storm in progress?
ss -tan | grep 8085                                          # live socket during attempt
```

**Fix: patience.** Wait until the world log has been **completely quiet for a minute** (no new "Synchronized..."/"Sent SMSG..." lines), then log in. You'll connect instantly.

**This only happens in the minutes after a server start.** Bots then stay logged in for good; all later logins (and everything else) are instant.

**If it still fails on a calm server:** the bots core is ~69 commits ahead of the repack — report it on the bots Discord with the A/B evidence (vanilla worldserver logs in instantly, bots worldserver never sees the session). That would be a client/core mismatch, not your fault.

---

## 11. Living with bots

### Recruiting
- `/who` — browse the living world (several hundred bots is normal; brackets with a real player draw *more* bots by design)
- **`.playerbots coa tank` / `heal` / `dps`** — a bot rebuilt at your level joins instantly... **⚠️ quirk:** it may pick the wrong faction (a Draenei in the Blood Elf zone gets executed by guards — yes, really). Prefer:
- **`.playerbots bot add <BotName>`** with a name from `/who` of your faction, or just **invite any bot you meet** — they answer instantly.
- Dismiss: `.playerbots bot remove <BotName>`

### Specs (e.g. make your Cultist a Heretic healer)
Whisper the bot (it must be in your group):
```
/w <BotName> talents spec list       ← its specs
/w <BotName> talents spec heal       ← switch + auto-spend talents (or: talents spec Heretic)
/w <BotName> co ?                    ← confirm combat strategies
```

### Strategies (fine-tuning behavior)
```
/w <BotName> co ?            ← list combat strategies
/w <BotName> nc ?            ← list non-combat strategies
/w <BotName> co +assist      ← enable
/w <BotName> nc +stay        ← hold position (nc -follow first)
/w <BotName> nc -stay / nc +follow   ← release
/w <BotName> co reset / nc reset     ← back to spec defaults
```
Common: `co`: heal, assist, tank, cc, aoe, flee · `nc`: follow, stay, guard, food

### Useful settings (`CoA-Bots/Core/configs/modules/playerbots.conf`, edit while stopped)
| Setting | Default | Note |
|---|---|---|
| `AiPlayerbot.MinRandomBots` / `MaxRandomBots` | 200 | cap to bound RAM (100 bots ≈ half) |
| `AiPlayerbot.BotsWhisperPublic` | on | `0` stops unasked whispers |
| `AiPlayerbot.BotTextLocale` | -1 | 2 = French, 3 German, 6 Spanish, 8 Russian |
| `AiPlayerbot.CoaHealerManaReserve` | 35 | % mana healers keep for heals |

### RAM reality check
`free -h` will show ~20+ GB "used" with a bots server + client up. **Look at `available`** — as long as it's >4 GB you're fine; ~10 GB of "used" is Linux disk cache (auto-reclaimed), and swap usage of a few hundred MB is normal zram breathing. 200 bots ≈ 4–6 GB server-side; the "dynamic brackets" system pulled 700+ bots online around a real player in our test — that's the living-world feature, not a leak.

### Dashboard (optional)
```bash
wine cmd /c "CoA-Bots\Dashboard\Start_Dashboard.bat"
# then open http://localhost:8088  (bot census, levels, chat, epic loot)
```

---

## 12. Daily operations cheat sheet

```bash
# ENTER THE BOX
distrobox enter coa-server
export WINEPREFIX=$HOME/Games/coa-server/wine-prefix
cd ~/Games/coa-server/Server

# START
wine Runtime/python/python.exe -B Scripts/manage.py start-all          # vanilla server
wine Runtime/python/python.exe -B CoA-Bots/coa_bots.py start-all       # server WITH bots
# (bots mode: go away for ~5+ min after World-OK while the bots log in, then play)

# MONITOR
wine Runtime/python/python.exe -B Scripts/manage.py status
tail -f Core/Logs/world-console.log
nc -z 127.0.0.1 3307 && echo DB-OK && nc -z 127.0.0.1 3724 && echo Auth-OK && nc -z 127.0.0.1 8085 && echo World-OK

# GM CONSOLE (also usable from the host)
nc 127.0.0.1 3443        # login: local / local

# STOP — always this, never close terminals
wine Runtime/python/python.exe -B Scripts/manage.py stop-all
ps aux | grep -iE "worldserver|authserver|mysqld" | grep -v grep   # verify empty
wineserver -k   # emergency brake only

# BACKUP (server stopped)
cd ~/Games/coa-server && tar -czf ~/coa-backup-$(date +%F).tar.gz Server/
```

Both modes share one database — characters persist whether you boot with or without bots.

**Two servers at once is impossible** (ports 3307/3724/8085). Always stop-all before switching modes or updating.

---

## 13. Appendix: error → fix quick table

| Error / symptom | Section | Fix |
|---|---|---|
| `Can't create UNDO tablespace innodb_undo_001 ... already exists` | 6.1 | Move `undo_*` files out of `mysql/data/` (to outside the data dir) |
| `Multiple files found for the same tablespace ID` | 6.2 | Your undo backup is INSIDE `mysql/data/` — move it out |
| `'Windows aio' returned OS error 0` then worldserver dies | 6.3 | `--innodb-use-native-aio=0`, patch `manage.py` permanently |
| Installer prints `fixme:powershell:wmain stub` then "Press any key" instantly | 8.4 | Wine PowerShell is fake — use the bash installer (Section 8) |
| `world is still starting` from the bots launcher | 9 | Launcher timeout, not a crash — keep waiting |
| Stuck on "connecting" + `Glue Screen` Lua error | 10 | Bot login storm — wait for world log to go quiet, then log in |
| Realm "offline" | 7 | Fix `acore_auth.realmlist` (`127.0.0.1:8085`) or restart stack |
| Everything died after a crash | 6 | Check undo files (6.1); relaunch; InnoDB recovers automatically |
| `nc: command not found` | — | You're on the host, not in the container (`distrobox enter coa-server`) |
| `mv: cible ... Aucun fichier ou dossier` | — | You skipped `mkdir` of the destination folder |
| Backslash vanishes (`CoA-BotsInstaller-Bots.bat`) | — | Bash ate `\I` — quote paths: `"CoA-Bots\Installer-Bots.bat"` |
| No bots in `/who` after 10 min | 11 | Check `AiPlayerbot.MinRandomBots` in playerbots.conf; check CoaBots.log |

### Golden rules (learned the hard way)
1. **Read a script before running it.** Every `.bat` is a wrapper; `cat` it first.
2. **Never keep InnoDB files in backups inside `mysql/data/`** — InnoDB scans subfolders.
3. **Silence ≠ failure** — MySQL logs to `mysql-error.log` because of `log-error=` in `my.ini`; the live process in the foreground IS the progress indicator.
4. **Never stop by closing terminals** — `stop-all`, then verify, then `wineserver -k` only as emergency.
5. **First bots boot is a marathon** (30–50 min). Daily boots are minutes. Logins during the storm time out; logins after are instant.
6. **Check `available` RAM, not `used`** — and verify with `free -h` before blaming RAM for anything.

---

---

## Credits & Disclaimer

- **Original Windows guide:** by **ScikuVRC (ARACHA)** — this Linux guide follows the same repack, re-targeted for Distrobox + Wine.
- **CoA Repack:** by **Jealous-Sound** & CoA contributors.
- **CoA Bots:** by **SQUID [EXFLIX]** / **Zyth45** (mod-playerbots branch `coa`), with rotations by steviecraycray, StevenLeclerc and the ascensionsidekick.com builds.
- **Foundations:** AzerothCore, mod-playerbots, UnBot, Wine, Distrobox.

*Unofficial community guide — not affiliated with or endorsed by the projects above. Tested on Bazzite 44 (September 2026). Corrections welcome.*

*Guide written from a real install on Bazzite 44, September 2026. Thanks to the CoA repack (Jealous-Sound) and CoA Bots (SQUID/Zyth45) teams — support them in their Discord.*

