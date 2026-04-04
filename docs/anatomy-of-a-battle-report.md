# Anatomy of a Battle Report
**Last updated:** 2026-04-04  
**Based on reports:** 4605240496314802177 (PvP, fightType=114), 4605144673410904064 (Wasteland Guardian, fightType=126)  
**Built from:** battle-report.html viewer, buff mapping memory, attributes spec, CDN/asset references, direct JSON analysis

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
   - 5.9 [armyEquips / newArmyEquips — HT Chips](#59-armyequips--newarmyequips--ht-chips)
   - 5.10 [CTCs — Collectible HT Chips](#510-ctcs--collectible-ht-chips)
   - 5.11 [elementLevels — Element Levels](#511-elementlevels--element-levels)
   - 5.12 [armyMastery — Army Mastery](#512-armymastery--army-mastery)
   - 5.13 [allStandings — Battle Outcome](#513-allstandings--battle-outcome)
   - 5.14 [armyUuids — Individual Troop UUIDs](#514-armyuuids--individual-troop-uuids)
   - 5.15 [buffs — Raw Buffs](#515-buffs--raw-buffs)
6. [units[] — Unit Formation Slots](#6-units--unit-formation-slots)
7. [mechas / mechas2 — Mecha Data](#7-mechas--mechas2--mecha-data)
8. [fightMarches[0] — Battle Outcome Data](#8-fightmarches0--battle-outcome-data)
   - 8.1 [Combat Power (CP)](#81-combat-power-cp)
   - 8.2 [Army Composition (attPutArmyV2)](#82-army-composition-attputtarmyv2)
   - 8.3 [Skill Damage (attSkillHurts)](#83-skill-damage-attskillhurts)
   - 8.4 [Traps](#84-traps)
   - 8.5 [Special Buffs](#85-special-buffs)
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

---

## 1. CDN Access

```
URL: https://fight-report-va.oss-accelerate.aliyuncs.com/prod/{first4}/{reportId}.json
Example: reportId=4605240496314802177 → .../prod/4605/4605240496314802177.json
```

- Public, no authentication required
- Only PvP battle reports are available — Dark Force / NPC guardian attacks are **not** on the CDN (404)
- Size: 200–650KB depending on player detail and process events
- `first4` = first 4 digits of the reportId string

---

## 2. Top-Level JSON Structure

```json
{
  "reportId": "4605240496314802177",
  "logtime": 1775319323,           // Unix timestamp of battle
  "battle": { ... }                // Full battle data (see below)
  // "newFight" may appear in some reports (unconfirmed purpose)
}
```

---

## 3. battle Root Fields

```json
{
  "result": 1,          // 1=attacker wins, 2=defender wins
  "ver": 1,             // Report format version
  "stage": 1,           // Battle stage (always 1 in observed reports)
  "fightType": 114,     // Battle type (see §14)
  "reportComparison": 1, // Whether Stats Comparison tab is available (1=yes)
  "extraFlag": { "putOnArmyV2": 1 }, // Feature flags; putOnArmyV2=1 means attPutArmyV2 is present
  "attacker": { ... },  // Attacker march data
  "defender": { ... },  // Defender march data
  "fightMarches": [ ... ], // Outcome data (always 1 item in observed reports)
  "process": [ ... ],   // Round-by-round event log
  "etReward": [ ... ]   // Rewards earned
}
```

---

## 4. attacker / defender (March Level)

These are the march-level containers. Most detailed data is inside `players[0]`.

```json
{
  "fightTeamId": 2,          // Which team (1=defender, 2=attacker in PvP; NPC varies)
  "marchId": 2,              // March ID (matches fightMarches attMarchId/defMarchId)
  "uid": XXXXXXX272,      // Player UID
  "career": 72001,           // Career/class ID
  "stamp": 1008,             // Framework stamp (internal)
  "alTag": "",               // Alliance tag
  "headFrameId": 0,          // Avatar frame ID
  "isolated": 0,             // Whether isolated (solo vs alliance rally)
  "attackPointNum": 9,       // Number of formation attack points used
  "hasEffectMecha": 1,       // Whether mecha is deployed (1=yes)
  "armyUuids": [ ... ],      // All individual troop UUIDs (404 items = 404 troops)
  "effectBuffs": [ ... ],    // Same as players[0].effectBuffs (aggregated buff pool)
  "traceEffectBuffs": [ ... ],// Same as players[0].traceEffectBuffs (per-source)
  "elementLevels": [ ... ],  // Same as players[0].elementLevels
  "allStandings": { ... },   // Same as players[0].allStandings
  "buffs": [ ... ],          // Raw buffs (small list; see §5.15)
  "units": [ ... ],          // Formation unit slots (see §6)
  "mechas": [ ... ],         // Mecha data (older format; see §7)
  "mechas2": { ... },        // Mecha data (newer keyed format; see §7)
  "players": [ ... ]         // Detailed player data (always 1 item in solo; multiple in rally)
}
```

> **Note:** `effectBuffs`, `traceEffectBuffs`, `elementLevels`, `allStandings`, `armyUuids` appear at BOTH the march level and inside `players[0]`. They are the same data. Use `players[0]` as the canonical source for detailed data.

---

## 5. players[0] — Full Player Detail

The richest data source. In rally battles this array has multiple players; in solo it has exactly 1.

```json
{
  "uid": 7550807444272,
  "career": 72001,
  "playerInfo": "{...JSON string...}",  // Name, avatar URLs, nationalflag, gender
  "stamp": 1008,
  "fightTeamId": 2,
  "marchId": 2,
  "isolated": 0,
  "attackPointNum": 9,
  "hasEffectMecha": 1,
  "effectBuffs": [ ... ],
  "traceEffectBuffs": [ ... ],
  "heroList": [ ... ],
  "heroEquips": null,         // ALWAYS null at this level — gear is inside heroList[].heroEquips
  "effectDecorations": { ... },
  "effectSkins": { ... },
  "enigmas": { ... },
  "formationV2": { ... },
  "armyEquips": [ ... ],
  "newArmyEquips": { ... },
  "CTCs": { ... },
  "elementLevels": [ ... ],
  "armyMastery": { ... },
  "allStandings": { ... },
  "armyUuids": [ ... ],
  "buffs": [ ... ],
  "traps": [ ... ]
}
```

### playerInfo (JSON string, not object)
```json
{
  "username": "Tex",
  "nickname": "",
  "nationalflag": 233,         // Int → ISO country code (see reference_nationalflag_mapping.md)
  "gender": 0,
  "usergender": 1,
  "headimgurl": null,          // In-game avatar URL (may be null)
  "headimgurl_custom": "https://knight-cdn.akamaized.net/headimg/{hash}.jpg",
  "avatarurl": null,
  "moreHeadImgs": [ "https://knight-cdn.akamaized.net/..." ],
  "cn": 0                      // Suspected: Chinese server flag
}
```

---

### 5.1 effectBuffs — Aggregated Buff Pool

Array of all buffs active during the battle, after grouping/redirect logic.

```json
{ "id": 930001, "val": 256300 }          // Standard buff: id + val
{ "bufid": 921006, "value": 7000, "specialType": 1, "data": ["10112"] }  // Special buff
```

- `val` / `value`: raw value where **100 = 1%** (so 256300 = 2563% Army ATK)
- Values are in per-10,000 units (basis points ÷ 100)
- Total across all effectBuffs in observed reports: ~2.9–3.1M (≈ 290–310% across all stat types combined if summed naively, but these are different stat categories not one pool)
- `specialType=1` with `data` array: conditional/triggered buff referencing specific unit UUIDs
- Some IDs are "redirect parents" — the game groups children under them (e.g., 930000 groups 930001+930002+930003)
- **24 buff IDs are silently skipped** by game's grouping logic (not in EFFECTBUFF table): 520130-133, 520141, 520143, 932001-032029, 933007-008, 936013, 950011

### 5.2 traceEffectBuffs — Per-Source Breakdown

```json
{ "id": 930001, "val": 48700, "s": 25 }
```

- Same buff IDs as effectBuffs, but split by source (`s` field)
- `s` = `type_small` source category (see §13 for full map)
- Sum of all `val` across traceEffectBuffs = sum of all effectBuffs val (they're the same data, different presentation)
- Total entries: 238–363 per player (varies by source diversity)
- **Key use:** drill into "how much of my ATK comes from decorations vs titan gear vs research"

---

### 5.3 heroList — Heroes in March

```json
{
  "id": 332,           // heroId
  "level": 120,        // Hero level (max 120)
  "star": 5,           // Star rating
  "war": 50234,        // ATK stat
  "def": 21481,        // DEF stat
  "command": 2500,     // Command (march size contribution)
  "nonActiveSkills": [
    { "skillId": 10332, "level": 0 },   // ← native/inherent passive (skillId = 10000+heroId)
    { "skillId": 20743, "level": 7 },   // exclusive skill (see §18 gotcha)
    { "skillId": 21119, "level": 8 },   // equipped skill 1
    ...                                  // up to 5 total equipped skills
  ],
  "heroEquips": [ ... ],  // Titan Gear (see §5.4)
  "talents": [
    { "talentId": 8, "ttsr": 10000 }    // Talent perk; ttsr = value (10000 = max)
  ]
}
```

**⚠️ Critical gotcha:** `nonActiveSkills[0].level` is **always 0** in battle reports for all heroes. The game does not send the actual exclusive skill level. We can show WHICH skill is equipped (name + icon) but NOT the level.

**Exclusive skill identification:**  
`skillId = 10000 + heroId` → this is the **native passive** (always level 0, ignore)  
`skillId = EXCL_SKILL[heroId].id` → this is the **actual exclusive skill**

---

### 5.4 heroEquips — Titan Gear

6 slots per hero (inside `heroList[].heroEquips`, NOT at `players[0].heroEquips` which is null).

```json
{
  "equipId": 100104,         // slot*100000 + quality (quality 4 = max/gold)
  "level": 20,               // Enhancement level (max 20)
  "heroId": 332,
  "landScore": 27000,        // CP contribution for Army branch
  "navyScore": 27000,        // CP contribution for Navy branch
  "airScore": 43632,         // CP contribution for Air Force branch
  "enhanceValue": { "930100": 2158 },  // buffId → enhancement bonus (raw ÷ 100 = %)
  "infos": [
    { "type": 1, "templateId": 9,      "buffValue": 330 },  // random stat roll
    { "type": 1, "templateId": 9,      "buffValue": 438 },  // same stat, different roll
    { "type": 1, "templateId": 8,      "buffValue": 420 },  // different stat
    { "type": 2, "templateId": 102021, "buffValue": 0 }     // rune slot
  ]
}
```

- `type=1`: random stat. `templateId` maps to a buff category (53 known templates from all-hero-gear.json)
- `type=2`: rune/special effect. `templateId` format = `{slot}{runeCode}` e.g. `102021` = slot 1, rune code 2021
- `buffValue ÷ 100` = percentage contribution
- Roll quality: `buffValue / GEAR_TEMPLATE[tid].maxRaw * 100` = roll percentage
- **Star level NOT in report** — infer from rune code or cross-reference armory data
- `landScore/navyScore/airScore` are per-slot CP by branch (NOT cumulative)

---

### 5.5 effectDecorations — Decorations

```json
{
  "ids": [ 52995, 47782, ... ],          // All placed decoration IDs (up to ~800)
  "sublimes": [                           // Upgraded decorations with buff contributions
    { "id": 50178, "buffs": [{ "buffId": 930100, "value": 0 }] }
    // Note: value here is the "Advanced Upgrade" bonus ONLY — often 0
    // Base decoration buff must be looked up from decoration-index.json by group
  ],
  "sublimeExps": [                        // Sublime experience entries
    { "val": 200, "id": 980204 }
  ],
  "suitLevelBuff": { "1201": 1, ... },    // Decoration suit set → active bonus level
  "suit": [ 1216, 1217, ... ],            // Active suit set IDs (22 in observed reports)
  "boxMaxIds": [ 4632002, ... ]           // Max-level decoration IDs (501 in observed reports)
}
```

**⚠️ Key gotcha:** `sublimes[].buffs[].value` is the small "Advanced Upgrade" bonus, NOT the decoration's base buff. Base buffs come from `decoration-index.json` lookup by group ID.

---

### 5.6 effectSkins — Base Skins

```json
{
  "skinId1": { "buffId": 930001, "val": 30000 },
  "citySkill1": { "buffId": 9990018, "val": 4560 },
  ...
}
```

- Keys are string indices (skinId1, skinId2..., citySkill1, citySkill2...)
- Base skin buffs map to standard buff IDs (930xxx)
- City skills use 9990xxx buff IDs
- Source in traceEffectBuffs: `s=4` (CPNT)
- **⚠️ Known limitation:** Base Skin Skill (type_big 88) sub-breakdown may show 0% for opponents — effectSkins pipeline for opponents has a separate code path not fully mapped

---

### 5.7 enigmas — Enigma Beasts

```json
{
  "beastDatas": [
    {
      "power": 180120,          // Beast CP
      "beastId": 1,             // Beast type ID (1-9)
      "element": 1,             // Element (1=Fire, 2=Wind, 3=Water, 4=Ground)
      "level": 120,
      "fieldType": 1,           // Field slot (1=CPNT Mastery, 2=Formation, 3=Equip, 4=Defense, 5=Offense)
      "beastSkills": [ ... ],   // Skill IDs and levels
      "attrs": [ ... ]          // Field attribute bonuses
    }
  ],
  "fieldAttrs": { ... }         // Aggregate field attribute bonuses
}
```

- Source in traceEffectBuffs: `s=106` and `s=107` (both map to type_big "Enigma Beast")
- Beast buff IDs in 520xxx range (fully mapped — see reference_enigma_beasts.md)
- Beast power values always use 16000 as universal max potential denominator

---

### 5.8 formationV2 — Formation

```json
{
  "formationId": 1,           // Formation type
  "slots": [ ... ],           // Formation slot assignments
  "activeSkills": [ ... ],    // Active formation skills
  "passiveSkills": [ ... ],   // Passive formation skills
  "level": 5,
  ...
}
```

- Formation contributes to `s=32` and `s=41` in traceEffectBuffs
- Scorpion formation skill 941215 → Counterblow (buff ID 100008) in skill damage

---

### 5.9 armyEquips / newArmyEquips — HT Chips

**`armyEquips[]`** (older format):
```json
{ "itemId": 403100, "pos": 1, "armyType": 101 }
// armyType: 101=Army, 201=Navy, 301=Air Force
// pos: slot position (1-6)
// itemId: 0 = empty slot
```

**`newArmyEquips.equips[]`** (newer format, also in `fightMarches`):
```json
{ "eid": 910039, "pos": 30101, "suit": 9, "id": 3559869806198284297 }
// pos format: {armyType}{slot:02d} e.g. 30101 = Air Force, slot 01
// eid: equipment template ID (910xxx range)
// suit: chip set/suit index
```

> **Status:** HT chip display is deferred. Structure is documented but not rendered in battle-report.html viewer.

---

### 5.10 CTCs — Collectible HT Chips

```json
{
  "themes": [1, 2, 3],
  "collects": [
    { "id": 1,  "lv": 2 },
    { "id": 12, "lv": 5 },
    ...   // up to ~38 entries
  ]
}
```

- `themes`: active theme IDs
- `collects`: chip ID → level (1-5), ~38 collectible chips observed
- Distinct from `armyEquips`/`newArmyEquips` (which are equipped chips, not collectibles)

---

### 5.11 elementLevels — Element Levels

```json
[ { "type": 1, "level": 5 }, { "type": 2, "level": 3 }, { "type": 4, "level": 4 } ]
```

- Types: 1=Fire, 2=Wind, 3=Water, 4=Ground (same as Enigma Beast element IDs)
- Levels correspond to player's element progression

---

### 5.12 armyMastery — Army Mastery

```json
{ "101": 121, "201": 219, "301": 321 }
```

- Keys: 101=Army, 201=Navy, 301=Air Force
- Values: mastery level (contributes to `s=17` in traceEffectBuffs)

---

### 5.13 allStandings — Battle Outcome

```json
{
  "total": 404,       // Total troops at battle start
  "survival": 225,    // Troops surviving
  "dead": 179,        // Troops killed
  "wounded": 0        // Troops wounded (non-zero in some battle types)
}
```

---

### 5.14 armyUuids — Individual Troop UUIDs

Array of 404 string UUIDs (one per troop). Format varies by battle:
- PvP: short sequential strings (`"10001"`, `"10048"`, `"10112"`)
- Wasteland/NPC: long numeric IDs (`"4430208425366817797"`)

Troop UUIDs cross-reference to `units[].uuid` (the formation slot the troop belongs to).

---

### 5.15 buffs — Raw Buffs

Small array of special/conditional buffs not captured in effectBuffs:
```json
[{ "specialType": 1, "data": ["10112"], "bufid": 921006, "value": 7000 }]
```

- `specialType=1`: conditional buff tied to specific unit UUID in `data`
- `bufid=921006`: "Golden Tank coming" (known ID from buff mapping)

---

## 6. units[] — Unit Formation Slots

One entry per hero formation slot (up to 9 slots observed).

```json
{
  "uid": 7550807444272,           // Player UID
  "uuid": "10112",                // Formation slot UUID (matches armyUuids entries)
  "armyId": 901103,               // Troop type ID (e.g. 901103=Valhalla Mecha, 505111=specific mecha troop)
  "maxPower": 204483073614300.0,  // Raw engine power for this slot (used in physics, NOT displayed CP)
  "maxElementPower": 27540000.0,  // Element power contribution
  "maxShield": 260265609623744.0, // Max shield value
  "isExtra": 0,                   // Whether this is an "extra" slot
  "isDead": 0                     // Whether this unit died
}
```

- `maxPower` values are enormous (10^14 range) — these are internal physics values, NOT the CP shown in the report
- `attMaxPower` in fightMarches = sum of all unit `maxPower` values
- `armyId` prefix convention: 9xxxxx = Valhalla/merged, 5xxxxx = mecha, 1/2/3xxxxx = base troops

---

## 7. mechas / mechas2 — Mecha Data

Two formats exist simultaneously (legacy + new).

```json
// mechas[] — array
{
  "mechaId": 1005,          // Mecha type ID (1002-1008 observed)
  "uuid": "10368",          // Which formation slot this mecha occupies
  "skinId": 0,              // Skin applied (0 = base)
  "skillIds": [100002, 104001, 104002, ...],  // 7 mecha skill IDs
  "parts": [34, 35, 36, ...],  // 9 mecha part IDs
  "chips": [
    { "chipId": 2500, "level": 0,  "rndAttrs": "32;2705|29;2638" },   // slot 0
    { "chipId": 2501, "level": 25, "rndAttrs": "8;688|10;499|4;1216|5;818" }, // slot 1
    ...   // 7 chip slots total
  ]
}

// mechas2 — keyed by fightTeamId string
{ "2": [ { ...same structure... } ] }
```

**chips.rndAttrs format:** `"attrType;value|attrType;value"` — pipe-separated attribute/value pairs  
**chipId range:** 2500-2506 for one mecha type, 6500-6506 for another  
**parts:** 9 component part IDs (body segments of the mecha)  
**skillIds[0]=100002:** always the base mecha attack skill; slots 1-6 are equipped skills  

> **Status:** Mecha chip display is deferred. Full attrType→stat mapping not yet extracted.

---

## 8. fightMarches[0] — Battle Outcome Data

### 8.1 Combat Power (CP)

| Field | Example | Description |
|-------|---------|-------------|
| `attBattlePowerVal` | 201,082,447 | Attacker's pre-battle CP (what game displays) |
| `defBattlePowerVal` | 224,970,647 | Defender's pre-battle CP |
| `attBattlePower` | "201M" | Human-readable string |
| `defBattlePower` | "225M" | Human-readable string |
| `attMaxPower` | 1.103×10¹⁵ | Sum of all unit `maxPower` (raw physics) |
| `defMaxPower` | 1.328×10¹⁵ | Same for defender |
| `attPower` | 6.169×10¹⁴ | Attacker's surviving power after battle |
| `defPower` | 0.0 | Defender's surviving power (0 = wiped) |

See §11 for full CP calculation analysis.

### 8.2 Army Composition (attPutArmyV2)

```json
[{ "armyId": 901103, "s1": 44 }, { "armyId": 901103, "s0": 56 }]
```

- `s0` = troops that survived, `s1` = troops that died/wounded
- `armyId` = troop type ID
- `attPutArmy` is the legacy format (less detailed)

### 8.3 Skill Damage (attSkillHurts)

```json
{
  "total_hurt": 28399440,
  "common_hurt": 18243200,
  "common_count": 404,
  "skill_hurts": {
    "10332": 4823040,    // skillId → total damage dealt
    "10169": 3421200
  },
  "skill_hurt_count": {
    "10332": 6,          // skillId → number of activations
    "10169": 3
  }
}
```

- `common_hurt`: standard troop damage (non-skill)
- `skill_hurts`: per-hero-skill damage totals
- `skill_hurt_count`: how many times each skill fired
- The `effect_buff_hurts` sub-object (if present) breaks down special effects like Magnetic Pulse, Ignition, Counterblow

### 8.4 Traps

```json
"attTraps": [],    // Attacker traps (usually empty)
"defTraps": []     // Defender traps deployed
```

- Present even when empty; format not fully characterized

### 8.5 Special Buffs

```json
"attSpecialBuff": { ... },
"defSpecialBuff": { ... }
```

- Contents not fully mapped; suspected to hold in-battle conditional effects

---

## 9. process[] — Round-by-Round Events

The full battle simulation log. 438–3350 events observed depending on battle length.

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

**Known `t` event types** (from observed reports):

| t= | Suspected Meaning |
|----|------------------|
| 1 | Troop kill event |
| 2 | Damage dealt |
| 3 | Skill triggered |
| 4 | Battle start / metadata |
| 5 | Shield/HP event |
| 6 | Buff applied |
| 7 | Debuff applied |
| 9 | Formation effect |
| 11 | Mecha action |
| 14 | Skill damage tick (effect_buff) |
| 19 | Round end |
| 25 | Special effect |
| 27 | Element effect |
| 30 | Attack event |
| 31 | Round start |
| 32 | Troop update |
| 33 | Status change |
| 35 | Heal/recovery |
| 37 | Resource event |

**Timeline (`tl`):** increments by 1200ms per round. `tl=0` = pre-battle init, `tl=1800` = round 1, `tl=3000` = round 2, etc. (gap between 0 and 1800 is setup phase). Max tl in observed 28-round battle: ~23,400ms.

> **⚠️ Largely unmapped in terms of individual event types.** However, the role of `process[]` is now confirmed — see §17.

**CONFIRMED: `process[]` IS the replay. There is no video endpoint.** When the in-game Replay button is clicked, `NMailBattlePanel.onReplayClick()` calls `h.replay(mailData)` passing the full report data (including `process[]`) to `NBattleMainComponent`, which renders the battle live using the game engine. `BattleReplayview` drives round navigation (`_currentRound`/`_currentRoundMax`). The only network traffic triggered is sprite atlas / prefab asset fetches for troop models — no MP4, WebM, or separate CDN video URL exists. The replay is fully self-contained in the CDN JSON.

---

## 10. etReward — Battle Rewards

```json
[
  { "count": 1, "id": 69159 },    // Item ID + quantity dropped
  { "count": 1, "id": 2300003 },
  { "count": 1, "id": 820016 }
]
```

- Present only on attacker side (winner gets rewards)
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

Where `buff_multiplier` applies only a **subset** of buffs — the permanent, peacetime buffs.

**Observed correlation:**
```
attMaxPower / attBattlePowerVal ≈ 5,373,000–5,487,000
```
This ratio is consistent across reports (within ~2%), suggesting `maxPower` and CP are computed from the same base stats scaled differently.

**Per-troop average CP observed:**
- Report 1 (PvP, fightType=114): 201,082,447 ÷ 404 troops = **~497,729 CP/troop**
- Report 2 (Guardian, fightType=126): 192,229,892 ÷ 404 troops = **~475,817 CP/troop**

The ~22K difference between same-troop-count marches is likely from different troop tier/rank composition (armyId differs: 901103 vs 905103).

**Why effectBuffs don't "add up" to CP:**
- `effectBuffs` total ≈ 3,048,107 raw units = **30,481% combined across all stat types**
- This is NOT a single percentage applied to CP
- Different buff IDs apply to different stats (ATK, HP, DEF, Lethality, etc.)
- CP uses only a subset — permanent research/tech/base buffs
- Wasteland buffs (s=112/134/135) are applied in-combat but excluded from the displayed CP number

---

## 12. Wasteland / Territory Buffs and CP

**Yes — Wasteland Effect buffs are applied in combat but do NOT count toward `attBattlePowerVal`.**

| Source (`s=`) | Category | Counted in CP? |
|---------------|----------|----------------|
| 112 | Wasteland Effect | **NO** |
| 134 | Wasteland Airship Effect | **NO** |
| 135 | Wasteland Effect (variant) | **NO** |
| 65 | Unknown — fightType=126 only; likely NPC/guardian bonus | **NO (suspected)** |
| 2–107 (non-wasteland) | Research, Gear, Heroes, Decorations, etc. | **YES** |

**In report 4605144673410904064 (fightType=126, Wasteland Guardian):**
- Wasteland buffs (s=112+134+135): **261,400** raw units (8.6% of total buff pool)
- `s=135` adds `id=992578 val=50000` — a large HP-type buff not in the PvP report
- `s=65` (unknown, 19,000 val): only appears in this wasteland-type report — possibly NPC battle bonus

**The HP column in stats comparison shows very high values in wasteland battles** because s=112 buff IDs like `992576` (108,000 = 1080% HP) and `992575` (54,000 = 540% Infantry ATK) inflate the HP/ATK totals shown in the breakdown, but these are excluded from the CP figure.

---

## 13. Buff Source Reference (traceEffectBuffs s= values)

The `s` field = `type_small`. Multiple `type_small` values can map to one `type_big` display category.

| s= | type_big | Display Name | Notes |
|----|----------|--------------|-------|
| 1 | 1 | Unknown(1) | Rare; small values |
| 2 | 2 | Hero Skills | Hero passive/active skill buffs |
| 3 | 3 | Hero Perk | Technology/research perks |
| 4 | 4 | CPNT / Base Skin | Base castle skin buffs |
| 5 | 5 | Suppress | Suppress-type buffs |
| 6 | 6 | Alliance Tech | Alliance technology |
| 7 | 7 | Decoration | Museum/decoration buffs |
| 8 | 8 | VIP Privileges | VIP level benefits |
| 9 | 9 | Class Talent | Career/class talent tree |
| 12 | ? | Unknown(12) | Contains 930100, 930200, 1101067; suspected academy or specific research |
| 13 | 13 | Alliance Badge | Alliance badge bonuses |
| 16 | 16 | Hero Bond | Hero bond/affinity bonuses |
| 17 | 17 | Mastery | Army mastery bonuses |
| 18 | 18 | Unknown(18) | Small values; possibly hero bond variant |
| 24 | 24 | Unknown(24) | Contains 1101042, 1131001; suspected specific research track |
| 25 | 25 | Base Effect | Titan Gear system base buffs |
| 26 | 26 | Unit Equipment | HT Chips / Mecha equipment |
| 27 | 27 | Heavy Trooper | HT-specific bonuses |
| 31 | 31 | Odinium Tech | Odinium technology tree |
| 32 | 32 | Formations | Formation bonuses |
| 33 | 33 | Secret Weapon | Secret weapon bonuses |
| 34 | ? | Unknown(34) | Rare; small values |
| 37 | 37 | Titan Gear | Titan Gear stat contributions |
| 41 | 32 | Formations (alt) | Maps to same Formations type_big |
| 44 | 100 | Loot Quest | Loot Quest battle bonuses |
| 64 | 64 | Unknown(64) | Contains 990117/118/121; 2 reports observed |
| 65 | ? | Unknown(65) | Only in fightType=126 (Wasteland Guardian); suspected NPC bonus |
| 75 | 25 | Base Effect (75) | Groups into Base Effect |
| 76 | 25 | Base Effect (76) | Groups into Base Effect |
| 88 | 88 | Base Skin Skill | City skill buffs (9990xxx IDs) |
| 95 | 95 | Ruin Bonus | Ruin/battlefield bonus |
| 98 | 98 | Official Position | Alliance officer role bonuses |
| 104 | ? | Unknown(104) | Contains 990145; 1 report observed |
| 106 | 106 | Enigma Beast | Beast field bonuses |
| 107 | 106 | Enigma Beast (107) | Maps to same Enigma Beast type_big |
| 112 | 135 | Wasteland Effect | Territory buffs — NOT in CP |
| 134 | 134 | Wasteland Airship | Airship territory buffs — NOT in CP |
| 135 | 135 | Wasteland Effect (var) | Territory variant — NOT in CP |

---

## 14. fightType Reference

| fightType | Name | Notes |
|-----------|------|-------|
| 21 | Time Clash Arena / SVS Warzone | PvP |
| 86 | Loot Quest Battle | PvP with loot quest modifier |
| 114 | Standard PvP | Most common; observed in report 4605240496314802177 |
| 126 | Attack on Warzone Guardian | NPC wasteland guardian; report 4605144673410904064 |
| 127 | Unknown | Not yet observed |

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
| Unit icons | `assets/unit-icons/` | Repo | 5 only (Army only); KongJun/HaiJun missing |
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
- HT chip display (`armyEquips` / `newArmyEquips` / `CTCs`)
- Battle rewards (`etReward`)
- Trap data (`attTraps` / `defTraps`)
- Mecha skill details beyond icon
- Process/replay (see §17)
- Rally battles (multiple players in march)

### enigma-viewer.html
Referenced via `?id=` passthrough. Shows enigma beast comparison.

### decor-index.html
Referenced via `?id=` passthrough. Shows decoration details.

---

## 17. Unmapped / Unknown Areas

### High Priority (suspected valuable data)

| Area | Location | Status | Notes |
|------|----------|--------|-------|
| **process[] event types** | `battle.process[]` | Unmapped | 3,350 events in a long PvP; used for in-game replay animation. t=1-37 types partially guessed. `tl` is millisecond timeline. |
| **Battle replay video** | `process[]` in CDN JSON | **CONFIRMED — no video endpoint** | Replay is client-side only. `NBattleMainComponent` + `BattleReplayview` render the battle live from `process[]`. No MP4/WebM/M3U8 exists. Only asset bundle fetches triggered (sprite atlases for troop models). |
| **attSpecialBuff / defSpecialBuff** | `fightMarches[0]` | Empty in observations | May hold conditional in-battle buffs not in effectBuffs |
| **defTraps / attTraps** | `fightMarches[0]` | Empty in observations | Format unknown; likely array of trap items |
| **mecha.chips rndAttrs attrType** | `mechas[].chips[].rndAttrs` | Partially mapped | Format: `"attrType;value|..."` — attrType integers not mapped to stat names |
| **mecha parts** | `mechas[].parts[]` | Unmapped | 9 integer IDs per mecha; likely body component IDs |
| **newArmyEquips / armyEquips** | `players[0]` | Documented, not rendered | HT chip item IDs need TABLE lookup for names/icons |
| **CTCs** | `players[0].CTCs` | Documented, not rendered | Collectible chip IDs and levels |

### Medium Priority (known but incomplete)

| Area | Location | Status | Notes |
|------|----------|--------|-------|
| **s=12 source** | traceEffectBuffs | Unconfirmed | Contains 930100 (Infantry HP), 930200 (Infantry ATK), 1101067. Possibly a specific academy/research track. |
| **s=24 source** | traceEffectBuffs | Unconfirmed | Contains 1101042, 1131001. Small values (~6500). |
| **s=65 source** | traceEffectBuffs | Only in fightType=126 | Buff IDs 990549/550, 990191, 992171/172. Likely NPC bonus active during Wasteland Guardian battles only. |
| **s=104 source** | traceEffectBuffs | Rare | Contains 990145, 3190 val. Only seen in 1 report. |
| **s=88 (Base Skin Skill)** | traceEffectBuffs | Partial | 9990xxx IDs known. Opponent sub-breakdown shows 0% due to separate effectSkins code path. |
| **992xxx buff IDs** | Wasteland Effect (s=112) | Partially named | 992575=Infantry ATK (wasteland), 992576=HP (wasteland), 992577/578=Cavalry variants. Others unmapped. |
| **Unit armyId mapping** | `units[].armyId` | Partial | 901103=Valhalla (Army offense), 905103=Valhalla variant, 505111/506371=mecha troops. Full table not extracted. |
| **BUFF_GROUP coverage** | `battle-report.html` | ~168 entries | New reports may contain unmapped IDs. Run AddOneBuffToBuffPool oracle to expand. |

### Low Priority / Low Confidence

| Area | Status | Notes |
|------|--------|-------|
| `extraFlag.putOnArmyV2` | Confirmed meaning | When 1, `attPutArmyV2` is present (vs legacy `attPutArmy`). |
| `stage` field | Always 1 | May indicate multi-stage battle possibility in other fightTypes. |
| `reportComparison` | Always 1 in observed | Possibly 0 for older reports without Stats Comparison data. |
| `ver` field | Always 1 | Report format version; future-proofing. |
| `isolated` | Always 0 in observations | Suspected: whether march was isolated from alliance rally. |
| `headFrameId` | Always 0 | Avatar frame — may show cosmetic frame for premium players. |
| `cn` in playerInfo | Always 0 | Suspected: Chinese server player flag. |
| `attOverRoundNum / defOverRoundNum` | Always False | Whether battle exceeded max rounds. |
| `newFight` key | Missing in both observed reports | Unknown purpose; may appear in specific battle types. |

---

## 18. Key Gotchas

1. **`nonActiveSkills[0]` is NOT the exclusive skill.** The entry where `skillId = 10000 + heroId` is the native passive (always level 0, ignore it). Find the exclusive skill via `EXCL_SKILL[heroId].id`. Exclusive skill levels are always 0 in CDN reports — the server does not include them.

2. **`effectBuffs` values are in per-10,000 units (basis points × 100).** Value of 100 = 1%. Total effectBuffs sum across all stat types ≈ 3M = ~30,000% if naively summed, but this is misleading because each ID applies to a different stat.

3. **Combat Power (`attBattlePowerVal`) ≠ effectBuffs arithmetic.** CP is pre-computed server-side from troop base stats × permanent buff multiplier. Wasteland buffs (s=112/134/135) appear in effectBuffs but are excluded from CP. The `effectBuffs` pool is what's applied during combat simulation.

4. **`sublimes[].buffs[].value` is the Advanced Upgrade bonus, NOT the base decoration buff.** Often 0. Base buff values come from `decoration-index.json` lookup by decoration group ID.

5. **`heroEquips` at `players[0].heroEquips` is always null.** Titan Gear is inside `heroList[].heroEquips`. Don't look for it at the player level.

6. **`armyUuids` format changes by battle type.** PvP uses short sequential strings (`"10001"`), wasteland/NPC uses long numeric UUIDs. Both cross-reference to `units[].uuid`.

7. **Number formatting:** K/M/B/T then aa/bb/cc/dd/ee/ff/gg for powers beyond trillion. NOT Q.

8. **Dark Force / NPC attack reports are NOT on the public CDN.** Only PvP and player-initiated wasteland battles appear. NPC-initiated attacks return 404.

9. **`maxPower` in units[] is raw physics engine power (10¹² - 10¹⁵ range), NOT displayed CP.** The relationship: `attMaxPower / attBattlePowerVal ≈ 5.4M` (consistent ratio, same base stat derivation, different scaling).

10. **`mechas[]` and `mechas2{}` duplicate the same data** in two formats (legacy array vs keyed by fightTeamId). `mechas2` is the canonical format.

11. **`traceEffectBuffs` and `effectBuffs` sum to the same total.** They're the same buff pool, differently presented. traceEffectBuffs splits by source (`s=`); effectBuffs merges all sources per buff ID.

12. **BUFF_GROUP may have unmapped IDs in new reports.** Run the `AddOneBuffToBuffPool` oracle on any new report before adding stat comparison logic. Unmapped IDs are silently dropped in the viewer.

13. **fightType=126 (Wasteland Guardian) adds unique buff sources** not present in PvP: s=65 (NPC bonus, ~19K), s=135 (wasteland variant). CP is still calculated the same way — these extras inflate the in-battle stats shown in comparison but not the CP figure.

14. **`etReward` (battle rewards) only appears on the winning side's report.** Loser has no etReward. Currently not rendered in the viewer.

15. **`attPower` (surviving power) is the attacker's remaining engine power after battle**, NOT the CP of survivors. It relates to `attMaxPower` (pre-battle), not `attBattlePowerVal` (displayed CP). `defPower: 0.0` means the defender was completely wiped.
