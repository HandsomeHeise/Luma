# Luma

Source repositories: [Luma](https://github.com/HandsomeHeise/Luma) ·
[RAGE Script Manager](https://github.com/HandsomeHeise/RageScriptManager).

A GoldHEN plugin that gives **RAGE games on PS4 a Lua runtime** — one `.prx` that
serves both Grand Theft Auto V and Red Dead Redemption 2.

It boots Lua 5.4 inside the game, binds the game's own native functions as ordinary
Lua functions, and runs your code on the game thread once per frame. With the
[VS Code extension](https://marketplace.visualstudio.com/items?itemName=DominikHeise.rage-script-manager)
you save a file and it is running on the console a moment later — no game restart.

**📖 [Documentation](https://xeghosted.github.io/luma/)** — install, scripting, the
control protocol, and how it works inside.

```lua
on_tick(function()
    local ped = GET_PLAYER_PED(-1)
    local c   = GET_ENTITY_COORDS(ped, true)   -- Vector3 returns arrive as {x, y, z}
    DRAW_RECT(0.5, 0.05, 0.2, 0.04, 0, 0, 0, 180, 0)
end)
```

## The two games

Luma identifies the game it was loaded into by a byte signature and switches
everything that differs behind one profile: native table, frame hook, data root,
control port.

| | Grand Theft Auto V | Red Dead Redemption 2 |
| --- | --- | --- |
| Build | CUSA00411 v1.57 | CUSA03041 v1.32 |
| Data root | `/data/gtalua/` | `/data/rdr2lua/` |
| Control port | 9615 | 9616 |
| Natives | 6,485 bound: 2,811 by address, 3,674 resolved by walking the masked registry at boot | 7,072 bound, all by address, compiled in from the eboot |
| Script globals | available | not located in this build |

The data roots are deliberately separate: one console can host both, and a script
must never land in the other game's `scripts/` directory.

**An unknown build does not load.** Luma writes the bytes it found at the signature
address into its log and stops, rather than running on with an address table that
belongs to a different executable. An RVA is valid for exactly one build.

## Quick start

```sh
# build (needs the OpenOrbis toolchain with OO_PS4_TOOLCHAIN set, under WSL or Linux)
./build.sh                      # -> build/Luma.prx
bash tests/run.sh               # host tests: bridge, marshalling, net, resources, Lua

# deploy
pwsh tools/deploy.ps1 -Ip <your console's IP> -Game gta5     # or -Game rdr2
```

Then list the plugin in `/data/GoldHEN/plugins.ini` under the game's title id and
start the game:

```ini
[CUSA00411]
/data/GoldHEN/plugins/Luma.prx
[CUSA03041]
/data/GoldHEN/plugins/Luma.prx
```

The full walkthrough, including what a healthy first boot looks like, is in the
[install guide](https://xeghosted.github.io/luma/install.html).

## Two things worth knowing before you start

**There is no authentication unless you add it.** Anyone who can reach the control
port can run arbitrary Lua inside the game. That is the same trust level GoldHEN's
FTP already grants on the same network: fine on a home LAN, not fine anywhere else.
Write a secret to the game's `token` file and set `rageScriptManager.token` to match,
and the channel refuses everything else.

**A new `.prx` needs a game restart.** Scripts and resources reload live; the plugin
itself is mapped at title launch. `deploy.ps1` stages the file beside the target and
renames over it, because overwriting a mapped plugin in place succeeds and leaves the
file holding neither build.

## Known gap

`script_global()` and `set_script_global()` refuse for RDR2. The script-global block
table has not been located in that build, so `profile.script_globals_rva` is 0 and
every caller raises rather than reading — and writing — an address derived from
nothing. GTA V carries it at `0x3E0CFB8`, inherited from the project this grew out of; it has not been re-verified on hardware since the merge. Everything else has, on both games.

## How the RDR2 native table was recovered

This is the part that is genuinely different, and the reason the port was tractable.

GTA V masks its native registry: every field is XOR-encoded and the table only exists
in the running process, so Luma walks it at runtime and verifies the result before
trusting it. **RDR2 does none of that.** The compiler emitted every registration
inline and in the clear:

```asm
movabs  rcx, 0x4EDE34FBADD967A6      ; the hash        (WAIT)
lea     rdx, [rip + 0x3906ef3]       ; the handler
mov     [r14 + rax*8 + 0x48], rcx    ; node->hashes[i]
mov     [r14 + rax*8 + 8],  rdx      ; node->funcs[i]
```

`tools/extract_registry.py` decodes those store pairs and recovers all 7,072
`hash -> RVA` mappings from `eboot.bin` alone — no console, no disassembler, about
four seconds. `tools/gen_natives.py` joins that against names and signatures **on the
hash**, never on position, which is what stops a wrong name dragging a wrong address
along with it.

```sh
python tools/extract_registry.py /path/to/eboot.bin -o tools/native_registry.json
python tools/gen_natives.py
```

`tests/run.sh` gates the committed bindings against a fresh regeneration, so they
cannot drift from the registry unnoticed.

## What has actually been proven on hardware

Both games, from their own boot logs:

```
[profile] Grand Theft Auto V (CUSA00411), data root /data/gtalua/, port 9615
[hash_natives] walked table: 6691 natives, 0 rejected
[hash_natives] GET_HASH_KEY -> 0x72B71C36 (OK, table usable)

[profile] Red Dead Redemption 2 (CUSA03041), data root /data/rdr2lua/, port 9616
[hash_natives] 7072 natives compiled in; GET_HASH_KEY -> 0x3DBB3D48 (OK)
player model is 2.01 m tall
```

Each line settles something different:

| evidence | what it establishes |
| --- | --- |
| `GET_HASH_KEY -> …` | the game computed the joaat hash itself and matched the predicted value — the registry and the base + RVA arithmetic are right |
| `frame hook is alive` | the detour and its trampoline survive being relocated |
| real world coordinates | Vector3 **returns** decode from the padded three-slot layout |
| `0.0 m/s` | float returns |
| `2.01 m tall` | Vector3 **out-params**: two padded 24-byte buffers through `set_vector_results` — the path that silently returned `{0,0,0}` for a long time, so a non-zero answer is the one that matters |

## Layout

```
src/rage/invoker/     the native call ABI, shared by both games
src/game/             profile detection; per-game native resolution
src/script/           Lua runtime, the resource model, the script loader
src/net/              the control channel
scripts/{gta5,rdr2}/  each game's flat scripts and generated natives.lua
resources/{gta5,rdr2}/ each game's resources
editor/lua-defs/      LuaLS definitions, per game, for autocomplete
tools/                extraction, generation, deploy, a shell client
tests/                host tests — everything below the ORBIS line runs on a PC
```

## Licence

MIT. See [LICENSE](LICENSE).

This project talks to games it does not ship, distribute or modify on disk. It
contains no game code — only addresses and hashes derived from a binary the user
already owns, the same category of information as the public
[PS4 offset catalog](https://xeghosted.github.io/ps4-offsets/).
