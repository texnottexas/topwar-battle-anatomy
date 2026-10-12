# Anatomy of a Battle Report
**Last updated:** 2026-10-12  
**Based on reports:** 4605240496314802177 (PvP, fightType=114), 4605144673410904064 (Wasteland Guardian, fightType=126), plus a 23-report study (March to October 2026, 46 player sides, mostly Time Clash)  
**Built from:** battle-report.html viewer, buff mapping, attributes spec, CDN/asset references, direct JSON analysis

**Confidence tags:** **Confirmed** = seen in the data and/or checked against a game table. **Likely** = consistent with all evidence, mechanism not proven. **Guess** = a hypothesis only. Untagged statements are Confirmed.

## Changelog

**2026-10: corrections and additions from a 23-report study**

- Exclusive and supporting skill levels ARE in the report (`nonActiveSkills[1]` and `[2..5]`); only the native passive is always 0. Awakening fields (`awakenLevel`, `awakenSkillMap`, `fullAwaken`) documented.
- Real layouts replace the old guesses for `enigmas`, `elementLevels`, `effectSkins`, `formationV2`, `armyMastery` and titan gear (`equipId` formula, rune star inside `templateId`, per-stat `enhanceValue`).
- `CTCs` are Loot Quest collectibles (not HT chips); `boxMaxIds` is not a max-level flag; HT chips live in `mechas[].chips[]`; `armyEquips` is the 3x7 Unit Equipment grid.
- `players[]` is one entry per march (a defender can field several); `heroEquips` does not exist at player level.
- Buff sources renamed and mapped: §13 now uses the game's own source-label table. Base Effect fully explained (s=25 skin ownership + equipped skin + exhibitions, s=75 castle effects, s=76 march skins); s=114-117 = Hero Awakening, s=139 = Hero Skin, s=3 = Hero Talent, s=41 = Formation Talent, s=64 = BUFF Item, s=104 = Hero Pre-battle Skill; s=134 is present in ordinary Time Clash fights.
- `attSpecialBuff` / `defSpecialBuff` decoded (Ultra Valhalla troop line); `units[].armyId` families decoded; mecha chip `rndAttrs` decoded.
- `process[]` is a sample with no crit, dodge or heal events (27 event types now listed); the old `maxPower / CP = 5.4M` rule is false.
- New: how to compute how close a fight was (section 8.6), command vs skill slots, march size approximation, and a new section 19, "Reading a report for advice".
- Sections 17 and 18 rewritten: only what is still unknown, and the corrected gotchas.

---

## Table of Contents

1. [CDN Access](#1-cdn-access)
2. [Top-Level JSON Structure](#2-top-level-json-structure)
3. [battle Root Fields](#3-battle-root-fields)
4. [attacker / defender (March Level)](#4-attacker--defender-march-level)
5. [players[0] — Full Player Detail](#5-players0--full-player-detail)
   - 5.1 [effectBuffs — Aggregated Buff Pool](#51-effectbuffs--aggregated-buff-pool)
   - 5.2 [traceEffectBuffs — Per-Source Breakdown](#52-traceeffectbuffs--per-source-breakdown)
   - 5.3 [heroList — Heroes in March](#53-herolist--heroes-in-march)
   - 5.4 [heroEquips — Titan Gear](#54-heroequips--titan-gear)
   - 5.5 [effectDecorations — Decorations](#55-effectdecorations--decorations)
   - 5.6 [effectSkins — Base Skins](#56-effectskins--base-skins)
   - 5.7 [enigmas — Enigma Beasts](#57-enigmas--enigma-beasts)
   - 5.8 [formationV2 — Formation](#58-formationv2--formation)
   - 5.9 [armyEquips / newArmyEquips — Unit Equipment](#59-armyequips--newarmyequips--unit-equipment)
   - 5.10 [CTCs — Loot Quest Collectibles](#510-ctcs--loot-quest-collectibles)
   - 5.11 [elementLevels — Element Levels](#511-elementlevels--element-levels)
   - 5.12 [armyMastery — Army Mastery](#512-armymastery--army-mastery)
   - 5.13 [allStandings — Battle Outcome](#513-allstandings--battle-outcome)
   - 5.14 [armyUuids — Individual Troop UUIDs](#514-armyuuids--individual-troop-uuids)
   - 5.15 [buffs — Raw Buffs](#515-buffs--raw-buffs)
   - 5.16 [Other players[] fields](#516-other-players-fields)
6. [units[] — Unit Formation Slots](#6-units--unit-formation-slots)
7. [mechas / mechas2 — Mecha Data](#7-mechas--mechas2--mecha-data)
8. [fightMarches[0] — Battle Outcome Data](#8-fightmarches0--battle-outcome-data)
   - 8.1 [Combat Power (CP)](#81-combat-power-cp)
   - 8.2 [Army Composition (attPutArmyV2)](#82-army-composition-attputtarmyv2)
   - 8.3 [Skill Damage (attSkillHurts)](#83-skill-damage-attskillhurts)
   - 8.4 [Traps](#84-traps)
   - 8.5 [Special Buffs](#85-special-buffs)
   - 8.6 [How Close Was the Fight](#86-how-close-was-the-fight)
9. [process[] — Round-by-Round Events](#9-process--round-by-round-events)
10. [etReward — Battle Rewards](#10-etreward--battle-rewards)
11. [Combat Power: How It's Calculated](#11-combat-power-how-its-calculated)
12. [Wasteland / Territory Buffs and CP](#12-wasteland--territory-buffs-and-cp)
13. [Buff Source Reference (traceEffectBuffs s= values)](#13-buff-source-reference-traceeffectbuffs-s-values)
14. [fightType Reference](#14-fighttype-reference)
15. [Assets: Icons and Portraits](#15-assets-icons-and-portraits)
16. [What We've Built (Pages + Skills)](#16-what-weve-built-pages--skills)
17. [Unmapped / Unknown Areas](#17-unmapped--unknown-areas)
18. [Key Gotchas](#18-key-gotchas)
19. [Reading a Report for Advice](#19-reading-a-report-for-advice)

---

## 1. CDN Access

```
URL: https://fight-report-va.oss-accelerate.aliyuncs.com/prod/{first4}/{reportId}.json
Example: reportId=4605240496314802177 → .../prod/4605/4605240496314802177.json
```

- Public, no authentication required
- Only PvP battle reports are available. Dark Force / NPC guardian attacks are **not** on the CDN (404)
- Size: 200-650KB depending on player detail and process events
- `first4` = first 4 digits of the reportId string

---

## 2. Top-Level JSON Structure

```
{
  "reportId": "4605240496314802177",
  "logtime": 1775319323,           // Unix timestamp of battle
  "battle": { ... },               // Full battle data (see below)
  "newFight": ...,                 // present in 22 of 23 studied reports; purpose unknown
  "pointStr": { "x": 27, "y": 465, "id": 0, "pointType": 1 },  // map tile of the fight (some reports)
  "srcPointStr": { "x": 30, "y": 480, "id": 0 },               // origin tile of the attacking march (base-attack report only)
  "pveId": 4004                    // NPC encounter id (PvE report only)
}
```

- `pointStr`, `srcPointStr`, `pveId`: **Likely** meanings (map tile of the fight, origin tile of the attacker, NPC encounter). Each appears in one or two reports only.

---

## 3. battle Root Fields

```json
{
  "result": 1,
  "ver": 1,
  "stage": 1,
  "fightType": 114,
  "fightType2": "(32-byte base64 string)",
  "serverVersion": "1.680.0",
  "reportComparison": 1,
  "extraFlag": { "putOnArmyV2": 1 },
  "attacker": {},
  "defender": {},
  "fightMarches": [],
  "process": [],
  "etReward": []
}
```

| Field | Meaning |
|-------|---------|
| `result` | 1 = attacker wins, 2 = defender wins (holds in all 23 studied reports) |
| `ver`, `stage`, `reportComparison`, `extraFlag.putOnArmyV2` | Identical in all 23 studied reports (1 / 1 / 1 / 1). `putOnArmyV2=1` means `attPutArmyV2` is present |
| `fightType` | Battle type (see §14) |
| `fightType2` | 32-byte base64 value, different in every report. **Guess:** a hash or signature |
| `serverVersion` | Game server build that produced the report ("1.657.0" to "1.680.0"); first seen 2026-06-01 |
| `fightMarches` | One entry per march (see §8). Several entries when a defender fields several marches |
| `etReward` | Win loot; present in only 1 of the 23 studied reports (see §10) |

---

## 4. attacker / defender (March Level)

These are the march-level containers. Most detailed data is inside `players[]`.

```
{
  "fightTeamId": 2,          // Team number
  "marchId": 2,              // March ID (matches fightMarches attMarchId/defMarchId)
  "uid": XXXXXXX 272,        // Player UID (masked here)
  "career": 72001,           // Class ID
  "stamp": 1008,             // Class progress step
  "alTag": "",               // Alliance tag
  "headFrameId": 0,          // Avatar frame ID
  "isolated": 0,
  "attackPointNum": 9,       // Number of formation attack points used
  "hasEffectMecha": 1,       // Whether a mecha is deployed (1=yes)
  "armyUuids": [ ... ],      // All individual troop UUIDs (404 items = 404 troops)
  "effectBuffs": [ ... ],    // Mirrors players[0].effectBuffs
  "traceEffectBuffs": [ ... ],// Mirrors players[0].traceEffectBuffs
  "elementLevels": [ ... ],  // Mirrors players[0].elementLevels
  "allStandings": { ... },   // Mirrors players[0].allStandings
  "buffs": [ ... ],          // Raw buffs (small list; see §5.15)
  "units": [ ... ],          // Formation unit slots (see §6)
  "mechas": [ ... ],         // Mecha data (older format; see §7)
  "mechas2": { ... },        // Mecha data (newer keyed format; see §7)
  "players": [ ... ]         // Detailed data, ONE ENTRY PER MARCH
}
```

> **Note:** `effectBuffs`, `traceEffectBuffs`, `elementLevels`, `allStandings`, `armyUuids` appear at BOTH the march level and inside `players[0]`. They are the same data, and the march-level copy mirrors `players[0]` only. Use `players[]` as the canonical source.

- **Team numbers are not fixed.** The attacker is team 2 in 18 of 23 reports and team 1 in 5. Use `fightMarches[].attFightTeamId`, never assume.
- **`players[]` is one entry per march, not per person.** A defender in a base-attack report fielded five marches: five `players[]` entries with the same nickname, different `fightTeamId` (2..6), different heroes, `allStandings` and `effectBuffs`, and five `fightMarches` entries. In a 1v1 duel it has exactly one entry.
- `career`: 72001 = Combat Elite, 72002 = Mechanical Master (matched against known rosters in 7 of 7 cases). `stamp` = 1xxx for 72001 and 2xxx for 72002, probably a class progress step (**Likely**; cap unknown).
- `isolated` and `headFrameId` were 0 on every side studied.

---

## 5. players[0] — Full Player Detail

The richest data source. One entry per march (see §4).

```
{
  "uid": XXXXXXX 272,
  "career": 72001,
  "playerInfo": "{...JSON string...}",  // Name, avatar URLs, nationalflag, gender
  "stamp": 1008,
  "fightTeamId": 2,
  "marchId": 2,
  "isolated": 0,
  "attackPointNum": 9,
  "hasEffectMecha": 1,
  "avatarId": 105,            // October reports only; meaning unknown
  "effectBuffs": [ ... ],
  "traceEffectBuffs": [ ... ],
  "heroList": [ ... ],        // gear is inside heroList[].heroEquips; there is NO players[].heroEquips key
  "effectDecorations": { ... },
  "effectSkins": { ... },
  "enigmas": { ... },
  "formationV2": { ... },     // empty object on some sides (NPCs, rally members)
  "armyEquips": [ ... ],
  "newArmyEquips": { ... },
  "CTCs": { ... },
  "elementLevels": [ ... ],
  "armyMastery": { ... },
  "allStandings": { ... },
  "armyUuids": [ ... ],
  "extraArmy": { ... },
  "buffs": [ ... ],
  "traps": [ ... ]
}
```

### playerInfo (JSON string, not object)
```json
{
  "username": "PlayerName",
  "nickname": "",
  "nationalflag": 233,
  "gender": 0,
  "usergender": 1,
  "headimgurl": null,
  "headimgurl_custom": "https://knight-cdn.akamaized.net/headimg/{hash}.jpg",
  "avatarurl": null,
  "moreHeadImgs": [ "https://knight-cdn.akamaized.net/..." ],
  "cn": 0
}
```

- `nationalflag`: integer that maps to an ISO country code
- `headimgurl`: in-game avatar URL (may be null)
- `cn`: always 0; suspected Chinese server flag (**Guess**)

---

### 5.1 effectBuffs — Aggregated Buff Pool

Array of all buffs active during the battle, after grouping/redirect logic.

```
{ "id": 930001, "val": 256300 }          // Standard buff: id + val
{ "bufid": 921006, "value": 7000, "specialType": 1, "data": ["10112"] }  // Special buff
```

- `val` / `value`: raw value where **100 = 1%** (so 256300 = 2563% Army ATK)
- Total across all effectBuffs in observed reports: ~2.9-3.1M, but these are different stat categories, not one pool
- `specialType=1` with `data` array: conditional/triggered buff referencing specific unit UUIDs
- Some IDs are "redirect parents": the game groups children under them (e.g., 930000 groups 930001+930002+930003; 9301160 = All units ATK adds its full value to Army, Navy and Air Force ATK)
- **Buff IDs the game's grouping logic silently skips** (not in the EFFECTBUFF table): 520130-133, 520141-143, 932001-043, 933007-008, 934001, 934004, 936007-013, 950011

### 5.2 traceEffectBuffs — Per-Source Breakdown

```json
{ "id": 930001, "val": 48700, "s": 25 }
```

- Same buff IDs as effectBuffs, but split by source (`s` field)
- `s` = `type_small` source category (see §13 for the map)
- **Confirmed:** summed per buff id, `traceEffectBuffs` equals `effectBuffs` exactly (0 mismatches over 8,731 side/id pairs, 45 sides). The trace is a faithful split of the effective pool
- Total entries: 238-363 per player (varies by source diversity)
- **Key use:** drill into how much of the ATK comes from decorations vs titan gear vs enigma beasts, and compare two accounts source by source (see §19)

---

### 5.3 heroList — Heroes in March

```
{
  "id": 332,           // heroId
  "level": 120,        // Hero level (max 120)
  "star": 5,           // Star rating (max 5)
  "war": 50234,        // Hero ATK stat
  "def": 21481,        // Hero DEF stat
  "command": 2500,     // Command (see "Command vs skill slots")
  "skinId": 0,         // Hero skin (nonzero on a few heroes)
  "nonActiveSkills": [
    { "skillId": 10332, "level": 0 },   // [0] native/inherent passive (skillId = 10000+heroId), always level 0
    { "skillId": 20743, "level": 7 },   // [1] EXCLUSIVE skill, real level
    { "skillId": 21119, "level": 8 },   // [2] supporting skill 1
    { "skillId": 21120, "level": 7 },   // [3] supporting skill 2
    { "skillId": 21116, "level": 7 },   // [4] supporting skill 3
    { "skillId": 20116, "level": 7 }    // [5] supporting skill 4
  ],
  "awakenLevel": 35,
  "awakenExp": 20,
  "awakenSkillMap": { "10021": { "level": 10, "exp": 0 } },
  "fullAwaken": false,
  "heroEquips": [ ... ],  // Titan Gear (see §5.4)
  "talents": [
    { "talentId": 8, "ttsr": 10000 }
  ]
}
```

**Skill layout (Confirmed; reverses the April note that skill levels were never sent):**

| Index | Content | Level meaning |
|-------|---------|---------------|
| `[0]` | native passive (`10000 + heroId`) | always 0, ignore |
| `[1]` | exclusive skill | **the real exclusive level** (3-9 seen, cap 10; present on 134 of 135 heroes studied) |
| `[2..5]` | up to four supporting skills (ids 201xx = quality 1, 211xx = quality 2) | skill level 3-10 |
| `{skillId:0, level:0}` | an empty slot | |

- Find the exclusive skill by `skillId` against the hero's known exclusive id, or take `[1]`.
- Supporting skill families: March Size (20116/21116), Army/Navy/Air Force Damage Boost, Army/Navy/Air Force Protection, Crit Damage, Attack, HP, Crit, Dodge, Hit and others. Strong marches usually run Damage Boost + Protection for the march's branch + two March Size.
- An entry missing from the end of the list is either a locked slot or an unfilled one; the report cannot tell which (open question).

**Command vs skill slots (Likely):** `command` is 2500 when four supporting slots are shown and 2000 when only three are (146 of 147 heroes). That is -500 command per hero, about -19 troops over a three-hero march (see "March size" below).

**Awakening (Confirmed):**
- `awakenLevel` 0-40 (the Hyper Awakening meter); skills 2/3/4 unlock at 10/20/30
- `awakenSkillMap {awakeningSkillId: {level, exp}}`: the four awakening skills, level 1-10 each; only unlocked skills are listed
- `fullAwaken`: true when level 40 and all four skills at 10 (**Likely** rule)
- `awakenExp`: progress inside the current level (**Likely**)
- `starExp`, `changeId`, `dressId` were 0 on all 135 heroes studied; `avatarId` appears in October reports only

**Talents:** `talents[]` holds up to 4 of 14 talent types (1/2 Army ATK/HP, 3/4 Navy, 5/6 Air Force, 7/8 All Forces, 9-14 gathering and march speed). `ttsr` 10000 = fully learned; partial values (9, 22, 1477, 3195) show up on weak accounts, and an NPC-like side can have `talents: []`. A talent set aimed at the wrong branch is detectable.

**March size (Likely):** `allStandings.total` is the march size at battle start. It is roughly the sum of buff 960012 (March Size) plus `sum(command) / 80`; within 5 troops on 17 of 37 sides, within 10 on 25. Per-slot troop counts do not follow hero `command` alone.

---

### 5.4 heroEquips — Titan Gear

6 slots per hero (inside `heroList[].heroEquips`; the key does not exist at player level).

```
{
  "equipId": 100104,         // 100000 + slot*100 + quality
  "level": 20,               // Enhancement level (max 20)
  "heroId": 332,
  "landScore": 27000,        // CP contribution for Army branch
  "navyScore": 27000,        // CP contribution for Navy branch
  "airScore": 43632,         // CP contribution for Air Force branch
  "enhanceValue": { "930100": 2158 },
  "infos": [
    { "type": 1, "templateId": 9,      "buffValue": 330 },
    { "type": 1, "templateId": 8,      "buffValue": 420, "enhanceValue": 184, "enhanceShow": 2 },
    { "type": 2, "templateId": 102021, "buffValue": 0 }
  ]
}
```

- **`equipId` format (Confirmed):** `100000 + slot*100 + quality`, so 100104 = slot 1, quality 4 (gold). Slot = `floor(equipId/100) % 10`, quality = `equipId % 100`. (The earlier `slot*100000 + quality` formula was wrong.)
- `type=1`: random stat. At most 3 per piece. `templateId` maps to a stat template (the armory table holds 540; all 107 ids seen resolve)
- Roll quality: `buffValue / template.maxRaw * 100`. Template maxima: DMG +/- 600 (6%), ATK/HP 3000 (30%), DEF 300 (3%). No roll exceeded its maximum in 2,300+ rolls. Judge each stat on its own; the game's refinement gate is about 70% **per stat**, never an average
- `type=2`: rune. `templateId` is 6 digits = `base*10 + star`, so **the rune star IS in the report** (the April note that it was not was wrong). Rune `buffValue` is always 0. No `type=2` entry means an empty rune slot. Star caps: 6 for slot-locked and tactical runes, 2 for applier/amplifier runes
- `enhanceValue` per stat (`infos[].enhanceValue` + `enhanceShow`): an extra bonus; active when `enhanceShow === 2`; add it to `buffValue`. The ratio to `buffValue` varies 0.1 to 1.2 by stat. Present on about 19% of rolls. **Likely** tied to Enigma field slot levels rather than anything trained on the gear itself
- Gear-level `enhanceValue` keys seen: 930000 and 930100 only
- `landScore/navyScore/airScore` are per-slot CP by branch (NOT cumulative)
- `rateEquipId` is always 0 (meaning unknown)
- Level cap 20 (767 of 770 pieces in the study were quality 4; almost all at level 20)

---

### 5.5 effectDecorations — Decorations

```
{
  "ids": [ 52995, 47782, ... ],          // Every PLACED piece (duplicates allowed)
  "sublimes": [ { "id": 50178, "buffs": [{ "buffId": 930100, "value": 0 }] } ],
  "sublimeExps": [ { "val": 200, "id": 980204 } ],
  "suitLevelBuff": { "1201": 1, ... },    // Suit set -> active tier
  "suit": [ 1216, 1217, ... ],            // Completed suit set IDs (up to 23)
  "boxMaxIds": [ 4632002, ... ]           // One id per owned decoration group; NOT a max flag
}
```

- `ids`: one entry per placed piece; duplicates are separate pieces (a single group can appear 80-100 times). A full account lists 770-810 pieces over about 580 distinct groups. Each id resolves to a group and a level via the decoration table; the per-group cap is `ml` (15 for most groups, 2 for suit pieces)
- `suit` / `suitLevelBuff`: 23 suits exist; the active tier per suit was 1-3 in the data (the suit table lists a max level of 2, so the tier meaning is unsettled)
- **`boxMaxIds` is NOT a max-level list (Confirmed correction).** It holds one id per owned group, the highest-level piece in storage. Only about 38% of those ids are at the group's cap, so "Max" in the name is misleading. Use `ids` plus the group cap to judge levels
- `sublimes[].buffs[].value` is the small "Advanced Upgrade" bonus only (often 0), NOT the decoration's base buff. Base buffs come from a decoration table lookup by group id
- Trace source: s=7 (Decoration), the largest permanent source after Enigma Beasts and base skins

---

### 5.6 effectSkins — Base Skins

All values are arrays of ids (Confirmed shape; the April `skinId1/citySkill1` object was wrong):

```json
{
  "effectIds": [ 1712000, ... ],
  "useIds": [ 1712000 ],
  "ownCastleEffects": [ ... ],
  "useCastleEffects": [ ... ],
  "ownMarchSkins": [ ... ],
  "useMarchSkins": [ ... ],
  "ownHaloSkins": [ ],
  "useHaloSkins": [ ],
  "ownHeadframes": [ ],
  "exhibitions": [ ],
  "ownMoveCityEffects": [ ],
  "useMoveCityEffects": [ ],
  "magicIds": [ ],
  "useInfos": [ ]
}
```

| Field | Meaning |
|-------|---------|
| `useIds` | Equipped base skin |
| `effectIds` | The player's **unexpired skins that have an ownership bonus, highest level per skin group only**. Not "all skins owned": on one account, 230 owned gave 142 credited (79 lower levels, 6 with no bonus, 3 expired) |
| `ownCastleEffects` / `useCastleEffects` | Castle effects owned / worn (source s=75) |
| `ownMarchSkins` / `useMarchSkins` | March skins (source s=76) |
| `ownHaloSkins` / `useHaloSkins` | Halos |
| `ownHeadframes`, `exhibitions` | Cosmetics; `exhibitions` ids add their buff to s=25 (game table `skin_exhibition`) |

**Where the bonuses show up (Confirmed unless tagged):**
- **s=25 "Base Effect"** = `own_buff` of each skin in `effectIds` + `equip_buff` of the skin in `useIds` + the buff of each `exhibitions` id. Exact on every current report (older reports differ only because skin values were rebalanced since). Second only to Enigma Beasts as a separator of strong and weak accounts
- **s=75** = castle effects (`ownCastleEffects` own_buff + `useCastleEffects` equip_buff); exact on all 46 sides
- **s=76** = march skins (`ownMarchSkins` / `useMarchSkins`, game table `march_skin`); exact on all 46 sides
- s=4 is CPNT, not skins, and is near-constant on a given server
- City skill buffs (9990xxx ids) sit under **s=18** ("Special Effect Upgrade")
- Cosmetics have no "max"; only ownership counts
- **Known limitation:** Base Skin Skill sub-breakdown for opponents may show 0% in the viewer (separate code path)

---

### 5.7 enigmas — Enigma Beasts

```
{
  "beastDatas": [
    {
      "id": 1, "cfg": 5011, "star": 5, "level": 100, "potential": 15996,
      "power": 180120, "mainBuff": { ... }, "baseBuff": [ ... ]
    }
  ],
  "fields": [
    {
      "cfg": 4, "active": true, "conditions": [19, 20, 21],
      "slots": [
        { "id": 19, "beastId": "7", "level": 12, "potential": 16000,
          "buffs": [ { "id": 520001, "val": 4608 } ] }
      ]
    }
  ]
}
```

(Confirmed shape. The April `beastDatas[{beastId, element, fieldType, beastSkills, attrs}]` and `fieldAttrs` do not exist.)

- `beastDatas[].cfg` = `enigma_beast` table id (quality, faction, type, max level, max star). Level cap 100 (quality 5), 80 (quality 4), 60 (quality 3). Star cap 5. `potential` cap 16000 (range of rolls 6558-15996 seen). Element is decoded from `cfg`, not a field
- `fields[]`: five fields, `cfg` 1-5 = CPNT Mastery, Formation, Equip, Defense, Offense (holes per field 5 / 7 / 6 / 9 / 9)
  - **A field with no beasts is left out of the array**, whether it is locked or not
  - Unlock rule (game table `enigma_field.unlock_condition`): Defense opens at **40** and Offense at **150** total beast field slot levels (the sum of `slots[].level`). Confirmed: it separates every player in a 23-report study. Fields 1-3 have no level rule visible in a report
  - `active`: field unlocked (false on 5 sides studied)
  - `conditions[]`: ids of the field-bonus conditions that are satisfied (field 1: 1-5, field 2: 6-12, field 3: 13-18, field 4: 19-27, field 5: 28-36)
  - `slots[].id` = hole (platform) id 1-36; `slots[].level` = platform enhancement level, **cap 20 for fields 1-3 and 40 for fields 4-5** (cost is paid in field repeaters, 5-80 per early level); `beastId` "0" = empty hole; a filled hole with `buffs: []` means its condition is not met
- Trace sources: s=106 (beast skill buffs) and s=107 (field attributes), both shown as "Enigma Beast". Buff IDs in the 520xxx range
- Beast power values use 16000 as the universal max potential denominator
- Enigma Beasts are the biggest single separator between strong and weak accounts (see §19)

---

### 5.8 formationV2 — Formation

```
{
  "id": 1002,                 // 1001 Shark, 1002 Scorpion, 1003 Eagle
  "lv": 50,                   // Formation level (cap 50)
  "quality": 9,               // Quality tier 0-10
  "masterys": { "1": 4, "2": 3, "3": 2, "11": 1 },   // slot position -> tier 0-5
  "sciences": { "1": 15, "2": 15, "3": 30, "4": 47, "5": 82 },
  "formationLvUp": 0,
  "formationMasteryUp": 1,
  "formationQualityUp": 1
}
```

(Confirmed shape; the April `slots/activeSkills/passiveSkills` description was wrong.)

- Science caps: 15 / 15 / 30 / 60 / 120 for sciences 1-5
- `masterys` positions: 1, 2, 3, 11, 12, 13, 21, 22, 23 (the 3x3 grid). The extra positions (11, 13, 21, 23) unlock as science 5 reaches level 20/40/60/80
- `formationLvUp/MasteryUp/QualityUp`: extra levels from buffs (0 or 1); effective tier = `quality + formationQualityUp` (**Likely**)
- The object is empty on some sides (NPCs, rally members) and absent on some older reports
- Trace sources: s=32 and s=41 (both "Formations"); ids 1000021 (HP) / 1000022 (ATK)
- Scorpion formation skill 941215 appears as Counterblow (buff ID 100008) in skill damage

---

### 5.9 armyEquips / newArmyEquips — Unit Equipment

**`armyEquips[]`** is the 3x7 **Unit Equipment** grid (21 entries; 3 branches x 7 slots). It is not an HT chip list:
```
{ "itemId": 403000, "pos": 1, "armyType": 301 }
// armyType: 101=Army, 201=Navy, 301=Air Force
// itemId 0 = empty slot; level = itemId minus the item group's base id
// slots 0-1: type 21 items (cap 100); slots 2-6: type 20 items (cap 10)
```
- Source in the trace: s=26 "Unit Equipment" (ids 930001-3, 930101-3). Only the fielded branch matters for a given march

**`newArmyEquips.equips[]`**:
```
{ "eid": 910039, "pos": 30101, "suit": 9, "id": 3559869806198284297 }
// pos = {branch 1/2/3}{slot 01-05}: 10101..10105 Army, 20101.. Navy, 30101.. Air Force
// eid: equipment template id (910xxx range)
// suit: per-piece progress number 0-12 (Likely; not a suit-set index)
```
- 5 slots per branch; slots 1-4 have a fixed `eid` per branch and slot. A piece at `suit 0` carries a different `eid`, so the `eid` changes on the first upgrade
- The `eid` table is not resolved (see §17)

**HT chips** are NOT here; they live in `mechas[].chips[]` (see §7).

---

### 5.10 CTCs — Loot Quest Collectibles

```
{
  "themes": [1, 2, 3],
  "collects": [ { "id": 1, "lv": 2 }, { "id": 12, "lv": 5 }, ... ]
}
```

- **Confirmed correction:** these are **Loot Quest collectibles** (40 items in 4 themes; 38 distinct ids seen), not HT chips
- `collects[].lv`: 1-5 (cap 5)
- `themes`: the themes that are **completed** (all items owned); this matched in 196 of 196 checks. Each completed theme gives a flat All ATK or All HP bonus of roughly +10% to +15%
- Sum of levels / 190 is a usable completeness score. Nobody in the study had theme 4; most were exactly the same two items short

---

### 5.11 elementLevels — Element Levels

```json
[ { "armyType": 101, "elementType": 1, "level": 142 }, { "armyType": 201, "elementType": 1, "level": 142 }, { "armyType": 301, "elementType": 1, "level": 142 } ]
```

- Three entries, one per army type. The three `level` values are equal on 44 of 46 sides
- Range seen: 102-278. **No cap table located**; the effect is unknown (open)

---

### 5.12 armyMastery — Army Mastery

```json
{ "101": 121, "201": 219, "301": 321 }
```

- Keys: 101=Army, 201=Navy, 301=Air Force
- **Level = value minus the key** (121 means level 20). Cap 20. Observed 15-20
- Contributes to `s=17` "Mastery" in traceEffectBuffs

---

### 5.13 allStandings — Battle Outcome

```
{
  "total": 404,       // Troops at battle start (march size)
  "survival": 225,    // Troops surviving
  "dead": 179,        // Troops killed
  "wounded": 0        // Non-zero in base attacks and a few other fights
}
```

`total` (plus extra troops) is the march size and is the best single "how many troops did you bring" number.

---

### 5.14 armyUuids — Individual Troop UUIDs

Array of string UUIDs (one per troop; 404 in the reference report). Format varies by battle:
- PvP: short sequential strings (`"10001"`, `"10048"`, `"10112"`)
- Wasteland/NPC: long numeric IDs (`"4430208425366817797"`)

Troop UUIDs cross-reference to `units[].uuids[].uuid` (each slot holds about 45 individual troops).

---

### 5.15 buffs — Raw Buffs

Small array of special/conditional buffs not captured in effectBuffs by id:
```json
[{ "specialType": 1, "data": ["10112"], "bufid": 921006, "value": 7000 }]
```

- `specialType=1`: conditional buff tied to a specific unit UUID in `data`
- 921006 = "Golden Tank coming" (known), 981021 = Golden Plane, 921016 = an appendage buff (not named in the viewer). `value` varies per player (2000-9000). **Likely** per-account progress
- The same entries are also matched in `traceEffectBuffs`

### 5.16 Other players[] fields

| Field | Meaning |
|-------|---------|
| `extraArmy {armyId, armyNum}` | Bonus troops added to the march (up to 40). Matches `fightMarches[].att/defExtraArmyNum` and process event t=28. **Likely**; the cause is unidentified |
| `traps` | Empty on every side studied (also `attTraps` / `defTraps`) |
| `avatarId` | October reports only; meaning unknown |

---

## 6. units[] — Unit Formation Slots

One entry per hero formation slot (up to 9 slots), plus extra-army slots (`isExtra`).

```
{
  "uid": XXXXXXX 272,
  "uuid": "10112",                // Formation slot UUID (matches armyUuids entries)
  "uuids": [ { "uuid": "...", "isDead": 0 } ],   // individual troops in this slot (about 45)
  "armyId": 901103,
  "maxPower": 204483073614300.0,  // Raw engine power for this slot (NOT displayed CP)
  "maxElementPower": 27540000.0,
  "maxShield": 260265609623744.0,
  "isExtra": 0,
  "isDead": 0
}
```

- `maxPower` values are enormous (10^12-10^15 range): internal engine values, NOT displayed CP. `attMaxPower` in fightMarches = sum of all unit `maxPower`
- `uuids.length` = troops in that slot, the way to read "troops per slot"
- **`armyId` families (Confirmed):**
  - `1xxxx` / `3xxxx` (5 digits) = base troop lines: Army (101) and Air Force (301), level 99-103 (`10100-10103`, `30099-30103`)
  - `9NN103` = **Ultra Valhalla**: 901 Army offense, 902 Army defense, 903 Navy offense, 904 Navy defense, 905 Air Force offense, 906 Air Force defense
  - `5NNNNN` = mecha troop; the digits after 50 identify the mecha (505111 = Mecha 1005, 506371 = Mecha 1006, 507111 = Mecha 1007)
- The final digits (tier suffix such as 101/102/103 or 371/111) are not decoded (see §17). Measured per-troop `maxPower` already shows the grade difference: in one report a side's base-grade Air troops had about 32% less engine power each than its Valhalla troops

---

## 7. mechas / mechas2 — Mecha Data

Two formats exist simultaneously (legacy array + keyed). `mechas2` is keyed by fightTeamId and is the canonical format. An empty `mechas[]` means no Heavy Trooper in the march. **HT chips are in `chips[]` here**, not in `newArmyEquips`.

```
// mechas[] — array
{
  "mechaId": 1005,          // Mecha type ID (1002-1007 seen; 1008 never)
  "uuid": "10368",          // Formation slot this mecha occupies
  "skinId": 0,              // 0 on most; 10012-10014 on a few
  "skillIds": [100002, 104001, 104002, ...],  // 7 mecha skill IDs
  "parts": [34, 35, 36, ...],  // 9 mecha part IDs
  "chips": [
    { "chipId": 2500, "level": 0,  "rndAttrs": "32;2705|29;2638" },   // slot 0 (core)
    { "chipId": 2501, "level": 25, "rndAttrs": "8;688|10;499|4;1216|5;818" }, // slot 1
    ...   // 7 chip slots total
  ]
}

// mechas2 — keyed by fightTeamId string
{ "2": [ { ...same structure... } ] }
```

- `parts[9]` and `skillIds[7]` are **static per mecha** (identical for every player using that mecha): no progress information. `skillIds[0]=100002` is the base attack skill
- `chips[].chipId` = `set*1000 + rarity*100 + position` (set 1-10; rarity 5 = legendary on 93% of chips). The core chip (slot 0) stays at level 0; other chips cap at **level 25**
- `rndAttrs` = `"attrIndex;value|attrIndex;value"`. `attrIndex` is a **0-based index into that chip's own attribute pool**: core chip 33 attributes, HT chips (positions 1-3) 12, global chips (positions 4-6) 28. Value = hundredths of a percent. Chips carry 4 attributes on positions 1-6 and 2 on the core; fewer than the full count means unfinished upgrades
- Set bonuses trigger on 2 / 4 / 6 pieces of the same set

---

## 8. fightMarches[0] — Battle Outcome Data

One entry per march. A defender with several marches has several entries.

### 8.1 Combat Power (CP)

| Field | Example | Description |
|-------|---------|-------------|
| `attBattlePowerVal` | 201,082,447 | Attacker's pre-battle CP (what game displays) |
| `defBattlePowerVal` | 224,970,647 | Defender's pre-battle CP |
| `attBattlePower` | "201M" | Human-readable string |
| `defBattlePower` | "225M" | Human-readable string |
| `attMaxPower` | 1.103×10¹⁵ | Sum of all unit `maxPower` (raw engine power) |
| `defMaxPower` | 1.328×10¹⁵ | Same for defender |
| `attPower` | 6.169×10¹⁴ | Attacker's surviving power after battle |
| `defPower` | 0.0 | Defender's surviving power (0 = wiped) |
| `attArmyNum` / `defArmyNum` | 404 | Troops brought (march size); extras in `*ExtraArmyNum` |
| `attFightTeamId` / `defFightTeamId` | 2 | Which team number each side is |

See §11 for the CP analysis.

### 8.2 Army Composition (attPutArmyV2)

```json
[{ "armyId": 901103, "s1": 44 }, { "armyId": 901103, "s0": 56 }]
```

| Key | Meaning |
|-----|---------|
| `s0` | troops that survived |
| `s1` | troops that died (sums to `allStandings.dead` on 45 of 46 sides) |
| `s2` | troops that were wounded (base attacks, a few other fights) |
| `s3` | survivors in groups that also have wounded |
| `s0Extra` / `s1Extra` | the extra-army share of survivors / dead |

`attPutArmy` is the legacy format (less detailed). `armyId` is decoded in §6.

### 8.3 Skill Damage (attSkillHurts)

```
{
  "total_hurt": 28399440,
  "common_hurt": 18243200,
  "common_count": 404,
  "skill_hurts": { "10332": 4823040, "10169": 3421200 },
  "skill_hurt_count": { "10332": 6, "10169": 3 },
  "effect_buff_hurts": { "100008": 1234567 }
}
```

- `total_hurt` = `common_hurt` + sum of `effect_buff_hurts` (within 0.1% on 41 of 45 sides)
- `skill_hurts` is keyed by the hero's **native skill id** (`10000 + heroId`), one hero per key. Other keys are formation skills: 941212 Drone, 941215 Scorpion Counterblow, 941216/7 Eagle Strike
- `skill_hurt_count` counts skill procs per hero (1-25 per fight), not hits; RNG-heavy
- `effect_buff_hurts` breaks out buff ticks such as Magnetic Pulse, Ignition, Counterblow and mecha chip skills. Some ticks are owned by no hero skill, so `sum(skill_hurts)` does not equal `sum(effect_buff_hurts)`
- A hero with `skill_hurts` 0 and `skill_hurt_count` 0 in a fight of 10+ rounds never fired (a deliberate support hero, or a swap candidate; the report cannot tell)
- Skills account for 79-100% of damage for **everyone**, winners and losers alike, so "skill share" separates nobody. **Damage per trigger** does

### 8.4 Traps

`attTraps`, `defTraps` and `players[].traps` are empty on all 46 sides studied, including a base-defense report. The format is still unknown.

### 8.5 Special Buffs

```json
"attSpecialBuff": { "effects": [{ "id": 1147, "val": 5000 }], "traceEffects": [{ "id": 1147, "val": 5000, "s": 31 }] }
```

**Confirmed:** present only for a side that fields an Ultra Valhalla troop. The effect id names the Valhalla line, matched exactly to `units[].armyId` in all 23 cases:

| Effect id | armyId on that side | Line |
|-----------|--------------------|------|
| 1139 | 902103 | Army defense |
| 1143 | 901103 | Army offense |
| 1145 | 903103 | Navy offense |
| 1166 | 904103 | Navy defense |
| 1147 | 905103 | Air Force offense |
| 1167 | 906103 | Air Force defense |

`val` is 2000, 3500 or 5000, probably a per-line upgrade tier (**Likely**; 5000 = best seen). It is not in `effectBuffs`, so it applies in battle only. Some 905103 sides show no special buff (**Guess:** tier 0).

### 8.6 How Close Was the Fight

Every duel in the study ended in a wipe of the loser, so "who won" says little. A better measure is **kill progress**:

```
loser_progress = loser.total_hurt / (2.7 x winner.maxPower)
```

- The 2.7 factor is empirical: across 21 wipes the winner's `total_hurt / loser.maxPower` was 1.6-5.0, median 2.7 (the 5.0 is a one-round overkill). A side needs roughly 2.7 times the enemy's `maxPower` in damage to wipe it
- **Confirmed:** `loser_progress` tracks the winner's troop loss within about 4 points in all 11 fights where the loser dealt a meaningful share (for example 45 vs 48, 40 vs 44, 35 vs 32)
- Reading it: **>= 30%** (or winner lost >= 30%, 40+ rounds) = close fight where small upgrades matter; **<= 3%** with <= 10 rounds = a stomp, compare the whole march first
- Rounds = `(last tl - 600) / 1200`
- The factor was fitted on mostly one hero set and troop type, so treat it as ordinal until re-fitted on Army/Navy and non-Valhalla marches
- CP is an input, not the outcome: the higher CP won 19 of 21 duels, but at the same CP ratio the margin varied from 1% to 45%

Other usable fight signals: troops brought and per slot (`units[].uuids.length`); engine power per troop (`maxPower / troops`); troop grade (base vs Valhalla vs mecha troop); whether a Protection supporting skill for the branch is equipped (all 21 winners had one); damage per skill trigger by hero; heroes that never fired. Not usable: skill share of damage, crit/dodge, totals from `process[]`, or "who dealt more damage" (the loser often does).

---

## 9. process[] — Round-by-Round Events

The battle simulation log. 438-3350 events observed depending on battle length. **It is a sample, not a complete record** (see below). 27 distinct `t` types appear in the study; the 19 below are decoded, the 8 added in 2026 are only partly understood.

```json
{ "val": 2.0, "selfVal": 0.0, "t": 4, "tl": 0, "source": "0" }
{ "val": 1.0, "selfVal": 0.0, "t": 30, "tl": 0, "source": "2", "target": "1" }
```

| Field | Description |
|-------|-------------|
| `t` | Event type (see below) |
| `tl` | Timeline position (milliseconds; increments by 1200ms per round) |
| `source` | Source entity (team ID or unit UUID string) |
| `target` | Target entity (optional) |
| `val` | Primary value (damage, count, etc.) |
| `selfVal` | Secondary value (self-damage, etc.) |
| `id` | Event sequence number (0, 1, 2, ...); present in newer reports only. **Likely** |
| `actType`, `eType` | On t=2 events, values 1/2/3 each. Meaning unknown (see §17) |

**`t` event type reference (19 decoded types):**

| t= | Name | Key Fields | Description |
|----|------|-----------|-------------|
| 1 | UNIT_REGISTER | source=unitUUID, val=gridPos | Position assignment at tl=0. Grid: 1/2/3=row0, 11/12/13=row1, 21/22/23=row2. Both sides share the same index, fighting from opposite sides of the diagonal. |
| 2 | UNIT_ATTACK | source=uuid, target=uuid, val=newHP, eValue=dmgDealt, oldHit=prevHP, power=unitMaxPow | Standard troop-vs-troop attack hit. `val` = HP after the hit, `eValue` = damage amount. Most frequent event in combat rounds. |
| 3 | SKILL_SELF_BUFF | source=uuid, hero=heroId, skill=skillId, val=power, triggerType=1 | Hero passive skill triggers as a self-buff on its own formation unit. Very frequent. |
| 4 | BATTLE_INIT | source='0', val=teamCount | Single event at tl=0. val=2. |
| 5 | HERO_SKILL_ACT | source=uuid, hero=heroId, skill=skillId, num=count | Hero activates their active skill on a unit. Distinct from t=7 which is the resulting damage hit. |
| 6 | HERO_REGISTER | source='0', hero=heroId, skill=1 | Hero roster registration at tl=0. One event per hero. |
| 7 | HERO_SKILL_HIT | source=uuid, target=uuid, hero=heroId, skill=skillId, val=damage | Hero skill deals direct damage to a specific enemy unit. |
| 9 | UNIT_INIT_STATS | source=uuid, val=HP, selfVal=shield, power=maxPower | Unit initial HP and shield values at tl=0. One event per unit. |
| 11 | EQUIP_SKILL_BUFF | source=teamId, target=uuid, skill=runeSkillId, directAddEffect=1 | Hero equipment/rune skill applies an effect to a unit. source='1' or '2' (team ID, NOT a unit UUID). |
| 14 | BUFF_DAMAGE | source=uuid, target=uuid, buffId=id, skill=skillId, power=unitPow, val=damage | Buff/DoT damage tick (Magnetic Pulse, Ignition, secondary effects). Same ids as `effect_buff_hurts`. |
| 19 | UNIT_DEATH | source=uuid, val=0 | A unit/formation is eliminated. Appears at the tl of the killing blow. Not 1:1 with troop deaths. |
| 25 | FORMATION_SKILL | source=uuid, target=uuid, skill=formationSkillId, val=buffId | Formation skill activation. skill=941215 = Scorpion/Counterblow, val=100008. |
| 27 | MECHA_ACTION | source=mechaUUID, val=actionValue | Mecha performs a special action. val is mecha-type-specific. High frequency when a mecha is active. |
| 30 | TEAM_SETUP | source=attTeamId, target=defTeamId, val=1 | Single event at tl=0 identifying which team is attacker (source) and defender (target). |
| 31 | STAGE_REGISTER | source='0', val=stageIndex | Stage/phase registration at tl=0, sequential. Exact purpose unconfirmed. |
| 32 | BUFF_APPLY_MULTI | source=uuid, targets=csv(uuids), nums=csv(stacks), buffId=id, skill=skillId | Buff applied to multiple targets at once. |
| 33 | UNIT_TURN_START | source=uuid | Unit's turn begins in the current round. |
| 35 | STAGE_CHANGE | source='0', val=stageNum, selfVal=totalStages | Stage transition (multi-stage battles only; about every 21,600ms). |
| 37 | RUNE_ACTIVATE | source=uuid, hero=heroId, skill=runeSkillId, val=0 | Rune skill activates on a unit; skill is a 400xxxx rune skill id. |

**Types added in the 2026 study (partly understood):**

| t= | Observed | Confidence |
|----|----------|-----------|
| 8 | Source-only marker (24 events, 11 reports) | Guess |
| 10 | Hero skill damage at tl=0, with `val` = damage and `id` | Guess |
| 17, 23 | Two skill=2 markers at tl=0 | Guess |
| 26 | `val` = buff id, `selfVal` 1, source equals target (a unit gets a buff) | Guess |
| 28 | Extra-army registration: `selfVal` = armyId, `num` = number of extra troops (equals `extraArmy.armyNum`) | Likely |
| 29 | `selfVal 1001`, `power 28`, `val` 2 or 3 | Guess |
| 34 | Single-target buff apply (like t=32 with one target: `buffId`, `skill`, `hero`, `num`) | Likely |

**Important limits (Confirmed):**
- **`process[]` is a sample.** Summed `t=14` damage is only 30-70% of `effect_buff_hurts`; `t=2` `eValue` totals 0.3-3% of `common_hurt`; `t=19` has 14 events for 334 dead troops. Never total damage, kills or deaths from it. Use `fightMarches[].*SkillHurts`, `allStandings` and `units[].uuids[].isDead` instead. Usable from `process[]`: number of rounds and rough timing
- **There are no crit, dodge, heal or shield event types.** A crit-resistance or "unlucky crits" claim cannot be supported from a report. On `t=2`, `actType` 1/2/3 splits about 10% / 80% / 10% for both sides regardless of outcome (`actType 1` hits deal about 0.09x and 3 about 1.7x a type 2 hit, so 1 may be glancing and 3 a crit; **Guess**)

**Timeline (`tl`):** increments by 1200ms per round. `tl=0` = pre-battle init, `tl=1800` = round 1, `tl=3000` = round 2, etc. Observed max about 100,200ms in a long battle.

**Source/target UUID conventions:**
- `'0'` — system/global (BATTLE_INIT, HERO_REGISTER, STAGE_REGISTER, STAGE_CHANGE)
- `'1'` or `'2'` — team IDs (TEAM_SETUP, EQUIP_SKILL_BUFF source)
- `'10xxx'` — attacker formation slot UUIDs (short sequential, PvP)
- `'50xxx'` — defender formation slot UUIDs (short sequential, PvP)
- Long numeric (e.g. `'4605144622810820666'`) — formation slot UUIDs in wasteland/NPC battles (both sides)

**The in-game replay is driven by `process[]`.** When the Replay button is clicked, the client passes `process[]` to its battle renderer, which draws the battle live. No MP4/WebM/M3U8 exists. Since the log is a sample, the replay is a visualization of the logged events, not a complete recount.

---

## 10. etReward — Battle Rewards

```
[
  { "count": 1, "id": 69159 },
  { "count": 1, "id": 2300003 }
]
```

- Present in only 1 of the 23 studied reports (a guardian win); absent on the losing side
- Item IDs correspond to the game's item table
- Not currently rendered in the viewer

---

## 11. Combat Power: How It's Calculated

**`attBattlePowerVal`** is a pre-computed server value, not derived from `effectBuffs` at read time.

**The formula (approximate):**
```
CP ≈ Σ over troop_types of:
  troop_count_of_type × base_CP_of_type × buff_multiplier
```

Where `buff_multiplier` applies only a **subset** of buffs (the permanent ones).

**`maxPower / CP` is NOT a constant ratio.** The April claim of about 5.4 million within 2% was wrong. Over 45 sides it ranged from 3.6M to 25M, median 4.8M. Time Clash sides sit at 3.6M-5.0M, guardian/city/PvE sides at 5.8M-7.7M, and two outliers at 15M and 25M. The two numbers move together but are not interchangeable.

**Per-troop average CP observed:**
- Report 1 (PvP, fightType=114): 201,082,447 ÷ 404 troops = **~497,729 CP/troop**
- Report 2 (Guardian, fightType=126): 192,229,892 ÷ 404 troops = **~475,817 CP/troop**

The difference between same-troop-count marches likely comes from troop tier/rank composition (armyId differs: 901103 vs 905103).

**Why effectBuffs don't "add up" to CP:**
- `effectBuffs` total ≈ 3,048,107 raw units = **30,481% combined across all stat types**. This is NOT a single percentage applied to CP
- Different buff IDs apply to different stats (ATK, HP, DEF, Lethality, etc.)
- CP uses only a subset, plus troops and heroes brought

**CP is a weak proxy for buff strength.** In one study of 23 accounts, rank correlation between CP and a buff-strength score was only 0.83. One account had top-tier buffs at 28M CP (its CP was low because of troops and heroes), and an account at 65M CP had a higher buff score than one at 69M. A single account's CP ranged from 5M to 237M across its own reports while its buff score barely moved. Do not infer buff gaps from CP gaps.

---

## 12. Wasteland / Territory Buffs and CP

Several sources are tied to the battle mode, not to the account. Drop them when comparing accounts.

| Source (`s=`) | Where it appears | Verdict |
|---------------|------------------|---------|
| 112 | fightType 127, 126, 114 and 2, never in Time Clash (21). Ids 992575-992580 (e.g. 992576 = 108,000 raw HP) | Mode-specific (Confirmed). Whether CP excludes it: **Guess** (not checkable without a CP breakdown) |
| 135 | One report only (fightType 126, id 992578 = 50,000) | Mode-specific |
| 111 | Only the **defender** in the fightType 127 report: ids 9301160/9301170 at 150,000 raw each (+1,500% ATK and HP) | Mode/defender-specific; doubles that side's buff total. Drop |
| 65 | fightType 21, 2 and 126 (11 sides). Ids 990549/550, 990191/192, 990197/198, 991007 | **Not** guardian-only. Labelled Base Skin Skill |
| 134 | **Present with identical values in ordinary Time Clash (fightType 21) fights** of several accounts (9301160 +5,000, 9301170 +2,000, 990202 +400) | The game labels it Base Effect, so it is a permanent account buff (not wasteland). Keep it |
| 44 | Present at 77-185 pp core in every Time Clash side | Labelled Loot Quest; permanent in practice (Likely). Keep |
| 95 | 10-20 pp on most sides | Small; mode-tied by name |
| 64 (BUFF Item), 104 (Hero Pre-battle Skill) | Small | See §13 |

**The HP column in stats comparison shows very high values in wasteland battles** because s=112 ids such as 992576 (1080% HP) and 992575 (540% Infantry ATK) inflate the totals.

**Role-specific buffs (Confirmed, Time Clash):** 941306 "Hall of Valhalla HP Boost" (+20% HP, Base Defense, s=31) appears on every defender and no attacker; 990209 "DEF when defending buildings" (defender) vs 990208 "when attacking" (attacker), same size; 29901301 Decreased DMG Taken (defensive) vs 29902211 (offensive), same size. Fort buffs 1002001/1002002 (s=6) appear for both roles. Net effect of role in Time Clash is about +20 pp HP for the defender.

---

## 13. Buff Source Reference (traceEffectBuffs s= values)

The `s` field = `type_small`. The labels below are the game's own table `fight_report_from_show` (104 rows: `type_small` -> `type_big` -> label; several `type_small` values share one display group). Notes are from the 2026 study.

| s= | Group (type_big) | Label | Notes |
|----|-----------------|-------|-------|
| 0, 78-80, 82, 85, 86, 89-93 | 0 | Others | |
| 1 | 1 | Science | Small |
| 2 | 2 | Hero Skills | Hero-specific |
| 3 | 3 | Hero Talent | Talent bonuses of the fielded heroes (§5.3) |
| 4 | 4 | CPNT | Near-constant on one server |
| 5 | 5 | Suppress | |
| 6 | 6 | Alliance Tech | (label key only; named Alliance Tech in the viewer) |
| 7 | 7 | Decoration | Placed decorations + suits (§5.5) |
| 8 | 8 | VIP privileges | |
| 9 | 9 | Class Talent | |
| 10 | 10 | Mechanical Master Potion | |
| 12 | 12 | Strengthening/Weakening building | 930100/930200 +2500, 1101067 +1500; negative on one attacker. Context bonus, not account progress |
| 13 | 13 | Alliance Badge | (label key only; named in the viewer) |
| 16 | 16 | Hero Bond | |
| 17 | 17 | Mastery | Army mastery (§5.12) |
| 18 | 18 | Special Effect Upgrade | 9990011-19 (base skin skills); smallest value per side is a multiple of 20 (value/20 probably an upgrade count, Likely) |
| 19 | 19 | Medal Promotion | |
| 20 | 20 | Alliance Titles | |
| 21 | 21 | Eternal Tech | |
| 24 | 24 | Morale | 1101042, 1131001, 1131002 |
| 25, 75, 76, 77, 134, 88 | 25 (88 -> 65) | **Base Effect** | Fully explained, see below |
| 26 | 26 | Unit Equipment | The 3x7 `armyEquips` grid (§5.9); ids 930001-3, 930101-3 |
| 27, 66 | 27 / 66 | Heavy Trooper | 933003/5, 937001-4 |
| 28, 29, 30 | 28 | (label key only) | |
| 31 | 31 | Odinium Tech | Includes the Hall/Ultra Valhalla family (941300-941313, 1139-1167); 941306 = defender-only +20% HP |
| 32, 41, 68, 69 | 32 | Formations / **Formation Talent** (s=41) | Ids 1000021 (HP) / 1000022 (ATK) |
| 33 | 33 | Secret Weapon | |
| 34, 96 | 34 / 96 | Alliance Monster | 230140-230145 |
| 37 | 37 | Titan Gear | Special effects 400305/311/318 |
| 38 | 38 | Master Race | |
| 39, 83, 84 | 39 | Titan Canyon | |
| 40, 105 | 40 / 105 | War Ethos | |
| 42 | 42 | Kartz's Trial | |
| 43 | 43 | Warzone Ranked Match - Charge Ahead | |
| 44, 99-103 | 100 / 99 | Loot Quest | s=44 is present on every Time Clash side (permanent, Likely) |
| 63 | 63 | Island Battle | |
| 64, 73, 74 | 64 | BUFF Item | Temporary items. 990117/118/121 at 2000/2000/1000 where present |
| 65 | 65 | Base Skin Skill | Castle-skin skill family (990549/550, 990191/192, ...); seen in fightType 21, 2 and 126 |
| 67 | 67 | Valhalla Units | |
| 70 | 70 | Eternal Land | |
| 71, 72 | 71 | Terra Aurorae | |
| 81 | 81 | Tower | |
| 87 | 87 | Dominator | |
| 94 | 94 | Airdrop | |
| 95 | 95 | Ruin Bonus | |
| 97 | 97 | Purifying Turbine | |
| 98 | 98 | Official Position Effect | |
| 104 | 104 | Hero Pre-battle Skill | Id 990145, per-player value 3149-3379 |
| 106, 107 | 106 | Enigma Beast | 106 beast skill buffs (520xxx); 107 field attributes |
| 108 | 108 | Enigma Realm | |
| 110 | 110 | Masters: Stellar Expedition | |
| 111 | 111 | Wasteland Airship Effect | Defender in fightType 127 |
| 112 | 112 | Wasteland Effect | Mode-specific |
| 114-117 | 114 | **Hero Awakening** | Ids 521011/012/013 (one per troop type: 011 Army, 012 Navy, 013 Air Force); 521021-23 too. s=114 values 2,400-24,000 (percent-like); s=116 values 10 or 20 (levels). Varies with the awakened heroes fielded |
| 120 | 120 | Return Bonus | |
| 122 | 122 | (label key only) | |
| 123, 124 | 123 | Global Masters Match | |
| 125 | 125 | Hero Trial | |
| 126 | 126 | Legendary Squad | |
| 129 | 129 | Buffs Provided by Passengers | |
| 135 | 135 | Warzone Guardian | Mode-specific |
| 136 | 136 | Seal Stone Chaos | |
| 138 | 138 | Avatar Frame | |
| 139 | 139 | **Hero Skin** | New in the October patch (1.680): id 9301170 (All Units HP) at 1000, 2000 or 3000 |

**Base Effect (Confirmed, exact on every current report; older reports differ only because skin values were rebalanced since):**
- **s=25** = the `own_buff` of every skin in `effectSkins.effectIds`, plus the `equip_buff` of the skin in `useIds`, plus the buff of each `exhibitions` id (game table `skin_exhibition`)
- **s=75** = castle effects: `own_buff` of `ownCastleEffects` plus `equip_buff` of `useCastleEffects` (exact on all 46 sides)
- **s=76** = march skins: own plus equip of `ownMarchSkins` / `useMarchSkins` (game table `march_skin`; exact on all 46 sides)
- s=134 and s=88 also belong to the Base Effect group; s=134 shows identical values on several accounts in ordinary Time Clash fights

**Which sources separate strong from weak accounts** (23 accounts on one server, Time Clash, troop-weighted, mode sources dropped): Enigma Beasts about 30% of the strong-vs-weak gap, Base Effect 23%, Decorations 16%, Unit Equipment 12%, Formations 7% (88% together). Class Talent, Alliance Tech/Badge/Monster, VIP, Science, CPNT and Secret Weapon barely vary on one server, so they are rarely the advice. Hero-dependent sources (Hero Skills, Hero Talent, Hero Bond, Heavy Trooper, Titan Gear) vary with the march, not the account.

---

## 14. fightType Reference

| fightType | Name | Notes |
|-----------|------|-------|
| 2 | City / base attack | Defender can field several marches; `pointStr`/`srcPointStr` present |
| 5 | PvE | `pveId` present; defender with 0 heroes |
| 21 | Time Clash Arena / SVS Warzone | PvP; the cleanest mode for comparing accounts |
| 86 | Loot Quest Battle | PvP with loot quest modifier |
| 114 | Standard PvP | Observed in report 4605240496314802177 |
| 126 | Attack on Warzone Guardian | NPC wasteland guardian; report 4605144673410904064 |
| 127 | Airship / wasteland-type fight | Observed in a 2026 report: s=111 on the defender, s=112 on the attacker |

---

## 15. Assets: Icons and Portraits

| Asset | Location | CDN? | Notes |
|-------|----------|------|-------|
| Battle report JSON | `fight-report-va.oss-accelerate.aliyuncs.com/prod/{first4}/{id}.json` | ✅ Public | |
| Hero portrait (small, 80×80) | `h5.topwargame.com/DynRes/images/headpic/hero_icon{heroId}_global.png?t=22.jpg` | ✅ | |
| Hero portrait (in-game avatar) | `h5.topwargame.com/DynRes/images/headpic/{avatarurl}.png?t=21.jpg` | ✅ | |
| Player custom photo | `knight-cdn.akamaized.net/headimg/{hash}.jpg` | ✅ | From `playerInfo.headimgurl_custom` |
| Hero active skill icons | `assets/skill-icons/skill_{skillId}.png` | Repo | 135 extracted, served via raw.githubusercontent.com |
| Exclusive skill icons | `assets/exclusive-skill-icons/exc-skill-{heroId}.png` | Repo | 123 extracted |
| Rune icons | `assets/rune-icons/{rune-name-kebab}.png` | Repo | 14 PNGs |
| Mecha icons (large) | `assets/mecha-icons/mecha_{id}.png` | Repo | 11 extracted |
| Unit icons | `assets/unit-icons/` | Repo | 5 only (Army only); Air Force/Navy missing |
| Skill/mecha/unit atlases | Packed sprites in game bundle | ❌ No direct URL | Must crop from 990×990 atlas PNG |

---

## 16. What We've Built (Pages + Skills)

### battle-report.html
`https://2864tw.com/battle-report.html`

**Implemented:**
- Fetch report by ID from CDN (or via `?id=` URL param)
- VS section: player names/avatars, CP, result, troop counts
- Hero panel: portraits, exclusive skill, equipped skills (name + icon), titan gear (slot/level/rune/stats/enhancement), beast field hero modal
- Skill damage table: per-hero damage totals, activation counts, percentage bars
- Army composition: troop types and counts from `attPutArmyV2`
- Stats Comparison tab: full effectBuffs breakdown by category, per-source drill-down (traceEffectBuffs), decoration drill-down (effectDecorations), enigma beast comparison, formation display
- Passthrough: `?id=` param propagates to related pages (decor-index, enigma-viewer)
- Raw JSON toggle (collapsible)

**Not yet implemented:**
- Unit Equipment display (`armyEquips` / `newArmyEquips`) and collectibles (`CTCs`)
- Battle rewards (`etReward`)
- Trap data (`attTraps` / `defTraps`; always empty anyway)
- Mecha skill details beyond icon
- Multi-march defenders (several `players[]` entries)
- Awakening fields (`awakenLevel`, `awakenSkillMap`) and the real exclusive/supporting skill levels

### battle-replay.html
`https://2864tw.com/battle-replay.html`

Standalone Canvas replay renderer built from `process[]` (positions t=1, HP t=2/t=19, skills t=7/14/25), with playback speed, scrubber and event log. Shareable via `?id=`. Because `process[]` is a sample, totals shown are not authoritative (§9).

### enigma-viewer.html
Referenced via `?id=` passthrough. Shows enigma beast comparison.

### decor-index.html
Referenced via `?id=` passthrough. Shows decoration details.

---

## 17. Unmapped / Unknown Areas

Only items that are **still unknown** as of 2026-10. Everything resolved in the 2026 study is documented in the sections above.

### Buff sources and ids

| Area | Status | Notes |
|------|--------|-------|
| **Which awakened-hero state drives s=114-117 and s=139 values** | Open | Labels are known (Hero Awakening, Hero Skin); the exact rule behind each value is not |
| **Pool arithmetic** | Open | Whether ATK/HP/DMG pools add or multiply is not in the report. Percentage-point gaps rank sources; they do not predict damage |
| **Unclassified ids** | Open | About 120 display ids sit outside the stat tables: hero-skill conditionals (s=2), beast skill buffs (s=106, 520032-520140), gear special effects (s=37), HT specifics (s=27), small ids under s=9/18/25/65. A few October ids (1151/1157/1163, 521022, 932018, 990871, 9301175) are probably new Valhalla tier lines and decoration entries (**Guess**) |
| **BUFF_GROUP coverage** | Ongoing | The viewer maps about 168 ids; new reports may contain unmapped ids that are dropped silently |

### Fields and structures

| Area | Status | Notes |
|------|--------|-------|
| **`newArmyEquips` `eid` table** | Open | 910xxx ids are not in the item or building tables; the `eid` changes on the first upgrade; `suit` 0-12 meaning is Likely only |
| **`elementLevels` caps and effect** | Open | Range 102-278; three equal values on 44 of 46 sides |
| **Troop tier from `armyId` suffix** | Open | Line family is decoded (§6); the 101/102/103 and 371/111 suffixes are not. Per-troop engine power measures the difference in practice |
| **`actType`, `eType` on t=2** | Open | Values 1/2/3 split about 10/80/10 for both sides; no crit ground truth |
| **Process types 8, 10, 17, 23, 26, 29** | Open | See §9 |
| **`extraArmy` / `*ExtraArmyNum`** | Open | Up to 40 bonus troops; cause not identified; meaning of `s0Extra`/`s1Extra` |
| **Locked vs unfilled supporting slot** | Open | A missing fourth `nonActiveSkills` entry could be either |
| **`exhibitions`, `ownHeadframes`, `magicIds`, `useInfos`** | Open | In `effectSkins`; no meaning or cap established |
| **`fightType2`** | Guess | 32-byte base64 value, different per report |
| **`avatarId`, `H.skinId`, `starExp`, `changeId`, `dressId`, `rateEquipId`** | Open | Constants or cosmetics; no meaning established |
| **`stamp` cap** | Open | 1003-1009 (Combat Elite), 2003-2008 (Mechanical Master) seen |
| **Traps** | Open | Empty on all 46 sides |
| **`suitLevelBuff` tier meaning** | Open | Tier 1-3 in reports vs a max level of 2 in the suit table |
| **Valhalla tier** | Likely | `*SpecialBuff` 2000/3500/5000 as a per-line upgrade tier; tier-0 sides show nothing |
| **Command vs march size** | Disputed | Command 2500 vs 2000 correlates with four vs three supporting slots, but per-slot troop counts do not follow command alone (see §5.3) |

### Low priority

`isolated`, `headFrameId`, `attOverRoundNum`/`defOverRoundNum` and `cn` are constant (0/false); `ver`, `stage` and `reportComparison` are constant in all 23 reports.

------|--------|-------|
| `isolated`, `headFrameId` | Always 0 | |
| `attOverRoundNum` / `defOverRoundNum` | Always false | |
| `cn` in playerInfo | Always 0 | |
| `ver`, `stage`, `reportComparison` | Constant in all 23 | |

---

## 18. Key Gotchas

1. **Only `nonActiveSkills[0]` (the native passive, `skillId = 10000 + heroId`) is always level 0.** `[1]` is the exclusive skill with its real level (3-9 seen, cap 10); `[2..5]` are supporting skills; `{0,0}` is an empty slot. Awakening is in `awakenLevel` / `awakenSkillMap`.

2. **`effectBuffs` values are raw: 100 = 1%.** Roll ids up to their display stat and troop type before comparing; a whole-pool sum is meaningless.

3. **CP (`attBattlePowerVal`) is not buff arithmetic** and is a poor proxy for buff strength (rank correlation about 0.83). `maxPower / CP` is not constant (3.6M to 25M).

4. **`sublimes[].buffs[].value` is only the Advanced Upgrade bonus**, and `boxMaxIds` is not a max flag (one id per owned decoration group). Use `effectDecorations.ids` for placed pieces.

5. **`armyUuids` format changes by battle type** (short sequential in PvP, long numeric in wasteland/NPC); both cross-reference `units[].uuids`.

6. **`heroEquips` exists only inside `heroList[]`.** There is no `players[].heroEquips` key (not even null). `equipId = 100000 + slot*100 + quality`.

7. **`players[]` is per march.** A defender can field several; march-level mirrors cover `players[0]` only. Use `fightMarches[].attFightTeamId` to find the attacker; never assume team 2.

8. **`CTCs` are Loot Quest collectibles**, not HT chips. HT chips are in `mechas[].chips[]`; `armyEquips` is the Unit Equipment grid; `newArmyEquips` is a separate piece-progress list.

9. **`effectSkins.effectIds` lists skins that give their bonus**, not every skin owned. s=25 is that ownership bonus (+ equipped skin + exhibitions); s=75 castle effects; s=76 march skins. City skill buffs are under s=18.

10. **An empty Enigma field is left out of `enigmas.fields`.** Absent means no beasts deployed, not necessarily locked. Cap slot levels at 20 (fields 1-3) and 40 (fields 4-5).

11. **Hero `command` 2500 vs 2000 tracks the number of supporting-skill slots shown (four vs three), but is not a reliable measure of march size.** Read `allStandings.total` and `units[].uuids.length`.

12. **`process[]` is a sample and has no crit, dodge, heal or shield events.** Never total damage, kills or deaths from it. Use `skill_hurts`, `allStandings` and `isDead` flags.

13. **Compare accounts fairly.** Weight each source by the troop type the player fielded (from `attPutArmyV2`), drop mode and role sources (s=111/112/135, 941306, 990208/990209, 29901301/29902211), keep s=134 and s=44, and use hero-dependent sources only when the hero sets match. Do not mix wasteland/guardian reports with Time Clash.

14. **`attSpecialBuff` / `defSpecialBuff` are Ultra Valhalla troop-line bonuses** (ids 1139-1167 = armyIds 90x103) and do not appear in `effectBuffs`.

15. **Number formatting:** K/M/B/T then aa/bb/cc/dd/ee/ff/gg beyond a trillion (not Q). `etReward` is rare and only on the winning side; `attPower` is remaining engine power (0.0 = wiped); `maxPower` in `units[]` is engine power, not CP, but useful per troop. Dark Force / NPC attack reports are not on the CDN.

16. **A report is a snapshot of ONE march, not the account.** Bench heroes, other marches, the bag and unfielded branches (a branch the player does not field shows empty) are invisible. A weak march can hide a strong account.

17. **BUFF_GROUP may have unmapped ids in new reports.** Unmapped ids are silently dropped in the viewer; run a grouping check on any new report before trusting stat comparisons.

---

## 19. Reading a Report for Advice

A neutral summary of the method that came out of the 23-report study. The report shows almost every part of one march at full detail and every buff split by source, so it can say **where** an account is behind and **which concrete upgrade** closes it. It cannot see the bag, so it cannot say what the player can afford.

### 19.1 Step 1: how did the fight go?

- Compute **kill progress** (§8.6): `loser.total_hurt / (2.7 x winner.maxPower)`. Close (>= 30%) means small upgrades matter; a stomp (<= 3%) means compare the whole march first
- Reliable signals: troops brought and per slot; engine power per troop; troop grade (base vs Valhalla vs mecha troop); missing Heavy Trooper; supporting-skill build (no branch Protection, empty or wrong-branch slots); damage per trigger by hero; a hero that never fired
- Not usable: skill share of damage, crit/dodge, totals from `process[]`, raw damage comparison across sides

### 19.2 Step 2: where is the account behind?

1. Sum `traceEffectBuffs` by source, rolled up to stats (ATK, HP, DMG Increase, Decreased DMG Taken as the core)
2. **Weight by the troop types the player fielded** (from `attPutArmyV2`). Weighting matters: with even thirds one account's Unit Equipment gap looked 1,155 pp, with fielded weighting 697 pp
3. **Drop** mode-only and role-only sources (s=111, 112, 135, 95; 941306; the attack/defend variants). **Keep** s=134 and s=44
4. Compare to the opponent in the same report as a reference point, and ideally to a typical strong account. Read ratios per stat as well as pp; a broad gap (every stat 1.3x to 1.5x) means no single fix
5. Compare like with like: account-level sources always; hero-dependent sources (Hero Skills, Hero Talent, Hero Bond, Heavy Trooper, Titan Gear) only if the hero sets match
6. Five sources make up about 88% of the strong-vs-weak gap: Enigma Beasts, Base Effect, Decorations, Unit Equipment, Formations (§13). Class Talent, Alliance Tech, VIP and Science rarely vary, so they are not advice
7. Caveats: share of a gap is not return on effort (no costs in a report); pool arithmetic (add vs multiply) is unknown; Hero Awakening (s=114-117) and Hero Skin (s=139) can hide a lever larger than any listed, so read them as awakening and skin state

### 19.3 Step 3: which lever closes the gap?

Each source drills down to a lever with a current state and a cap, all readable from one report:

Levers and caps (state fields are in the sections named): hero talents (`ttsr`, 10000 each, §5.3); exclusive skill (cap 10, milestones 3/5/7); supporting skills (cap 10, fill empty slots, match the branch); awakening (40 levels, four skills at 10); titan gear level (20), refinement per stat (70% gate, 100% max) and rune star (§5.4); Enigma beasts (star 5, level 100/80/60, potential 16000) and field slot levels (20 or 40) plus empty or absent fields (§5.7); decoration piece levels and 23 suits (§5.5); base skins, castle effects, march skins and exhibitions (ownership, a collection goal, §5.6); formation level 50, quality 10, sciences 15/15/30/60/120, mastery tier 5 (§5.8); army mastery 20 (§5.12); Unit Equipment 100 / 10 (§5.9); HT chips level 25, rarity 5, full attribute count (§7); collectibles level 5 and 4 themes (§5.10).

Ordering that worked in practice (a judgement, not a measured result): **free moves first** (fill an empty skill slot, deploy beasts into an empty field, move a skill back onto the main march, equip a combat skin instead of a gathering one), then **cheap moves** the player can afford now (this needs the bag), then the **biggest gap** phrased as a direction rather than a shopping list. Never advise on hero-specific sources unless the heroes match.

### 19.4 Limits

- One report is one march; an account can look weak because its weaker heroes were fielded
- The study was 23 accounts on one server, mostly Air Force Time Clash, and every hero was level 120 with 5 stars; thresholds (the 2.7 kill factor, the 70% gear gate) need more Army/Navy and lower-level reports
- Things a report cannot show: the bag (shards, repeaters, rune fodder, skill books), bench heroes, spare beasts, other marches, VIP and class talent levels, research levels
