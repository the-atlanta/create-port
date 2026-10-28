# create-port

**Create, ported to Horizon.** A clean-room rewrite of the [Create](https://www.curseforge.com/minecraft/mc-mods/create)
mod for [Horizon](https://playhorizon.dev/) — Minecraft: Java Edition 26.3 rebuilt in Rust — targeting Horizon's
own mod system: Rust crates compiled to sandboxed WebAssembly components (`.hmod`).

Java mod jars do not run in Horizon. Fabric, Forge and NeoForge code calls and patches Minecraft's Java classes,
and Horizon has none. So Create is not *loaded* here — it is **rewritten**, subsystem by subsystem, against
Horizon's versioned mod API.

Unofficial. Not affiliated with Mojang, Microsoft, the Create team or the Horizon project.
Create is © simibubi and the Create team (MIT). Horizon's original work is MIT.

---

## Why this is hard (read before writing code)

Create is not a content mod. Its substance is a **simulation**: rotational power networks with RPM, stress and
overstress, kinetic block entities that transfer and consume it, machines that transform blocks and items in the
world, and *contraptions* — structures that detach from the world, move as entities, and reattach.

Horizon's mod API today (see [`docs/mods.md`](https://github.com/horizon-rust/horizon/blob/main/docs/mods.md)):

| Horizon gives us | Line | Enough for |
| --- | --- | --- |
| `registries` — blocks/items from JSON on vanilla templates, plus recipes, loot, tags, models, textures, structures | 0.2 content | Create's blocks and items as **data**: visible, craftable, placeable — inert |
| `world` — `get-block`, `get-biome`, `set-blocks` (as `/setblock` makes them), `players`, typed server commands, `storage`, hooks for tick / join / chat / break / place | 0.3 server (experimental) | the **kinetic simulation itself**, owned by the mod |
| `game.raycast`, `game-read`, `hud`, `render` (HUD shapes, corners), `ui` screens, `recipes` | 0.1 client | a Create-style **debug overlay**: what network am I looking at, RPM, stress |

| Horizon does **not** give us yet | Blocks Create on |
| --- | --- |
| block entities, ticking machines with slots and screens | every kinetic block's own state and inventory |
| entities: query, spawn, ride | **contraptions** — the moving-structure half of Create |
| world-space rendering (lines, boxes, billboards in the world) | rotation indicators, gauge needles, moving structures |
| item stacks, inventories, item transport | belts, chutes, funnels, arms — item logistics |
| `net`: a typed channel between a mod's client and server half | any client that must *see* the simulation |

The consequence shapes the whole architecture: **the mod owns the simulation, the game only projects it.**
There is no block entity to hang a shaft's state on, so the state lives in the mod — a shadow model of every
kinetic network, keyed by block position, ticked on the server tick, persisted in `storage`, and made visible by
`set-blocks` for anything the world itself must show (a block breaking, a processed output appearing) and by the
client overlay for anything it must not (RPM, stress, rotation direction). Horizon's docs call an API shortfall
like this *an API gap*, and gaps become issues upstream — this port is expected to file them, and to shrink as
they close.

## Clean-room rule

Horizon's own rule for ports, and this repo's hard rule:

> A mod's code is never decompiled, read, quoted or paraphrased. Ports are written from the mod's observed
> behaviour, its own data files and its public documentation only.

Concretely, in this repo:

- **Never** open, read, summarise or paste Create's Java source — not the jar, not the GitHub repo, not a
  decompiler's output, not an AI summary of it. This includes "just to check how they did it".
- **Do** run Create in a dev Java instance and observe it (`oracle/`): what a block does, what it renders, what
  it prints, what its screens show, what NBT it stores.
- **Do** read Create's own data files (its `data/` and `assets/` — models, blockstates, recipes, loot, lang,
  textures) and the [Create Wiki](https://create.fandom.com/) and other public documentation, where the formulas
  (RPM sources, stress costs, gear ratios, processing times) are published.
- **Do** write a harness per subsystem: the real Create mod in the dev oracle vs this port, same inputs, compared
  outputs. A subsystem is done when the harness says the two agree — not when it looks right.

Assets carried over from Create's jar stay under Create's licence with attribution; per-file licence status is
checked in M0 before anything ships.

## Plan

Milestones, decisions and open questions live in `SPEC.md` (the source of truth; `✅` = decided, `💡` = proposal).
The shape of it:

- **M0 — Recon and oracle.** Create's real version coverage against Horizon's pinned 26.3; a dev Java instance
  running it; a subsystem inventory (kinetics → processing → logistics → contraptions → schematics); the formulas
  from public docs; the harness skeleton; the licence and asset audit; the first upstream API-gap issues.
- **M1 — Content parity (API 0.2).** `create-content.hmod`: Create's blocks and items in the `create` namespace
  from vanilla templates, with models, textures, blockstates, recipes, loot and tags. Inert, but the world looks
  like Create and crafting works.
- **M2 — Kinetic core (API 0.3 server).** The shadow simulation: networks, rotation sources, shafts/cogs/gearboxes
  as propagation, RPM and stress, overstress and its consequences. Pure Rust maths in the sandbox, fuel-budgeted.
- **M3 — Projection and inspection.** `set-blocks` for observable effects; typed commands (`/create network`,
  `/create rpm`); the client overlay via `raycast` + `hud`/`render` — look at a shaft, see its network.
- **M4 — Machines.** The processing blocks that only need the world: pressing, crushing, mixing, cutting, as far
  as `world` and the recipe tables carry them.
- **M5 — Contraptions.** Blocked on entities, world-space render and `net`. Tracked, designed, and started the
  moment the API allows — or prototyped earlier as a static-structure approximation if that survives the harness.

Ordering follows Horizon's own: content first, then server hooks, then whatever the API unlocks next.

## Repository layout

```
create-port/
├── SPEC.md                 decisions, milestones, open questions — source of truth
├── docs/
│   ├── clean-room.md       the observation rules above, in full
│   ├── recon/              M0: subsystem inventory, formulas, version coverage
│   └── api-gaps.md         what Horizon lacks, and the upstream issues for it
├── oracle/                 dev Java instance config, observation and parity harnesses (dev tooling: Python/Java fine)
├── mods/
│   ├── create-content/     API 0.2 content mod — data only, no code
│   └── create-kinetics/    API 0.3 server mod — Rust → WebAssembly
└── tools/                  data extraction, model/texture/lang conversion, harness drivers
```

Everything players run is Rust compiled to WebAssembly. Python and Java are allowed for dev tools and harness
drivers in `oracle/` and `tools/`, never in a shipped mod.

## Building

Horizon's toolchain, per its [mod guide](https://playhorizon.dev/docs/make-a-mod):

```
rustup target add wasm32-unknown-unknown
cargo install --git https://github.com/horizon-rust/horizon cargo-horizon-mod
```

Then, in a mod folder:

```
cargo horizon-mod build      # → target/horizon-mod/<id>.hmod, imports checked against the manifest
cargo horizon-mod run        # start Horizon with it as a dev mod; rebuild and the game reloads it
cargo horizon-mod test       # headless: run tests/*.toml scripts in the same sandbox
cargo horizon-mod check <f>  # what the registry's CI runs
```

Mods go in `horizon-mods/` beside the game's `options.txt` and are switched on in Options > Mods.

## Status

**M0 — not started.** No code, no assets, no oracle yet.
