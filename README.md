# Ultrakill: Gabriel - Forge 1.20.1 port (work in progress)

Port of the boss **Gabriel** from the Bedrock add-on `Ultrakill.mcaddon` to Minecraft Java 1.20.1 (Forge 47.x, GeckoLib 4.4.x).
Mod id `ukgabriel`. See `ARCHITECTURE.md` for how each original system maps to Java.

## Status in one paragraph
All assets are converted (13 geometries, 49 + 52 animations, 17 particle definitions, 55 sound events / 54 ogg files, textures, lang, loot table).
The boss logic is ported: Bedrock controllers/events/timelines run unchanged through a Java interpreter, 1,190 Bedrock commands are
translated to Java 1.20.1, and `ai.js`, `projectiles.js`, `damage.js`, `armor.js`, `target.js`, `anti_heal.js`, `difficulty.js`, `main.js` are ported.
**Eight headless test suites pass (tools/test). The Forge/GeckoLib code has never been compiled or run** - it was written without
access to those libraries, so expect to fix some API names on the first build.

## Build
1. Copy a Gradle wrapper (`gradlew`, `gradlew.bat`, `gradle/`) from a Forge 1.20.1 MDK (47.3.x) into this folder.
2. `./gradlew build` (JDK 17). Fix any compile errors (most likely GeckoLib/Forge method names; see ARCHITECTURE.md "Forge layer status").
3. `./gradlew runClient`, create a creative world with cheats, `/give @s ukgabriel:gabriel_spawn_egg`.

Headless tests (JDK 21, no Minecraft): compile `src/main/java/**/bedrock/**`, `ai/{GabrielTier,MovePicker,Movement,FlightMath}.java` and `tools/test/*.java`,
then run e.g. `java BrainTest src/main/resources/data/ukgabriel`.

## Commands
`/ukgabriel tier [harmless|lenient|standard|violent|brutal|mob_battle|default]` - difficulty tier (default follows world difficulty).
`/ukgabriel script skip_aoh_intro` - skip the transformation cutscene. The remaining `/ukgabriel event|particle|summon|sub|camera|hud|input|script`
commands are internal: the translated boss timelines call them.

## First in-game test plan (in this order)
1. **Loads**: log shows `loaded 17 Bedrock particle definitions`; no `Cannot load Gabriel AI data`; spawn egg exists.
2. **Spawn**: Gabriel appears with the Judge model and judge texture, a boss bar (name from the `minecraft:boss` component group), is immune ~7 s (intro), plays the intro animation.
3. **Animations**: he idles, then attacks (m1..m4); check animations change with attacks. Unknown-animation warnings in the log point at mapping bugs.
4. **Particles**: sword impacts show blast core/shell/bolts; `/ukgabriel particle uk:shockwave_cyan ~ ~1 ~` shows a shockwave. Check additive blending looks right.
5. **Projectiles**: spear beam / thrown sword / axe / greatsword fly, home, can be parried by hitting them (reflected back at Gabriel), and explode/fizzle.
6. **Stage changes** (use `/damage` or a strong weapon): stage 2 at 50% HP, transformation cutscene at 8% (camera, fade, HUD hidden, input locked,
   subtitles, music), apostate stage 4/5, defeat at 3%, loot drop (nether star, gold, diamonds).
7. **Edge cases**: leave the area mid-cutscene (HUD/camera/input must restore within ~1 s), relog during the fight, `/ukgabriel tier mob_battle`.

## Known deviations / gaps
- Chara buff hooks (`chara_buffs.js`) and `summon_rules.js` are not ported (they belong to other bosses' systems).
- Damage causes are approximated with Java damage types; boss music has no 1 s fade; animation/particle `locator`s are ignored.
- `spiral_swords` (ring entity) is registered but nothing in the add-on spawns it. The unused `combat` controller and the 23 legacy-model
  animations are included in the merged animation file but are never played, exactly as in the original client entity.
- Base BP behaviours `look_at_target` is not ported (no visible effect in the original model); hurt_by_target / nearest_attackable_target / look_at_player are.
- `gabriel:summon` / `skip_aoh_intro` scriptevents: only skip_aoh_intro is implemented (the other is unused by the add-on data).

## Difficulty config
`config/ukgabriel-common.toml` (created on first launch), section `[difficulty]`: `tier` (auto|harmless|lenient|standard|violent|brutal|mob_battle),
`paceMultiplier`, `projectileSpeedMultiplier`, `damageMultiplier`, `armorCapMultiplier`.
Precedence: `/ukgabriel tier <name>` (saved per world) > config `tier` > world difficulty. The multipliers apply on top of whichever tier is active.

`[damage]` section of the same file: `playerDamageMultiplier`, `mobDamageMultiplier`, `maxPlayerHit` (0 = no cap), `parryDamageMultiplier`.
They apply after the tier scaling to every hit from Gabriel's melee, projectiles and explosions (all go through `GabrielDamage.onHurt`).

## Camera item
`/give @s ukgabriel:gabriel_camera` (also in the Spawn Eggs creative tab): right-click to enter Camera Mode (port of `camera.js`: free camera at offset -7/+3/+2 from the nearest Gabriel, HUD hidden, movement locked, player invisible), right-click again to exit. Leaves automatically if Gabriel is gone, on dimension change or logout.

## Alternate skin
`textures/entity/gabriel/judge_alt.png` (red/gold recolor taken from `Gaybriel.bbmodel`, whose geometry is identical to the existing judge model). Enable with `[visual] alternateSkin = true` in `ukgabriel-common.toml`; it replaces the default judge skin only (not the phase 2 / enraged ones).
