# create-port

**Create, ported to Horizon.** A rewrite of the [Create](https://www.curseforge.com/minecraft/mc-mods/create)
mod for [Horizon](https://playhorizon.dev/) — Minecraft: Java Edition 26.3 rebuilt in Rust — targeting Horizon's
own mod system: Rust crates compiled to sandboxed WebAssembly components (`.hmod`).

Java mod jars do not run in Horizon. Fabric, Forge and NeoForge code calls and patches Minecraft's Java classes,
and Horizon has none. So Create is not *loaded* here — it is **rewritten**, subsystem by subsystem, against
Horizon's versioned mod API.

Unofficial. Not affiliated with Mojang, Microsoft, the Create team or the Horizon project.

---

## Licence: what we may take

Create is open source, but its [`LICENSE.md`](https://github.com/Creators-of-Create/Create/blob/mc1.21.1/dev/LICENSE.md)
splits the mod in two, and the split decides the shape of this port:

| Part | Licence | Consequence for us |
| --- | --- | --- |
| **Code** — everything outside `src/main/resources/assets/` | **MIT** © The Create Team | we may read it, translate it to Rust and publish the derivative, with attribution in our `LICENSE` and in `ported-from` of each mod's manifest |
| **Assets** — `src/main/resources/assets/`: models, textures, blockstates, sounds, lang | **All Rights Reserved** | we may **not** copy or redistribute them; they have to come from the player's own Create install |

So the code port is a translation job with no legal ambiguity. Assets are the real constraint, and there is a
working precedent for it: **Create: Adapted**, the community adaptation of Create for Minecraft 26.x, publishes
`create-adapted-…-public-external-assets.jar` — its own code, with assets resolved at runtime from a separately
installed official Create. That is the model for M1: `create-content.hmod` declares the blocks and items and
**references** assets the player already has, or ships only assets we make ourselves.

Upstream, Horizon applies a stricter rule to its own modpack ports — *a mod's code is never decompiled, read,
quoted or paraphrased* — because most mods in All the Mods 10 are ARR, and because they pulled their shader packs
in October 2026 for exactly that contamination. That rule is Horizon's, and Create's MIT licence does not impose
it on us. Two consequences we accept:

- The port lives in **this** repository, where reading Create's source is legitimate. If a port is ever offered to
  Horizon's registry or its `horizon-port-pack` pipeline, that contribution is re-derived under their clean-room
  rule, or cleared with the maintainer first.
- We keep the *behavioural* discipline anyway, because it is what makes a port trustworthy: every subsystem gets a
  harness running the real Create in a dev Java instance against this port — same inputs, compared outputs. The
  source tells you the intent; the harness proves the result.

Never commit Create's assets, its jars, or decompiler output into this repo (see `.gitignore`).

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

## Version target

Create's official releases stop at **`6.0.10+mc1.21.1`** (NeoForge, April 2026); there is no official Create for
Minecraft 26.x, and Horizon pins **26.3**. So the port bridges a version gap as well as an API gap:

- the **oracle** for harnesses runs Create 6.0.10 on NeoForge 1.21.1 — that is the behaviour we match;
- block, item, recipe, loot and data-pack **formats** must be 26.3's, as Horizon reads them (its
  `horizon-port-pack carry` step already converts carried data to the current formats);
- where 1.21.1 and 26.3 disagree on vanilla behaviour, 26.3 wins — that is the game we run in.

## Plan

Milestones, decisions and open questions live in `SPEC.md` (the source of truth; `✅` = decided, `💡` = proposal).
The shape of it:

- **M0 — Recon and oracle.** Create's subsystem inventory (kinetics → processing → logistics → contraptions →
  schematics); a dev NeoForge 1.21.1 instance running Create 6.0.10 as the oracle; the formulas — RPM sources,
  stress costs, gear ratios, processing times — read from the source and cross-checked against the
  [Create Wiki](https://create.fandom.com/); the harness skeleton; the asset-resolution design for the ARR
  problem; the first upstream API-gap issues.
- **M1 — Content parity (API 0.2).** `create-content.hmod`: Create's blocks and items in the `create` namespace
  from vanilla templates, with blockstates, models, textures, recipes, loot and tags resolved **externally** from
  the player's own Create install (Create: Adapted's approach) or from assets we make. Inert, but the world looks
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
├── LICENSE                 MIT, carrying The Create Team's copyright notice for the translated code
├── docs/
│   ├── licence.md          the MIT/ARR split, asset-resolution options, attribution
│   ├── recon/              M0: subsystem inventory, formulas, version coverage
│   └── api-gaps.md         what Horizon lacks, and the upstream issues for it
├── oracle/                 dev NeoForge 1.21.1 instance config, observation and parity harnesses (dev tooling: Python/Java fine)
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
