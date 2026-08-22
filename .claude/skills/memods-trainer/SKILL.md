---
name: memods-trainer
description: Research, verify and author a memods trainer pack for a game — from web research and published offsets, from live memory probing, or from offsets the user already has. Use whenever building or repairing a trainer, converting a Cheat Engine table, or evaluating offsets found online. Encodes the failure modes that waste whole sessions, chief among them trusting published offsets.
---

# Building a memods trainer

Finding offsets is easy. Google, GitHub and the Cheat Engine forums are full of
them. **Almost none of them are trustworthy**, and the entire craft of this skill
is telling the difference.

A published offset is a **hypothesis**. It earns trust by resolving against the
running game and passing invariants — never by looking confident.

---

## 0. Which route are you on?

Ask the user before doing anything. Any one of these is sufficient; do not
assume a tool is installed.

| Route | They bring | You do |
|---|---|---|
| **A** | Offsets from Cheat Engine, x64dbg, or a table | Go straight to §3, then verify |
| **B** | Nothing | Research (§2), propose, confirm in game |
| **C** | A probing tool | Drive it (§4) |
| **D** | Partial info | Fill gaps around what exists |

Route A with no probing tooling installed at all is a fully supported path. Do
not push the user toward a tool they did not ask for.

---

## 1. Establish the runtime FIRST — it decides everything

Before evaluating a single offset, determine what the game is built with. This
is both the cheapest fraud detector you have and the constraint every later
decision hangs off.

Two independent sources, and they must agree:

**Web**: search the engine. "«game» Godot / Unity / Unreal / IL2CPP / engine".
Studios announce migrations; press covers them.

**The process**, if it is running:

```bash
memods-cli probe "<game>.exe" --modules
```

It prints the runtime it detected from the module list:

| Modules present | Runtime | Durable anchors |
|---|---|---|
| `GameAssembly.dll` | `il2cpp` | `signature`, `module_rva` |
| `mono*.dll` | `mono` | `mono_field` **by name** — best there is |
| `coreclr.dll` + `clrjit.dll` | `coreclr` | `dotnet_type`, `shape` |
| `*-Win64-Shipping.exe` | `unreal` | `signature`, engine globals |
| none of the above | `native` | `signature`, pointer chains |

A large main executable is **not** evidence of IL2CPP. Check the modules.

Two consequences that catch people out:

- On **coreclr** and **mono**, code is JIT-compiled: it lives outside any
  module, gets a new address every launch, and moves again when a hot method is
  recompiled. **A code signature is not a valid anchor there.**
- On those same runtimes the collector **relocates live objects**, so an address
  goes stale while the game is running, not just between launches.

---

## 2. Triage published offsets before believing them

This section exists because of a real case. A public "Slay the Spire 2 trainer"
repository ships an `offsets.h` with tidy namespaces, a module base, and
sequential field offsets:

```c
Offsets::BaseGameAssembly       = 0x180000000;
Offsets::Player::LocalPlayerController = 0x2A4F880;
Offsets::Player::Health         = 0x1C;
Offsets::Player::MaxHealth      = 0x20;
Offsets::Player::Gold           = 0x30;
```

Every line of it is fiction. Slay the Spire 2 is a **Godot** game on CoreCLR;
`GameAssembly.dll` is a **Unity IL2CPP** artifact and does not exist in the
process. Probing the real game finds health at `0x84` on an object with a
managed type handle at `+0x0` — nothing like `0x1C` inside a Unity module.

Run these checks on anything you find:

**Engine contradiction.** Does the module named match the engine from §1?
`GameAssembly.dll` for a non-Unity game, `mono-2.0-bdwgc.dll` for a native game,
a fixed module RVA for a managed static — each is impossible, not merely
unlikely. `memods-cli validate` now rejects these outright.

**Managed-object shape.** On coreclr/mono an object starts with a type handle
(8 bytes) plus a sync-block word. Field offsets that start below `0x10` on a
managed object are wrong on their face.

**Does the "trainer" actually attach?** Read the code. The repository above
contains a Python file whose game loop *simulates* damage and gold rather than
reading any process. A trainer that never calls `ReadProcessMemory` is not a
trainer.

**Tidiness.** Real reverse-engineered layouts have gaps, padding and odd
offsets. Perfectly sequential `0x1C, 0x20, 0x24, 0x28` across every subsystem is
what a language model produces when asked to invent plausible offsets.

**Provenance.** Is there a recorded game build? Real commit history, or a single
dump? An `.exe` in Releases and no source that produces it? Many such repos
exist to distribute the binary, not the source.

**Staleness.** Even genuine offsets rot. A pointer chain from a table two
patches old is a starting hypothesis, not an answer.

Record every source you used in `provenance.sources`, including the ones you
rejected and why. The next person re-treads that ground otherwise.

---

## 3. Converting offsets you already have

A Cheat Engine chain `[["game.exe"+0x1A2B3C]+0x10]+0x84`:

```json
"anchor": { "kind": "pointer", "module": "game.exe", "rva": "0x1A2B3C", "offsets": ["0x10"] },
"chain":  [{ "offset": "0x84", "means": "hp" }]
```

memods normalises `pointer` into `module_rva` plus hops on load, inserting the
leading dereference of the static. Offsets bind as `"0x84"`, `132` or `"132"` —
write them however the tool printed them.

Then **immediately** confirm it resolves:

```bash
memods-cli probe "<game>.exe" --pack packs/<game>/<game>.memods.json
```

Any value that does not resolve is not a value you have.

---

## 4. Probing, when you need to find them yourself

Use whatever is available. If the `agentcheat` MCP server is configured, **load
its skill and follow it** — it encodes the scanning failure modes and this one
does not restate them. Otherwise step the user through Cheat Engine.

Two rules that survive whichever tool you use:

**Scan for the rarest number the player can see** — gold, XP, currency. Never
start on a small integer; energy `3` appears thousands of times per megabyte and
saturates any candidate cap before reaching the real object.

**Never let the value change mid-scan.** Ask the user to change it, then *wait
for confirmation* before filtering. A filter returning zero survivors after a
confirmed change usually means the scan raced the game, not that the value moved.

Then map the struct around what you pinned rather than hunting each field
separately. Neighbouring fields are usually right there, and the offsets are the
durable part.

---

## 5. Authoring the pack

Structure and field reference: `docs/pack-format.md`. Schema:
`schema/memods-pack.schema.json`.

### Anchors, best first for the runtime

Consult §1. In short: `mono_field` on Mono; `shape` or `dotnet_type` on CoreCLR;
`signature` on native/IL2CPP/Unreal; `module_rva` when the value is a module
static. `absolute` is a placeholder while hunting and is refused when publishing.

### Invariants are the load-bearing part

They are what turns a stale pack into a refusal instead of a corrupted save.

```json
"invariants": ["0 < value <= max_hp", "1 <= max_hp <= 2000"]
```

- **Prefer cross-field checks to ranges.** `0 <= value <= 10` matches a dead
  pooled object holding zero exactly as well as the live one.
  `value <= max and 1 <= max <= 10` does not.
- **Write ones that would actually fail** if the layout shifted.
- **Never loosen an invariant to make resolution pass.** A failing invariant is
  the system working.

### Two traps specific to authoring

**Never pin a value your own cheats edit.** A shape anchor discriminating on
`gold == 1234` stops finding the object the moment a gold cheat succeeds — the
trainer works exactly once and then reports itself stale. Seed on something rare
*and stable*: an entity id, a type tag, a maximum.

**Never name a field from one coincidence.** A counter reading 10 while the deck
holds 10 cards is not a deck counter. An unidentified field is still a perfectly
good *constraint* — just do not give it a name you cannot defend.

### Cheats

`freeze` holds a value; `write` sets it once; `nop` deletes an instruction
(the ammo decrement); `patch` replaces bytes; `codecave` runs authored assembly.

Prefer `"to": "@max_hp"` over a large constant. Many games treat health above
maximum as a corrupt state and clamp, crash, or fail the save.

There is **no undo operation** and none is needed — memods records original
bytes and reverts automatically on disable, on game exit, and on close.

### Anti-cheat

If the game ships with EAC/BattlEye, declare it *with guidance*:

```json
"anticheat": {
  "modules": ["EasyAntiCheat*"],
  "guidance": "Launch offline via Steam → Play Offline, and stay disconnected."
}
```

Detection without guidance is just a scary message. memods reports and never
blocks — plenty of these games are played single-player offline. Set
`"online_risk": "high"` on cheats that would matter in a connected session.

---

## 6. The verify gate — nothing publishes without it

**Restart the game cold.** Then:

1. `memods-cli probe "<game>.exe" --pack <pack>` — every value resolves and
   passes its invariants from a fresh launch.
2. In memods: toggle each cheat on, confirm the effect **in game**, toggle off,
   confirm the revert.
3. On coreclr/mono: play long enough to force a collection and confirm the
   freeze survives.
4. Deliberately corrupt one offset and confirm memods refuses rather than
   writing garbage.

Addresses found in a warm session prove nothing. That is the whole reason this
gate exists.

Record the result:

```json
"provenance": { "verified": { "date": "YYYY-MM-DD", "build": "1.16" } }
```

---

## 7. Publish

```bash
memods-cli validate packs/<game>/<game>.memods.json --strict
```

Strict adds the publishing rules: no session-only anchors, provenance present,
invariants on every value. Then open a PR against the database repo.

---

## Reading failures correctly

| Symptom | Almost always means |
|---|---|
| `AnchorUnresolved` — module not loaded | The engine is not what the offsets assumed. Re-read §2. |
| `AnchorUnresolved` — signature not found | The game was patched since the pack was written. |
| `AnchorAmbiguous` | Signature matched twice; it identifies nothing. Widen or re-derive. |
| `NullPointer` | The object is not live yet — not in a run, not in combat. |
| `ChainReadFailed` at hop 1 | The base is stale; the chain itself may be fine. |
| `InvariantsFailed` | Working as intended. **Do not loosen the invariant.** |
| Cheat works once, then "stale" | A shape anchor pinned a value the cheat edits. |
| Freeze drifts on coreclr/mono | Expected without re-resolution; check the runtime is declared. |

---

*A pack that fails loudly has done its job. The dangerous outcome is a confident
wrong number — which is why invariants exist, and why published offsets are
hypotheses until the running game says otherwise.*
