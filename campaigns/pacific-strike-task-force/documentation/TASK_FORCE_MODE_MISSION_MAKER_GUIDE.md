# Task Force Mode Campaign Author Guide

This guide describes the current Task Force Mode implementation used by `Pacific Strike 1985 - Task Force Mode`. It is a practical reference for campaign makers who want to build or modify linear campaigns that use a persistent player task force.

## Core Concept

Task Force Mode is campaign-side logic layered on top of the existing linear campaign and mission formats. Mission files remain normal mission INI files. At launch, the campaign reads the player task force save state, the player roster, and mission-specific settings, then writes a temporary generated mission INI with the player's selected vessels, submarines, aircraft pools, loadouts, crew skill, campaign tags, and spawn positions.

The authored mission provides anchors and normal mission context. Task Force Mode decides which player units replace those anchors.

Fresh-branch implementation note: the current rebuilt slice only adds the active linear campaign view bottom command bar with placeholder `Task Force Status`, `Task Force Builder`, and `Service Record` buttons. Free-event overlays now leave room above that bar. This UI-only step does not require any mission INI edits.

## Required Campaign Files

A Task Force Mode campaign needs:

- `campaign.ini`
- `player_task_force_roster.ini`
- normal mission INI files
- normal campaign art, briefing, and language content

Set Task Force Mode in `campaign.ini`:

```ini
[TaskForceMode]
Enabled=True
RosterFile=player_task_force_roster.ini
StartingPoints=50
PointCap=50
BasicShipLoadoutVariants=Default|Early
LockedShipLoadoutVariants=Late
BasicAircraftLoadouts=Default|Ferry|Empty|AirToAir|ASW|AntiShip|Strike|EW|AEW|Recon
LockedAircraftLoadouts=AirToAirIntercept|AntiShipHeavy|AntiShipPrecision|SEADAdvanced
DefaultTaskForceName=Task Group 77.3
TaskForceNameOptions=Task Group 77.3|Cruiser Striking Force|Cactus Striking Force
CrewSkillInitial=Trained
CrewSkillThresholds=Seasoned:1|Veterans:3|Ultra:5
UnitDecommissionPointReturnModifier=0.25
UnitDismissPointReturnModifier=0.5
UnitDestroyedPointReturnModifier=0
DamageToAllowRepair=Light,Moderate
DamageToDisallowRepair=Heavy
RepairPointsCost=Light,0.1|Moderate,0.25
```

If `Enabled=True`, the roster file is required. Missing or invalid rosters are campaign authoring errors.

Commander setup is campaign-authored in `commander_settings.ini`, referenced from `[TaskForceMode]`:

```ini
[TaskForceMode]
CommanderSettingsFile=commander_settings.ini
```

The commander settings file owns commander identity rules, same-nation discounts, navy display resources, and officer ranks:

```ini
[CommanderSettings]
CommanderNations=US|Japan|Australia
CommanderDefaultNation=US
CommanderNameDefaultUS=Claude Robinson
CommanderNameDefaultJapan=Hiroshi Watanabe
CommanderNameDefaultAustralia=Grant Morrison
CommanderNamePoolUS=Names_USA
CommanderNamePoolJapan=Names_Japan
CommanderNamePoolAustralia=Names_Australia
CommanderStartingRankLevel=5
SameNationUnitDiscount=0.2

NavyNameUS=United States Navy
NavyEmblemUS=ui/campaign/navy_emblems/usn_emblem.png
NavyNameJapan=Japan Maritime Self-Defense Force
NavyEmblemJapan=ui/campaign/navy_emblems/jmsdf_emblem.png
NavyNameAustralia=Royal Australian Navy
NavyEmblemAustralia=ui/campaign/navy_emblems/ran_emblem.png

[OfficerRanks]
US=Ensign,ENS,O-1,1,ui/campaign/officer_ranks/usa/insignia_ens.png|Commander,CDR,O-5,5,ui/campaign/officer_ranks/usa/insignia_cdr.png
Japan=Ensign,ENS,OF-1,1,ui/campaign/officer_ranks/japan/insignia_ens.png|Commander,CDR,OF-4,5,ui/campaign/officer_ranks/japan/insignia_cdr.png
Australia=Midshipman,MIDN,OF-D,0,ui/campaign/officer_ranks/australia/insignia_midn.png|Commander,CMDR,OF-4,5,ui/campaign/officer_ranks/australia/insignia_cdr.png
```

Each rank entry is `DisplayName,Abbreviation,Grade,RankLevel,ResourcePath`. `CommanderStartingRankLevel` resolves against the selected nation's rank entries, preferring an exact `RankLevel` match and falling back to the nearest lower rank when a navy skips a level. Rank insignia PNGs are stored under `Assets/Resources/ui/campaign/officer_ranks/<nation>/`.

## Localization Keys

Task Force Mode UI text belongs under `[TaskForceMode]` in each `language_*/ui.ini`. Inside that section, keys should be written without the `taskforcemode` prefix:

```ini
[TaskForceMode]
aircraft=Aircraft
aviation=Aviation
assignsquadron=Assign Squadron
tooltipcheckout=Commit this task force before launching the mission.
TaskForceBuilder=Task Force Builder
Submarines=Submarines
ConfirmTaskForceSelection=Confirm Task Force Selection
```

The runtime and XAML still resolve those strings as `taskforcemodeaircraft`, `taskforcemodeaviation`, and so on. Do not write source keys such as `taskforcemodeaircraft` inside `[TaskForceMode]`; the section loader will double-prefix them into `taskforcemodetaskforcemodeaircraft`.

The current Task Force Builder pass localizes all visible builder labels, tabs, buttons, overlays, tooltips, debug controls, and confirmation dialog text through this block. Non-English `language_*/ui.ini` files have machine-translated values and should be reviewed by native speakers before final release.

Service Record nation tiles read navy display names and emblem resource paths from `commander_settings.ini`, not from `ui.ini` or `campaign.ini`. Nation identifiers should match the raw nation keys used by unit metadata, such as `US`, while localized display names come from `language_*/nations.ini`.

## Player Roster Format

The player roster is intentionally compact. Costs are per ship/submarine or per aircraft. Display names, flags, profile images, classes, and squadron names are resolved from normal language and unit data files where possible.

```ini
[AllowedVessels]
usn_ddg_kidd=Variant2,Variant3,Variant4|25
usn_cg_ticonderoga=Variant4|65
usn_cv_kitty_hawk=Variant1,Variant2|150

[AllowedSubmarines]
usn_ssn_permit=Variant3,Variant10|24
rn_ss_oberon=Variant14,Variant15|14

[AllowedHelicopters]
usn_sh-2f=Squadron8,6|5
usn_sh-3h=Squadron1,2|7

[AllowedAircraft]
usn_f-14a=Squadron9,4|30
usn_e-2c=Squadron2,2|50
usn_p-3c=Squadron29,1,Squadron31,1|20

```

Vessel and submarine lines are:

```ini
type=VariantA,VariantB|cost
```

Roster entries may add an optional third pipe-separated field for a short gameplay/tactical description shown in the Task Force Builder Unit Details panel:

```ini
type=VariantA,VariantB|cost|Short tactical description for players.
```

Aircraft and helicopter lines are:

```ini
type=SquadronA,count,SquadronB,count|cost_per_aircraft
```

Aircraft and helicopter entries use the same optional third description field:

```ini
type=SquadronA,count,SquadronB,count|cost_per_aircraft|Short tactical description for players.
```

Aircraft and helicopters are purchased from squadron pools one aircraft at a time. The count after each squadron reference is the total number of aircraft available from that squadron pool. The cost is the cost for one aircraft from that pool.

When multiple squadrons use the same aircraft type, put them on the same type line. For example, this keeps Australian and Japanese P-3C squadrons grouped under one P-3C aircraft type in the builder:

```ini
usn_p-3c=Squadron29,1,Squadron31,1|20
```

Any surface ship in the task force can serve as flagship. The first purchased ship becomes the default flagship, and players can assign a different ship from Task Force Status.

## Mission Anchors

Surface task force units spawn from the authored `Taskforce1Vessel1` anchor. Mark that unit with:

```ini
TaskForceModeAnchor=True
```

Task Force Mode uses this unit's position, heading, and `Telegraph` value. If the player has edited formation offsets, generated vessels use those offsets around the anchor. If no formation has been edited, generated vessels use a simple fallback formation.

Submarines are not part of the player campaign task force. Submarine side missions should omit `TaskForceModeMissionGenerationType` and launch the authored mission as-is with the pre-placed player submarine that was balanced in the mission file.

## Pre-Placed Player Units

For generated Task Force missions, authored player units can be preserved only when explicitly marked as not joining the generated task force. Otherwise, Task Force Mode removes authored player force sections and writes generated sections into the temporary mission INI. Missions without `TaskForceModeMissionGenerationType` launch authored content unchanged.

The cleanest mission structure is:

- one surface task force anchor
- optional submarine anchor
- optional campaign airfield
- enemy and neutral forces authored normally

## Points And Progression

Mission rewards live in each mission section of `campaign.ini`:

```ini
[Mission3]
TaskForceCompletionPointReward=8
TaskForceCompletionPointCapIncrease=5
```

Reward points add spendable command points. Cap increases raise the maximum force size the player can manage. Missions can also grant cap without points, which is useful when command gives the player authority for a larger force before new assets actually arrive.

Loadout rewards can be granted or removed after mission completion:

```ini
TaskForceLoadoutsToUnlock=AntiShipHeavy|AntiShipPrecision|Late
TaskForceLoadoutsToLock=
```

Baseline loadouts from `[TaskForceMode]` remain available. Locked loadouts are shown with a lock icon in the task force UI until unlocked.

## Repair, Rearm, And Air Group Reset

Mission-level logistics flags apply when that mission is prepared:

```ini
TaskForceModeRearm=True
TaskForceModeRepair=True
```

`TaskForceModeRearm=True` automatically replenishes ship and submarine ordnance at launch. `TaskForceModeRepair=True` means repair facilities are available before the mission; repairs still cost task force points and are performed from Task Force Status.

Damaged units are shown in the task force builder. Current repair, decommission, and dismissal behavior is controlled by:

```ini
UnitDecommissionPointReturnModifier=0.25
UnitDismissPointReturnModifier=0.5
UnitDestroyedPointReturnModifier=0
DamageToAllowRepair=Light,Moderate
DamageToDisallowRepair=Heavy
RepairPointsCost=Light,0.1|Moderate,0.25
```

Repairing Light or Moderate damage spends `[PersistentData] TaskForceAvailablePoints`, preserves ammunition state, and immediately clears the unit's saved damage state. Heavy damage cannot be repaired in theater. Decommissioning removes a sufficiently damaged unit and refunds the configured fraction of its cost. Dismissal removes a unit at any time and returns it to the roster for the configured fraction. Destroyed units are removed with the destroyed refund modifier.

## Surface Mission Restrictions

Surface side missions can require the player to pick one owned surface vessel:

```ini
TaskForceModeRequiredUnitType=Vessel
TaskForceModeMaxUnits=1
```

The map view does not lock or warn based on missing required units. At launch, `Vessel` matches owned surface ships and opens the restricted side-mission unit picker. Submarine side missions should omit restriction keys and launch the authored mission as-is, using the pre-placed submarine in the mission file.

## Aircraft And Flight Decks

Aircraft and helicopters are purchased from the same point pool as ships. The UI groups aircraft by type and squadron and shows pool counts.

Aircraft pools behave like this:

- the left roster shows the remaining aircraft available to purchase from each squadron
- the right `Aviation` tab shows aircraft already owned from each squadron
- right-side aviation entries should show pool counts as available versus assigned aircraft
- adding a squadron aircraft buys one aircraft from that pool
- removing an owned aircraft returns one aircraft from that pool
- assigned aircraft are written as grouped airgroup counts on compatible flight deck hosts
- aircraft assignment details are shown from the `Ships` tab, not from the center inspector panel

Helicopters can be assigned to compatible ship flight decks. Carrier aircraft can be assigned to compatible carriers and other valid flight-deck hosts. If a helicopter is not assigned to a ship, it can be generated airborne near the task force with unlimited fuel and a default loadout.

Large air wings are still expected to use the pooled squadron model rather than one UI row per aircraft.

## Task Force Builder Checkout

The Task Force Builder is a purchase/commit flow:

- the player adds ships, submarines, and aviation from the available roster
- aircraft and helicopters can be bought into the Aviation pool or assigned directly to a compatible ship
- the player must commit purchases before those units become part of the campaign task force
- the commit action shows a confirmation dialog
- committed purchases append to `[PersistentData] CurrentTaskForce`
- committed purchases subtract from `[PersistentData] TaskForceAvailablePoints`

If the player tries to launch with no units or without committing the starting task force, the campaign map shows a warning and offers a button to open the Task Force Builder.

Purchase windows are controlled by `TaskForceBuilderPurchasesEnabled=True` on mission sections. Task Force Status, not Builder, owns current-force management such as assignment, dismiss, decommission, repair, and detailed snapshot display.

## Briefings And Objectives

Mission authors should write objectives around the player's flagship:

- use `Your flagship must survive`
- avoid naming fixed player ships such as `USS Scott must survive`
- remove survival gates for generated non-flagship escorts unless intentionally required

Dynamic flagship name injection into objectives and trigger messages is parked for later. For now, generic flagship wording is the safest authoring style.

## Testing Checklist

For each mission:

- confirm `Taskforce1Vessel1` exists and has `TaskForceModeAnchor=True`
- confirm submarine anchors exist for missions that support submarines
- confirm generated task force units inherit heading and `Telegraph` from the anchor
- confirm authored ready-up tasks are stripped and replaced by player choices
- confirm mission rewards and cap increases apply after completion
- confirm repair/rearm flags are set in `campaign.ini`
- confirm side mission restrictions produce correct warnings
- confirm the mission still plays if the player brings a different legal task force

## Parked Items

These items are intentionally deferred:

- dynamic flagship name injection into objectives, triggers, and messages
- full AAR/debrief page integration for Task Force Mode rewards
- dedicated Task Force Status tile UI
- campaign bottom bar panel wiring beyond the initial placeholder buttons
- serial number assignment for individual aircraft
- optional `RequireFlagship=False` campaign mode
- enemy task force rosters
- full storage/ammunition management between missions
