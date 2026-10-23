# EU5 assimilation transparency mod: research notes

Game: Europa Universalis V (Jomini engine), current digest 1.3.11. This is not EU4: no `.mod` descriptor, no province scripting, different folder layout. Only EU5 sources are used below.

Source labels: **[wiki]** = eu5.paradoxwikis.com, fetched raw. **[digest]** = `Europa-Universalis-5-Modding-Co-op/modding-digests` (script_docs/dump_data_types output for 1.3.10-1.3.11, plus per-patch file lists). **[inferred]** = my reasoning, not verified in-game.

Not available to me: the game's own `in_game/` script and GUI files (the digests list changed file names and docs dumps, not file contents). The actual monthly assimilation maths appears to be engine code, see 2.3.

## 1. Confirmed facts

### 1.1 Assimilation and conversion inputs
- **[wiki Population]** A pop whose culture or religion differs from the country's primary one slowly changes. Rate depends on average Cleric literacy and drops 1% per 1% Control below 100% in the location.
- **[wiki Population]** Social classes are defined in `common/pop_types`. Clerics give +25% pop assimilation speed and +25% conversion speed at their max effect (10% of pops). Each pop type has `assimilation_conversion_factor` **[wiki Pop modding]**.
- **[wiki Culture]** Assimilation speed depends on "culture war power" (Cultural Tradition vs Cultural Influence, both decay 5%/month) and a per-country assimilation rate.
- **[wiki Culture]** Culture opinion changes speed. Table (opinion of us on them / of them on us): Kindred -30% / +60%, Positive -10% / +20%, Neutral 0 / 0, Negative +10% / -20%, Enemy +30% / -60%.
- **[wiki Culture]** Each culture has a status: Primary, Accepted, Tolerated, Discriminated. Accepted and tolerated cultures consume culture capacity; going over capacity gives penalties.
- **[wiki Defines, NPop/NCulture]** Assimilation weights (additive modifiers on the speed):

  | Define | Value |
  |---|---|
  | `ASSIMILATION_SHARED_CULTURE_GROUPS_MODIFIER` | 0.3 |
  | `ASSIMILATION_DOMINANT_ACTOR_CULTURE_MODIFIER` | 0.5 |
  | `ASSIMILATION_SAME_LANGUAGE_MODIFIER` | 0.5 |
  | `ASSIMILATION_SAME_LANGUAGE_FAMILY_MODIFIER` | 0.1 |
  | `ASSIMILATION_ACTOR_AND_TARGET_HAVE_MARKET_LANGUAGE_MODIFIER` | 0.4 |
  | `ASSIMILATION_ACTOR_HAS_MARKET_LANGUAGE_MODIFIER` | 0.2 |
  | `ASSIMILATION_TARGET_HAS_MARKET_LANGUAGE_MODIFIER` | -0.2 |
  | `ASSIMILATION_BURGHER_MARKET_LANGUAGE_MIN_MULTIPLIER` | 0.01 (burghers whose language is the market language assimilate at 1 - market access, floored here, instead of being blocked) |
  | `ASSIMILATION_PRIMARY_CULTURE_MODIFIER` | -1 |
  | `ASSIMILATION_ACCEPTED_CULTURE_MODIFIER` | -0.75 |
  | `ASSIMILATION_TOLERATED_CULTURE_MODIFIER` | -0.25 |
  | `ASSIMILATION_IS_MERGED_CULTURE_GROUP_MODIFIER` | 1 (owner has a Unified Culture Group containing the target) |
  | `ASSIMILATION_TARGET_CULTURE_OPINION_OF_OWNER_MODIFER` | 0.2 (opinion range -3..+3) |
  | `ASSIMILATION_OWNER_CULTURE_OPINION_OF_TARGET_MODIFER` | -0.1 (range -3..+3) |

- **[wiki Religion]** Tolerance defaults: +1 primary faith, -2.5 heretic (same religion group), -5 heathen (other group). Religions have separate "pop conversion speed" modifiers, and the modifier list has heretic- and heathen-specific conversion speeds.

### 1.2 Modifier hooks that can silently stop assimilation or conversion **[digest modifiers.log]**
- Speed: `local_/global_pop_assimilation_speed`, `..._speed_modifier`, same for `pop_conversion_speed`, plus `heretic_` and `heathen_pop_conversion_speed_modifier`.
- Hard blocks **per social class**, local and global: `{nobles,clergy,burghers,peasants,laborers,soldiers,slaves,tribesmen}_assimilation_blocked` and `..._conversion_blocked`.
- Country: `blocked_from_conversion`, `allow_slave_conversion`, `allow_cabinet_assimilate_area`.

### 1.3 Script-exposed data functions the UI can call **[digest data_types_uncategorized.txt]**
These are available to GUI scripts and to localization. Return types are listed, argument semantics are not documented.
- Location: `CanAssimilate` (bool), `CanConvert` (bool), `GetCannotAssimilateInfo` (string), `GetCannotConvertInfo` (string), `GetAssimilateInfo`, `GetAssimilateInfoForCulture(culture)`, `GetAssimilatePrimaryDescriptionForCulture(culture)`, `GetAssimilatingCultureInfo(culture)`, `GetMonthlyAssimilationChangeForCulture(culture)` (string), `GetMonthlyChangeBreakdownForCulture(culture)`, `IsAssimilatingToCulture(culture)`, and the religion equivalents (`GetConvertInfo`, `GetConvertInfoForReligion(religion)`, `GetConvertPrimaryDescriptionForReligion`, `GetConvertingReligionInfo`, `IsConvertingToReligion`, `GetConvert`).
- Pop: `CanPopAssimilate`, `CanPopAssimilateInfo`, `CanPopConvert`, `CanPopConvertInfo`.
- Other: `Country.GetAcceptedCulturesInfo`, `Culture.IsAccepted/IsTolerated/IsNonTolerated`.

### 1.4 Mod mechanics **[wiki Modding, Mod structure]**
- Local mods go in `Documents/Paradox Interactive/Europa Universalis V/mod/<name>/`. The folder mirrors the game's `game/` folder: top folders `in_game`, `main_menu`, `loading_screen`. A wrong path level fails silently.
- Required: `.metadata/metadata.json` with `name`, `id`, `version`, `supported_game_version` (e.g. `"1.3.*"`), optional `relationships` (dependencies) and `game_custom_data.replace_paths`. Thumbnail `.metadata/thumbnail.png` is under 1 MB.
- A file with the same path and name as vanilla replaces it entirely. In most `common/` subfolders, database entry modes avoid that: `INJECT:`, `REPLACE:`, `TRY_INJECT:`, `TRY_REPLACE:`, `INJECT_OR_CREATE:`, `REPLACE_OR_CREATE:`. `INJECT` does not work for scripted effects or triggers (it overwrites them). Defines and on_actions have their own rules (see wiki Defines and On actions pages).
- Localization files must end `_l_english.yml` and be UTF-8 with BOM.
- Debugging: launch option `-debug_mode` (hot reload, console, error.log button). Console commands `script_docs` and `dump_data_types` regenerate the exact documentation for your installed version (to `Documents/Paradox Interactive/Europa Universalis V/docs` and `logs/data_types`). Read `logs/error.log`.
- Tooling: VS Code with Paradox Highlight and CWTools; the Community Mod Framework; Discord "EU5 Mod Coop".
- Wiki warns that some logic is hardcoded (it names migration and battle casualties).

## 2. What this means for your Castile / Morocco problem

### 2.1 Candidate explanations, in the order I would test them **[inferred]**
1. **Per-class blocks and factors.** Assimilation is computed per pop class (per-class `assimilation_conversion_factor`, per-class `*_assimilation_blocked`). A tooltip total that sums all classes will not match the monthly result if some classes are blocked or cannot assimilate.
2. **Burghers and market language.** Burghers are throttled by market language (floor 0.01 of normal) so a location with many Maghrebi burghers can show a large expected figure that the burgher share never delivers.
3. **Culture status and opinion modifiers.** If Moroccan cultures are Accepted (-0.75) or Tolerated (-0.25) in Castile, or the opinion values are unfavourable, the real rate is much lower. The tooltip may display only some of these.
4. **Religion interaction.** Your observation (converting to Catholic first fixes it) suggests religion affects culture assimilation. The data shows a dedicated `CanPopAssimilate` / `CanPopAssimilateInfo` and `GetCannotAssimilateInfo`, so there is an engine-side eligibility check that lists reasons, and the demographics tooltip might not call it. Whether a non-primary religion blocks or slows culture assimilation is **not confirmed** by any source I found. This needs an in-game test.
5. **Rounding / minimum increments.** Small rates on small groups can produce zero or lumpy whole-pop moves versus a fractional "expected" number.
6. **Group merging.** The "Foreign Pop Types" mod description says small same-type groups of different culture or religion are merged into the larger one. This would move pops without being counted as assimilation.

### 2.2 What the mod can and cannot do
- **Can (low risk, no engine changes):** add or modify GUI and localization so the demographics tooltip also shows `Location.GetCannotAssimilateInfo`, `GetMonthlyAssimilationChangeForCulture`, `GetMonthlyChangeBreakdownForCulture`, per-class eligibility via `Pop.CanPopAssimilateInfo`, plus our own computed warnings from triggers (culture status, Control, religion status, class blocks). This is the core of the requested transparency.
- **Can (optional):** change the modifier values and defines the maths reads, via `INJECT:` and defines modding.
- **Cannot (likely):** change or fix the engine's tick itself if it is hardcoded. The wiki explicitly says some logic is hardcoded, and no assimilation script appears in the digests (they contain no script bodies for it). To confirm, grep the installed game's `in_game/common` and `in_game/gui` for the function names in 1.3.

### 2.3 Caveat
I found no vanilla script text for the monthly assimilation update. The existence of the data functions in 1.3 suggests the tick lives in the engine. Treat the explanations in 2.1 as hypotheses until tested against your game files and a save.

## 3. Plan

1. **Ground truth (needs your game install).** From `Europa Universalis V/game/` copy `in_game/gui` (demographics and location views), `in_game/common/pop_types`, `in_game/common/cabinet_actions/assimilate_area.txt`, `in_game/common/defines` and the matching localization. Grep for `GetAssimilateInfo`, `GetCannotAssimilateInfo`, `GetMonthlyAssimilationChangeForCulture`, `assimilation` and `conversion`.
2. **Reproduce.** In `-debug_mode`, take a Castile save with Moroccan Muslims. Record the tooltip numbers, then the pop deltas over several months, per class. Repeat after converting the religion. Check `error.log`.
3. **Build v0.1 (transparency only).** New mod `eu5_assimilation_transparency` with `.metadata/metadata.json`, new GUI additions and localization, using `INJECT`-style or new files rather than whole-file replacement where the GUI allows it, to limit conflicts with other mods such as Realistic Conversion and Assimilation.
4. **Warnings.** Show a flagged line for each blocker found in 2.1 for the viewed culture/location.
5. **Optional v0.2.** Behaviour changes, only if tests show a vanilla inconsistency worth fixing.

## 4. Open questions
- Game version and active mods?
- Can you provide the game files listed in plan step 1, or run `script_docs` / `dump_data_types` and share the output?
- Explanation only, or also fixes to the numbers?
