# memods-db

The trainer database for [memods](https://github.com/hectormiranda8/memods).

Every trainer here is **data**, not an executable. A pack is a JSON document
describing anchors, pointer chains, byte patches and — where it needs them —
x86-64 instructions as assembly *text*, assembled locally by memods at load
time. Nothing served from this repository is ever executed as code memods did
not build itself.

That is the point. You can read a diff and know what a cheat does before you
run it.

## Layout

```
packs/<game-id>/<game-id>.memods.json   one trainer per game
art/<game-id>.jpg                       cover art
index.json                              generated catalog (id, hash, version)
.claude/skills/memods-trainer/          the authoring skill
```

## Adding a trainer

The short version:

```bash
memods-cli validate packs/<game>/<game>.memods.json --strict
```

The long version is the [`memods-trainer` skill](.claude/skills/memods-trainer/SKILL.md),
which walks the whole pipeline — research, triage, authoring, verification.
Working in this repository with Claude Code loads it automatically.

Format reference lives in the app repository: [`docs/pack-format.md`](https://github.com/hectormiranda8/memods/blob/main/docs/pack-format.md)
and [`schema/memods-pack.schema.json`](https://github.com/hectormiranda8/memods/blob/main/schema/memods-pack.schema.json).

## The two rules that matter

**Published offsets are a hypothesis, not a result.** Google and GitHub are full
of trainer repositories carrying confident-looking offsets that cannot possibly
work — wrong engine, invented layouts, "trainers" whose code never attaches to a
process. `memods-cli validate` catches the impossible ones statically; the rest
is on you to verify against the running game.

**A pack publishes only after surviving a cold restart.** Every value resolves
and passes its invariants from a fresh launch, every cheat toggles on with a
visible effect, every cheat toggles off and reverts. Addresses found in a warm
session prove nothing. Record it:

```json
"provenance": { "verified": { "date": "2026-08-21", "build": "1.16" } }
```

## Scope

Single-player focused. Anti-cheat is *declared* so memods can report it and tell
players how a given game is normally launched without it — never to block
anything. Set `"online_risk": "high"` on cheats that would matter in a connected
session, and let people decide for themselves.

## License

MIT.
