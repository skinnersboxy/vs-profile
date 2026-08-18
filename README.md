# vs-profile

Isolated mod/save profiles for [Vintage Story](https://www.vintagestory.at/) on Linux,
with optional cloud sync.

Vintage Story keeps one `Mods` folder per data directory, not per world. Every
singleplayer world in an install shares that folder, and joining a modded server
requires your client to match the server's mod set exactly — the game does not
download mods for you. So the moment you play on more than one server, or want a
vanilla world alongside a modded one, you need more than one data directory.

`vs-profile` manages those directories and launches the game against them.

```
vs-profile myserver         # play with that server's mod set
vs-profile vanilla          # a clean profile, no mods
vs-profile list             # what exists, how many mods and saves
```

Each profile is a full data directory — its own `Mods`, `Saves`, `ModConfig`,
keybinds and settings — so mod sets never collide.

## Install

Requires `bash` and `python3`.

```sh
curl -o ~/.local/bin/vs-profile \
  https://raw.githubusercontent.com/skinnersboxy/vs-profile/main/vs-profile
chmod +x ~/.local/bin/vs-profile
```

Game installs are expected at `~/games/vintagestory-<version>`, e.g.
`~/games/vintagestory-1.22.5`. Extract each release tarball to its own versioned
directory and `vs-profile` will find them all.

## Usage

```
vs-profile <name> [game args...]   launch a profile, creating it if new
vs-profile list                    profiles with mod/save counts and sync state
vs-profile mods <name>             list a profile's mods
vs-profile path <name>             print a profile's data directory
vs-profile pin <name> <ver>        always launch this profile on game <ver>
vs-profile unpin <name>            go back to the newest installed version
vs-profile sync <name>             move a profile into your cloud folder
vs-profile unsync <name>           move it back to local disk
vs-profile update <name>           update its mods from ModDB; check for a new game
vs-profile install [<ver>]         install a game version from the CDN

  -g, --game-version <ver>   launch a specific game version (overrides a pin)
  -f, --force                launch despite another machine's sync lock
  -n, --dry-run              update: report what would change, download nothing
      --game                 update: also install a new game version if one is out
```

Profiles live in `~/.config/VintagestoryData-<name>`. Creating one is just
launching it — the directory is made on first use, seeded with your existing
keybinds and graphics settings so a new profile is not bare.

Extra arguments pass through to the game, so `vs-profile myserver --connect
example.com` works.

To add mods to a profile, drop the `.zip` files into its `Mods` folder:

```sh
cp ~/Downloads/*.zip "$(vs-profile path myserver)/Mods"
```

### Multiple game versions

Mods usually target one game version, and a server pinned to an older release
needs a matching client. Point `-g` at whichever install you want:

```sh
vs-profile -g 1.21.6 oldworld
```

A profile that belongs to one server should not have to be told this every
time, so pin it once:

```sh
vs-profile pin myserver 1.22.2   # this profile always launches on 1.22.2
vs-profile unpin myserver        # back to the newest install
```

The pin beats the default and loses to an explicit `-g`, and `vs-profile list`
shows it. Note that a pin is not the same as "the oldest install" or "whatever
was newest when you made the profile" — it is an exact version, so installing a
newer release later cannot silently take a server profile off the version its
server runs. The pin is stored inside the profile directory, so a synced
profile carries it to your other machines; if that version is not installed
there, the launch stops and tells you rather than falling back.

The script also handles the .NET split across versions: Vintage Story 1.22+
requires the .NET 10 runtime, which it looks for at `~/.dotnet`, while 1.21 and
earlier run on the system .NET. Install a user-local runtime with:

```sh
curl -sSL https://dot.net/v1/dotnet-install.sh | bash -s -- \
  --runtime dotnet --channel 10.0 --install-dir ~/.dotnet
```

## Staying up to date

```
$ vs-profile update singleplayer
game:  1.22.7 is the newest stable release and is installed
mods:  1 update(s) for game 1.22.7, 9 already current
       bettererprospecting  3.4.3 -> 3.4.5  (tagged up to 1.22.6, you run 1.22.7)
       bettererprospecting -> 3.4.5  (bettererprospecting_3.4.5.zip)
```

`update` reads each `.zip` in the profile's `Mods` folder, looks its `modid` up
on the [mod database](https://mods.vintagestory.at/), and replaces it with the
newest release that fits the game version *that profile* launches on. Add
`--dry-run` to see the plan without downloading anything.

Which game version that is matters, and it is the reason this is per-profile
rather than one global "update my mods": a profile pinned to 1.22.2 for a server
gets mods for 1.22.2, while an unpinned one gets mods for your newest install.
Updating a profile never moves its pin.

Mod authors stop re-tagging a release long before it stops working, so an update
tagged for an earlier patch of the same minor version is still offered, with a
note saying which way the tag misses — a mod that was last tagged for 1.22.6
while you run 1.22.7 is a different risk from one built against a patch newer
than the version this profile is pinned to. Releases from another minor version
are never offered, and a release candidate is only offered when the mod has no
stable release newer than yours.

The replaced `.zip` goes to `.vs-profile-oldmods/` inside the profile, one
generation deep, so a bad update is a `mv` away from being undone and a synced
profile does not slowly fill up with every mod version it has ever had.

Every update also checks the release index at `api.vintagestory.at` for a newer
game version. Downloading a 600 MB client is not something to do behind your
back, so it is only reported unless you ask for it:

```sh
vs-profile update singleplayer --game   # install the new client too, then the mods
vs-profile install                      # just install the newest client
vs-profile install 1.22.5               # or a specific one
```

`install` verifies the MD5 the index publishes, flattens the tarball's nested
`vintagestory/` directory into `~/games/vintagestory-<version>`, and adds the
`DOTNET_ROOT` export that 1.22+ needs to the shipped `run.sh`. If you keep a
`~/games/vintagestory` symlink pointing at your newest install, it follows —
but one is never created for you.

## Cloud sync

`vs-profile sync <name>` moves a profile into your cloud folder and symlinks it
back, so mods, settings and worlds follow you between machines:

```sh
vs-profile sync survival     # -> ~/Dropbox/VintageStory/survival
vs-profile unsync survival   # move it back
```

Dropbox is auto-detected from `~/.dropbox/info.json`. For anything else, point
`VS_PROFILE_SYNC_DIR` at the local folder your client syncs.

Syncing a game save is genuinely risky, so synced profiles get two safeguards:

**Sync-before-launch.** Vintage Story saves are SQLite databases. Opening one
that is still mid-download gives you a corrupt or half-rolled-back world, so
`vs-profile` waits for the client to report idle before starting the game, and
waits again after you quit so the upload finishes before you close the laptop.
Wait time is capped by `VS_PROFILE_SYNC_WAIT` (default 180s).

**Cross-machine locking.** Running the same synced profile on two machines at
once will corrupt the save, and there is no sane way to merge a conflicted world
file — you will be picking one and losing the other. `vs-profile` writes a
lock naming the current host and refuses to launch if another host holds it:

```
vs-profile: profile is locked by 'laptop'.
  Running it on two machines at once will corrupt the save.
```

Clear a stale lock by deleting the named file, or override with `-f`.

One thing to avoid: do **not** use selective sync to exclude `Logs`, `Cache` or
`Backups`. The game recreates those directories on every launch and the sync
client will fight it, producing an endless trail of `(Selective Sync Conflict)`
folders. Let them sync, or leave the profile local.

## The `modPaths` gotcha

`--dataPath` does **not** fully control where the game looks for mods, which
makes hand-rolled profiles fail in a way that is genuinely confusing to debug.

Vintage Story reads its mod folders from `modPaths` inside
`clientsettings.json`, and that list contains **absolute** paths:

```json
"modPaths": ["Mods", "/home/you/.config/VintagestoryData/Mods"]
```

Copy a `clientsettings.json` between profiles — the obvious way to carry your
keybinds over — and the new profile silently keeps loading the *old* profile's
mods. `--dataPath` still redirects saves, logs and configs, so the profile looks
like it is working. The only symptom is the wrong mod list, and the game's own
"Open mods folder" button opens the stale directory.

`vs-profile` pins `modPaths` to the profile's own `Mods` directory on every
launch, so a profile seeded from another one repairs itself.

If you are doing this by hand instead, either delete `clientsettings.json` from
the new profile and let the game regenerate it, or edit `modPaths` to match.

## License

MIT
