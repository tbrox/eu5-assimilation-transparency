# EU5 assimilation transparency mod: research notes

Status: **desk research only.** The game files were not available in this environment and WebFetch could not resolve hosts, so everything below comes from search-result summaries. Nothing here was verified against the game's script files. Items are marked **[sourced]**, **[hypothesis]** or **[verify]**.

This is Europa Universalis V (Jomini engine). EU4 techniques (`common/` at the root, `on_actions` province events, `province_id` scripting, `.mod` descriptors with `path=`) do not carry over. Use only EU5 sources.

## 1. What is known about vanilla assimilation

- **[sourced]** The wiki's Population page: a pop whose culture or religion differs from the country's primary one slowly changes to it. The rate depends on the average literacy of Clerics and is reduced by 1% for each 1% of Control below 100% in the location. ([wiki](https://eu5.paradoxwikis.com/Population))
- **[sourced]** Pop types (`in_game/common/pop_types/`) have an `assimilation_conversion_factor`. ([Pop modding](https://eu5.paradoxwikis.com/Pop_modding))
- **[sourced]** The Defines page lists weights such as `ASSIMILATION_PRIMARY_CULTURE_MODIFIER` = -1 (primary culture is slowest to assimilate), a shared-culture-group modifier of 0.3 and a dominant-actor-culture modifier of 0.5. ([Defines](https://eu5.paradoxwikis.com/Defines))
- **[sourced, old]** A developer reply in [this thread](https://forum.paradoxplaza.com/forum/threads/how-does-culture-conversion-work-in-eu5.1743848/) says the cabinet action converts pops at a base rate (about 10 pops/month at the time), multiplied by Control, societal-value bonuses, and so on. It predates current patches.
- **[sourced]** Another forum post says religious conversion is faster than culture assimilation but is capped while the religion is not the culture's primary one. The same post says the cabinet action only affects land-bordering pops. ([thread](https://admin-forum.paradoxplaza.com/forum/threads/assimilation-particularly-for-religion-happens-too-fast.1850699))
- **[sourced]** The "Foreign Pop Types" mod notes the game merges the smaller group into the larger one when a location has too many pops of the same type with different cultures or religions. This may be relevant to why counts shift.

## 2. Hypotheses for the mismatch you see (all unverified)

Your observation is that converting to Catholic first makes assimilation work as the tooltip predicts. That points to religion gating or modifying culture assimilation. Candidates:

1. The tooltip computes the rate for the culture alone, but the real tick applies an extra religion-acceptance factor or cap for non-accepted religions (Muslim pops under a Catholic ruler).
2. Pops are grouped by (type, culture, religion). The tooltip aggregates by culture, while the tick works per group, so the target group or its size differs.
3. Rounding or fractional pops: a small per-group rate produces zero or lumpy whole-pop moves, so monthly results don't match the tooltip's expected number.
4. The tooltip shows the potential rate before Control, a minimum-size threshold, or the "primary culture below X% of location" penalty.
5. Group merging (see above) moves pops that were not counted as assimilation.

Each can be confirmed or ruled out by reading the vanilla script, see section 4.

## 3. Mod structure (EU5) **[verify each against a current mod]**

- Mod folder under `Documents/Paradox Interactive/Europa Universalis V/mod/<name>/`.
- `.metadata/metadata.json` holds id, name, version, supported game version.
- Game content lives under `in_game/` (e.g. `in_game/common/...`, `in_game/events/`, `in_game/gui/`), and localization under `localization/` in a language subfolder, files named `*_l_english.yml`.
- Files with the same relative path replace vanilla ones. Prefer new, uniquely named files over full overrides, which is the main compatibility risk with the Realistic Conversion and Assimilation mod and with patches that add defines.
- Existing EU5 mods touching this area, useful as references for which files and hooks they use: [Realistic Conversion and Assimilation](https://steamcommunity.com/sharedfiles/filedetails/?id=3683834085), Conversion and Assimilation Overhaul, Tolerated/Accepted Culture Assimilation (its author says some per-location behavior is hardcoded). The Tinto Talks #85 modding dev diary and the `eu5miner` Python library may help with parsing.

## 4. Plan

1. **Get ground truth.** Copy the game's `in_game/` folder (or use `eu5miner`) and grep for `assimilation`, `conversion`, `culture_conversion`, and the tooltip localization keys. Identify the script_values or defines that drive the monthly tick and where the tooltip text is generated. If the tick is hardcoded in the engine, the mod cannot change it, only explain it.
2. **Reproduce.** Castile vs Moroccan Muslims, same location, record tooltip numbers against save-to-save pop deltas, with and without the Catholic conversion.
3. **Transparency layer (low risk first):**
   - Add localization/tooltip overrides that show each factor and its value.
   - Add a warning surface for blockers: religion not accepted, Control below 100%, primary culture below the size threshold, no land border (for the cabinet action), culture not accepted, pop-merging risk.
4. **Behavior changes (only if needed):** if the data shows a vanilla bug, a separate optional fix file.
5. Test on a new game and a loaded save, check the error log for script errors, and publish with the supported game version stated.

## 5. Open questions for you

- Game version/patch and any mods currently active?
- Do you want explanation only, or also fixes to the numbers?
- Can you share the game install path contents (or the relevant `in_game/common` files) so step 1 can be done accurately?
