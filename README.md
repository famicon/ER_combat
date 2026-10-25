# Elden Ring Combat Rewrite — Minecraft port

A Fabric port of [Funny-Bones/ELDEN-RING-Combat-Rewrite](https://github.com/Funny-Bones/ELDEN-RING-Combat-Rewrite),
the Rust/Bevy sandbox that reimplements Elden Ring's player combat from
extracted game data. This mod runs the same mechanics on a Minecraft player:
equipment load decides your roll, iframes are frame-exact, guards can be
countered, power stance mirrors every swing, and the blade only hurts anything
on the frames the animation actually passes through the target.

* Minecraft **26.1.1**, Fabric loader **0.20.1**, Fabric API **0.145.4+26.1.1**
* Java **25**, Gradle **9.8.2**, Mojang official mappings
* MIT, like the original

The server is authoritative: it steps a 60 Hz pure-Java simulation of the
player and the sparring dummy, and the client only sends input and draws what
comes back. That is why dodges, guard counters and hit trades agree between
both sides.

---

## Build and run

```sh
./gradlew build          # jar in build/libs/ercombat-0.1.0.jar
./gradlew runClient      # dev client
./gradlew runServer      # dev server (needs run/eula.txt)
./gradlew test           # 43 simulation behaviour tests, no Minecraft needed
```

Drop the jar in `mods/` with Fabric API. The mod is required on both sides:
the client owns the animation rig and the input snapshot, the server owns the
simulation.

### In game

Combat mode is **off by default**, so a server with the mod installed plays
like vanilla until a player opts in:

```
/ercombat on          # or press B
/ercombat off
/ercombat status      # HP, stamina, weapon, current clip
/ercombat dummy       # make the sparring dummy hostile
/ercombat calm        # ...and passive again
/ercombat reset       # restore HP/stamina/state, respawn the dummy
/ercombat lock        # toggle lock-on without the key
/ercombat weapon greatsword
/ercombat left shield
/ercombat load heavy  # light | medium | heavy
/ercombat arena       # raise a flat stone arena under you
```

While combat mode is on, vanilla attacks, item use, block digging and player
movement handling are suppressed for that player: the simulation moves the
body, not the vanilla controller.

---

## Controls

| Action | Default key |
| --- | --- |
| Light attack (chains 1→6) | Left mouse |
| Guard / left-hand attack | Right mouse |
| Dodge (tap) / sprint (hold) | Left Shift |
| Jump / jump attack | Space |
| Movement | WASD |
| Heavy attack (and guard counter) | `V` |
| Crouch (and crouch attack) | `C` |
| Walk | `Left Alt` |
| Lock on | `X` |
| Two-hand right weapon | `Z` |
| Two-hand left weapon | `Y` |
| Next right weapon | `G` |
| Next left weapon | `H` |
| Toggle combat mode | `B` |

All of them are rebindable in the vanilla **Elden Ring Combat** category.

Left mouse runs the right-hand light chain. Right mouse guards while held when
the off hand is a shield, and runs the left-hand chain when the off hand holds
a weapon — if both hands hold the same weapon that chain is the paired
(power-stance) chain instead. `V` is the heavy attack; a tap starts a charged
heavy that can be released early into the uncharged one, and if you have just
blocked a hit it becomes the guard counter instead. Shift is a tap-to-roll,
hold-to-sprint button: the roll only fires on release if you let go inside the
sprint threshold.

---

## What is implemented

**Movement and stamina**

* Walk / run / sprint with Elden Ring's speeds, acceleration and turn rates;
  sprinting drains stamina and ends in a sprint stop.
* Dodge rolls with a direction (forward, back, left, right), and **roll frames
  depend on equipment load** (`light`, `medium`, `heavy`) exactly like the
  original's equip-load table. Backstep is its own cheaper action.
* Jumps are simulated (`GRAVITY`, terminal velocity, heavy landing, fall
  damage thresholds) rather than left to vanilla.
* Crouch walk, crouch run, and a stamina pool with regeneration and a reduced
  regeneration rate while guarding.

**Attacks**

* Six-hit light chains, six-hit left-hand chains, and power-stance chains when
  both hands hold a weapon.
* Running light/heavy, rolling attack, backstep attack, crouch attack, jump
  attack with separate long-landing and short-landing follow-ups.
* Charged heavies (`HEAVY_1_CHARGE` / `HEAVY_2_CHARGE`) that release early into
  the uncharged variant, using the per-action charge window.
* 25 weapon classes (dagger through colossal weapon, plus fist, claw, torch
  and shield) with their own attack rating, weight, reach (two blade segments)
  and chain length. Reach and swing shape come from the animation's blade
  path, so a spear hits further than a dagger on the same frame.
* Hit detection runs off the animation: `BladePath` gives the blade's sweep for
  each swing, and only the frames marked active can connect. Each action has
  its own hit table (damage multiplier, poise, hit-stop, hurt level).

**Defence**

* Guard with a stamina cost per blocked hit, guard break when stamina runs
  out, and a guard-counter window after a successful block.
* Iframes per action (the roll's window comes from the action table), plus
  jump-over evasion: low sweeps can be jumped instead of rolled.
* Hurt reactions in four levels (small, middle, large, knockdown) with the hit
  direction choosing which way the body is thrown; knockdowns have their own
  iframes and get-up time.
* Hit-stop: the attacker and victim freeze for a fraction of the hit's stop
  value, so heavy hits feel heavy.

**Items and loadout**

* 25 `WeaponItem`s whose held mainhand item *is* the simulation's weapon; the
  next-weapon keys move the selected hotbar slot (right hand) or offhand item
  (left hand) so the model matches the sim.
* Equipment load is derived from the carried weapons, and it changes the roll.
* Two-handing either hand switches the moveset and puts the off hand on the
  haft.

**Presentation**

* `CombatHud`: HP and stamina bars, weapon and grip label, iframe and guard
  counter lights, the target's health bar, a lock-on reticle, the combat log
  and a "YOU DIED" banner.
* Lock-on steering: the body turns to the target while the camera stays free,
  and the movement input is remapped to the lock direction.
* The sparring dummy telegraphs its swings: the husk is posed from the
  simulation state (overhead or low-sweep windup, strike, recovery, stagger),
  so the tell you read is the animation the hit frames are built from.
* One position packet per server tick with a zero rotation delta, so the
  simulated body reaches the owning client without disturbing the camera, and
  motion is interpolated instead of snapping.

---

## Animation

The upstream repository does **not** contain the animation data. Its
`assets/player_anims.bin` is generated from a retail copy of the game by
`tools/bake_anims.py`, and the proprietary clips and sounds are not
redistributable. So the port has two layers:

1. **Procedural clips** (`net.ercombat.client.anim.Animator`) — every action
   the simulation can report has a hand-authored pose curve on the vanilla
   player model: idle breathing, run/sprint cycles, crouch, air, roll, backstep,
   sprint stop, jump, landings, guard, guard hit, guard break, hurt (all four
   levels, per direction), death, and one swing curve per blade path
   (overhead, horizontal left/right, rising, thrust, low sweep, spin). The
   curves are keyed to the same frame numbers the hit tables use, so the blade
   arrives when the damage does.
2. **Retargeted clips** — if you build `player_anims.bin` yourself, convert it
   and the animator prefers the real animation for any clip it knows.

### Retargeting the Elden Ring rig

```sh
python tools/anim/er_to_minecraft.py \
    --input assets/player_anims.bin \
    --output src/client/resources/assets/ercombat/animations/clips.json
```

The tool reads the `ERAN` file format (magic, joint/oriented/clip counts,
length-prefixed names, then per clip: name, blend, frame count and
`frames × stride` little-endian floats) and writes the clip sheet the client
loads from `/assets/ercombat/animations/clips.json`.

The ER skeleton has forty joints and the player model has six boxes, so each
Minecraft part is driven by the direction between two joints:

| Minecraft part | Elden Ring joints | Meaning |
| --- | --- | --- |
| `root` | `Pelvis` → `Spine1` | whole-body tilt, plus a vertical crouch offset |
| `body` | `Spine1` → `Spine2` | torso lean |
| `head` | `Neck` → `Head` | head pitch |
| `rightArm` / `leftArm` | `R_UpperArm` → `R_Hand`, `L_UpperArm` → `L_Hand` | upper arm, or forearm when no hand joint |
| `rightLeg` / `leftLeg` | `R_Thigh` → `R_Foot`, `L_Thigh` → `L_Foot` | thigh, or calf when no foot joint |

Only the pitch and roll that a rigid box can show are kept; yaw is left to the
simulation, which already knows which way the body faces. The vertical offset
is measured against the tallest standing pelvis height found anywhere in the
file, so crouches and knockdowns drop the model instead of sinking it through
the floor.

Axis conventions differ between the bake script and Minecraft's model space
(+Y down, −Z forward), so the mapping is a per-axis sign triple:

```sh
--axes x,-y,-z    # default
--axes -x,-y,z    # mirrored / upside down
--axes x,-y,z
```

`--loop Idle,Run,Sprint` marks clips to wrap instead of clamp, and `--pretty`
indents the JSON. The sheet is a classpath resource, so rebuild the jar after
regenerating it.

### Clip sheet format

```json
{
  "framesPerSecond": 30,
  "clips": {
    "a000_000000": {
      "blend": 3.0,
      "loop": true,
      "root": [[pitch, yaw, roll, verticalOffset], ...],
      "head": [[pitch, yaw, roll], ...],
      "body": [[pitch, yaw, roll], ...],
      "rightArm": [[pitch, yaw, roll], ...],
      "leftArm": [[pitch, yaw, roll], ...],
      "rightLeg": [[pitch, yaw, roll], ...],
      "leftLeg": [[pitch, yaw, roll], ...]
    }
  }
}
```

Frames are sampled with interpolation; a missing part array is simply not
overridden by that clip, so a partial sheet can replace, say, just the attack
swings and leave everything else procedural. Clip names are the simulation's
animation names (the game's ids such as `a000_027000`, or the port's own
`Idle`, `Run`, `Crouch`, `AirAttack:<id>`); the client strips the
`AirAttack:` prefix before lookup.

---

## Code layout

```
src/main/java/net/ercombat/
  sim/       pure Java 60 Hz simulation, no Minecraft imports
  server/    authoritative session, world adapter, packets, commands
  net/       payload records and the wire format
  registry/  the 25 weapon items
  mixin/     Player (attack/hurt), LivingEntity (travel/fall), packet handler
src/client/java/net/ercombat/client/
  ErCombatClient   keys, input snapshot, receivers, HUD hook
  ClientCombatState  latest state/log per entity
  CombatHud        bars, lock reticle, log
  anim/            procedural Animator + optional JsonClips
  mixin/           PlayerModel pose injection, vanilla attack suppression,
                   teleport interpolation
src/test/java/net/ercombat/sim/CombatTests.java   43 behaviour tests
tools/gen_items.py                  item/model/lang JSON + textures
tools/anim/er_to_minecraft.py       ERAN -> clips.json retarget
```

The simulation is deliberately free of Minecraft types so it can be tested
headlessly and reused: the server adapter is the only place that knows about
`ServerPlayer`, blocks and sounds.

---

## Fidelity limits

* **No extracted data ships here.** The action table in `ActionLibrary` is a
  regenerated estimate in the same shape and units (30 fps frames, per-action
  windup/active/recover, cancel windows, iframes, hit tables, root motion) as
  the sandbox's table, and `Const` marks what the original read from the game
  (`EXT`) versus what the sandbox estimated (`EST`). Values will not be
  bit-identical to the Rust project.
* **No proprietary animations or sounds.** See the animation section; the
  procedural clips are an approximation of the real thing, and without an IK
  pass a six-box model cannot reproduce finger placement, spine curvature or
  foot contact. Foot sliding is expected — the game moves the body from the
  simulation, not from the clip.
* The dummy is a vanilla husk driven by `DummySim` rather than a custom entity,
  so it uses the zombie rig: the client poses its windup, strike and recover
  from the simulation timer, but it has no bespoke model or weapon.
* Vanilla mobs take damage from the port's hits through the normal damage
  path; they do not use the player's combat simulation when attacking you,
  so their attacks arrive as ordinary Minecraft damage (and are ignored while
  the simulation is active, which is why only the sparring dummy can hurt you).

## Credits

Mechanics, tuning and the extracted-data approach are Funny-Bones'
[ELDEN-RING-Combat-Rewrite](https://github.com/Funny-Bones/ELDEN-RING-Combat-Rewrite)
(MIT). Elden Ring is a trademark of FromSoftware; this project ships no game
assets.
