# Task Force Mode Latest State

This is the living project summary for the Pacific Strike Task Force Mode campaign variant. It should be kept as a "current state" release note: what exists now, what decisions are settled, how campaign authors should use it, and what is intentionally parked for later. It is not meant to be a detailed changelog.

## Overview

Task Force Mode is a campaign-side extension for linear campaigns. The authored mission format and normal mission loading flow stay mostly intact. Generated Task Force missions read campaign save state, roster data, player task force choices, and mission settings, then write a temporary mission INI that the existing mission scene can load normally. Authored side missions can omit Task Force generation and launch their mission file unchanged.

The design goal is to let players build and manage a persistent task force across a linear campaign without forcing mission authors to hand-place every possible player unit combination.

Fresh-branch implementation note: the first rebuilt slice is the linear campaign bottom command bar in the active campaign window view. It adds empty campaign buttons for `Task Force Status`, `Task Force Builder`, and `Service Record`; the dedicated panels, Task Force Mode runtime, and mission generation are still upcoming work on this branch. The free-event overlay now reserves spacing above the command bar, and the unused `LinearCampaignMapScreen` view has been removed.

## Core Decisions

- Task Force Mode is enabled per campaign with a `[TaskForceMode]` block in `campaign.ini`.
- Task Force Mode campaigns require a roster file. If the roster is missing or invalid, the campaign should fail clearly.
- Unit costs live in a separate roster file next to `campaign.ini`, not inside the main campaign file.
- Ships, helicopters, and aircraft all spend from the same point pool. Submarines are not part of the player campaign task force.
- Player units are generated into a temporary mission INI at mission launch time.
- The actual mission scene and SceneCreator path should stay minimally changed.
- Player task force units replace a mission-authored anchor unit, currently `Taskforce1Vessel1`.
- Pre-placed player units can still exist if marked/preserved separately, but Task Force Mode-generated units are the primary player force.
- The flagship is the only unit whose survival is mandatory.
- Submarine side missions use authored, pre-placed player submarines and launch without Task Force generation.
- Smaller escorts are not flagship eligible for MVP.
- Initial task force setup uses a checkout/commit model. Before Mission 1, the player queues ships and aviation in the builder, then commits the force. After commit, the initial force is locked unless the player restarts the campaign or uses debug tools.
- Helicopters and aircraft are bought from squadron pools one aircraft at a time, but represented as pooled squadron counts instead of one persistent UI row per aircraft.
- Aircraft counts in the roster represent the total available inventory for that squadron pool. The right-side Aviation tab should show pool available versus assigned counts.
- Crew skill starts at `Trained`, then advances by successful deployed mission credits.
- Post-mission Task Force Mode results stay inside the popup that appears after returning to the campaign map. This includes crew progression, command point rewards, and command cap increases.
- SceneCreator and the after-action/debrief page are intentionally not modified for MVP Task Force Mode results.
- Localization keys under `[TaskForceMode]` are stored as bare keys such as `aircraft`, `assignsquadron`, and `tooltipcheckout`. The runtime/XAML prefix is still `taskforcemode...`, but the source INI should not use doubled keys such as `taskforcemodetaskforcemodeaircraft`.
- Commander setup is campaign-authored in `commander_settings.ini`, referenced from `[TaskForceMode] CommanderSettingsFile`. That file owns commander nations, name defaults, name pools, same-nation discount value, navy display names, navy emblem paths, and `[OfficerRanks]`.
- Commander rank progression uses `[OfficerRanks]` entries in `commander_settings.ini`. Each rank entry includes a gameplay `RankLevel`, allowing one `CommanderStartingRankLevel` value to resolve across U.S., Japanese, and Australian naval ranks without relying on NATO O-grade parity.

## Implemented Features

- Fresh-branch UI shell: the active linear campaign view has a tactical-style bottom command bar with placeholder buttons for `Task Force Status`, `Task Force Builder`, and `Service Record`.
- Free-event overlays respect the bottom command bar by sizing to the active window-host area and leaving a small spacer above the bar.
- Removed the unused `LinearCampaignMapScreen` XAML/code-behind pair so future campaign-map work targets `LinearCampaignView.xaml`.
- Task Force Mode campaign parsing.
- Required task force roster parsing.
- Persistent task force state:
  - points
  - point cap
  - owned units
  - owned aircraft/helicopter pool counts
  - flagship
  - task force name
  - aircraft and helicopter assignment counts
  - default circular spawn offsets
  - unlocked loadouts
  - crew mission credits
- Task Force Builder UI on the linear campaign map.
- Mission modal `Task Force Status` button.
- Task force name editing and random name selection.
- Point spending and point cap enforcement.
- Task force checkout confirmation flow. A mission cannot launch with an empty or uncommitted starting task force; the warning popup includes a button to open the Task Force Builder.
- Debug point and point-cap controls in F10/developer mode.
- Debug reset access in F10/developer mode, allowing testers to rebuild a committed starting force.
- Ships and aircraft roster tabs.
- Aircraft tab merges helicopters and fixed-wing aircraft, separated by section headers.
- Aviation pool UI for owned aircraft/helicopters. The right-side tab is named `Aviation`, and it separates helicopters and fixed-wing aircraft with simple list rows.
- Current task force tabs hide empty submarine/aircraft tabs.
- Current task force list sorts the flagship to the top.
- Flagship star badge in the builder and inspector.
- Unit inspector panel:
  - profile image
  - points overlay
  - class/type overlay
  - crew proficiency
  - flagship status
  - hangar capacity
  - supported aircraft
  - available loadouts with store contents and locked-loadout indicators
  - assigned aircraft
  - roster-authored short tactical descriptions for player-facing procurement guidance
- Task Force Builder plain UI text is localized through `[TaskForceMode]` keys in `language_*/ui.ini`, including tabs, buttons, overlays, tooltips, debug controls, and confirmation dialog text.
- The Builder roster tabs are `Ships` and `Aircraft`; submarines are omitted from the player purchase roster.
- Commander setup now lives in the Service Record window. It supports first-open onboarding, nation selection, name defaults/randomization from `ui.ini` name pools, one-way commander commit state, and same-nation Builder purchase discounts.
- Aircraft inspector purchase controls:
  - quantity selector
  - total cost
  - `Add to Unit` dropdown for compatible flight deck hosts
  - `Add to Aviation Pool`
- Ship profile images in the Current Task Force list.
- Unit status popups on the campaign map show the actual generated task force.
- Briefing substitution for the task force box and flagship address.
- Runtime mission generation:
  - generated temp mission INI
  - generated player vessels
  - generated submarines using a mission-defined submarine spawn anchor
  - generated unassigned helicopters near the task force
  - generated air groups assigned to compatible flight deck hosts
  - campaign tags
  - crew skill
  - loadout variants
  - formation offsets
  - anchor heading
  - anchor telegraph/speed setting
  - authored ready-up task stripping
  - zero-time pre-mission ready-up tasks
- Mission editor preservation of manually-authored `TaskForceModeAnchor=True`.
- Campaign-view post-mission Task Force summary popup for command point rewards, point cap increases, loadout access changes, crew proficiency gains, and damaged-unit outcomes.
- Per-mission force restrictions for side missions, including detached single-unit missions.
- Aircraft and helicopters now use pooled squadron counts. The player buys one aircraft at a time from a squadron pool, assigns counts to compatible hosts, and the generated mission writes grouped airgroup counts.
- Current Task Force context menus:
  - ships can remove, assign squadron, switch flagship when eligible, and open Unit Reference
  - aviation pool entries can remove all, assign to compatible hosts, dismiss when eligible, and open Unit Reference
  - decommission is ship/submarine-only and should be disabled unless damage rules allow it
- MVP damage handling in the Task Force Builder:
  - Light/moderate damaged ships and submarines can be sent to repair for one mission.
  - Heavily damaged ships and submarines can be decommissioned for a configurable partial point refund.
  - Destroyed units are removed with no refund.
  - The flagship must transfer flag before repair or decommission.
- First playable vertical slice of Pacific Strike Task Force Mode.
- Mainline missions updated to use generated flagship survival logic:
  - Missions 1, 2, 3, 5, 6, and 7 require only `Your flagship must survive` for player task force survival.
  - Mission 4 retains the authored `M/V Sealift Caribbean Sea must survive` tanker escort objective.
  - Mission 8 retains the authored `All Allied amphibious ships must survive` defense objective.
  - Optional missions 03A and 03B are parked for later design review and remain authored around their specific detached units.
  - Named generated-escort survival gates such as Scott, Preble, Tachikaze, and Sterett were removed from the mainline Task Force Mode mission flow.

## Campaign INI Setup

Task Force Mode is configured in `campaign.ini`:

```ini
[TaskForceMode]
Enabled=True
RosterFile=player_task_force_roster.ini
StartingPoints=50
PointCap=50
UnitDecommissionPointReturnModifier=0.25
UnitDismissPointReturnModifier=0.5
UnitDestroyedPointReturnModifier=0
DamageToAllowRepair=Light,Moderate
DamageToDisallowRepair=Heavy
RepairPointsCost=Light,0.1|Moderate,0.25
DefaultTaskForceName=Task Group 77.3
TaskForceNameOptions=Task Group 77.3|Cruiser Striking Force|TG 77.3 - Surface Striking Force|Cactus Strike Force
BasicShipLoadoutVariants=Default|Early
LockedShipLoadoutVariants=Late
BasicAircraftLoadouts=AirToAir|Default|Ferry|ASW|AntiShip|Strike|AntiShipLongRange|StrikeLongRange|SEADLongRange|EW|SEAD|Recon
LockedAircraftLoadouts=AntiShipHeavy|AirToAirIntercept|SEADAdvanced|AntiShipPrecision
SubmarineSpawnMode=MissionDefined
CrewSkillInitial=Trained
CrewSkillThresholds=Seasoned:1|Veterans:3|Ultra:5
```

Per-mission campaign settings can be added under mission event sections:

```ini
[Mission 1 Event Name]
TaskForceCompletionPointReward=10
TaskForceCompletionPointCapIncrease=0
TaskForceBuilderPurchasesEnabled=True
TaskForceUnlockShipLoadoutVariants=
TaskForceUnlockAircraftLoadouts=
TaskForceLoadoutsToUnlock=
TaskForceLoadoutsToLock=
TaskForceModeRequiredUnitType=
TaskForceModeMaxUnits=
```

`TaskForceBuilderPurchasesEnabled=True` marks a reinforcement/purchase window for the selected mission. The Task Force Builder can still be opened as a status, naming, formation, and assignment screen outside those windows, but adding new roster units is disabled.

Builder persistence is intentionally narrow: it builds an available roster from the mission/roster rules, reads points from `[PersistentData] TaskForceAvailablePoints` and `TaskForcePointCap`, and commits purchases by appending to `[PersistentData] CurrentTaskForce` while subtracting the purchase cost. It does not scan mission snapshots or derive available points from the cost of the current task force.

`TaskForceLoadoutsToUnlock` and `TaskForceLoadoutsToLock` are generic mission-completion controls. They accept both aircraft loadout names, such as `AntiShipHeavy`, and ship loadout variant names, such as `Late`. Locking removes previously earned reward loadouts from the save, while baseline loadouts from `[TaskForceMode]` remain available.

`TaskForceModeRequiredUnitType` and `TaskForceModeMaxUnits` support surface-vessel side missions. The map view does not disable Play for missing required units; the launch flow opens a unit picker when the current task force has eligible surface ships. Submarine side missions should omit these keys and launch the authored mission as-is with its pre-placed, balance-tested submarine. Example:

```ini
; Require at least one surface vessel in the deployable task force.
TaskForceModeRequiredUnitType=Vessel

; Restrict the side mission launch selection to a very small force.
TaskForceModeMaxUnits=1
```

Damage and repair authoring knobs:

- `UnitDecommissionPointReturnModifier` controls the point refund for decommissioning a heavily damaged unit.
- `UnitDestroyedPointReturnModifier` controls the point refund for destroyed units. Pacific Strike TF currently uses `0`.
- `DamageToAllowRepair` and `DamageToDisallowRepair` let campaign makers tune which damage states can be repaired or must be decommissioned.
- `RepairPointsCost` maps repairable damage states to a fraction of the unit's original cost.

Current Pacific Strike TF purchase windows:

- Initial setup before Mission 1.
- Refit/reinforcement window before Mission 4.
- Refit/reinforcement window before Mission 7.

## Roster File Format

The roster file should be placed next to `campaign.ini`, for example:

```ini
[AllowedVessels]
usn_ff_knox_84=Variant18,Variant22,Variant23|11
usn_bb_iowa=Variant3|65

[AllowedSubmarines]
usn_ssn_permit=Variant3,Variant10,Variant13,Variant9|24

[AllowedAircraft]
usn_f-14a=Squadron9,4|30
usn_p-3c=Squadron29,1,Squadron31,1|20

[AllowedHelicopters]
usn_sh-2f=Squadron8,6|5
usn_sh-3h=SquadronNameOrReference,2|7

```

Roster values are intentionally compact:

- Left side is the unit type.
- First pipe field is the allowed variant or squadron references.
- Second pipe field is the point cost.
- For aircraft/helicopters, each squadron reference is followed by its available aircraft count before the pipe. The point cost after the pipe is per aircraft.
- Multiple squadrons of the same aircraft type should be placed on the same roster line. The UI groups them under one aircraft type expander on the available roster side.

The UI pulls display names, flags, profile images, classes, types, and squadron names from the normal language and unit data files where possible.

## Mission Authoring Rules

- Place a player task force anchor unit in the mission, normally `Taskforce1Vessel1`.
- Mark the anchor with:

```ini
TaskForceModeAnchor=True
```

- Task Force Mode replaces the anchor with the generated player surface task force.
- The generated force inherits the anchor location, heading, and telegraph setting.
- If no player formation has been edited, ships use the simple circle formation fallback.
- If the player edits formation before launch, formation offsets are written into the generated mission INI.
- Formation editing support exists in runtime state and generation, but the dedicated button is currently being moved out of the Task Force Builder. The likely future home is a campaign bottom bar or pre-launch command surface.
- Submarine side missions omit Task Force generation and launch the authored mission file as-is.
- Authored submarine side missions should place and balance the player submarine directly in the mission.
- The campaign map does not gate submarine side missions against the player task force.
- Unassigned helicopters spawn near the task force in the air with unlimited fuel and default loadout.
- Aircraft not assigned to a ship should eventually be assigned to a mission-declared campaign airfield when available.
- Authored ready-up tasks should not be used for Task Force Mode player aircraft; the player prepares aircraft before launch.

## Mission-Level Logistics

Task Force Mode does not require campaign repair/rearm/reset flags to be hand-authored on the generated unit sections in mission INIs. Instead, campaign makers set logistics behavior on the mission section in `campaign.ini`.

Supported mission-section keys:

```ini
TaskForceModeRepair=True
TaskForceModeRearm=True
```

The runtime writes `CampaignRearm=True` into generated ship/submarine sections when mission rearm is enabled. `TaskForceModeRepair=True` means repair facilities are available before that mission; repairs are paid actions in Task Force Status and are applied immediately to saved unit state.

Generated Task Force Mode vessels always write `CustomAirGroup=True`, even when no aircraft are assigned, so mission-level airgroup reset does not restore unit-file default aircraft. The campaign task force air assignments remain the authority.

Current Pacific Strike TF usage:

- These flags apply to the mission section where they are declared. For example, `[Mission4]` is Pacific Strike Mission 2, so logistics flags in `[Mission4]` apply when Mission 2 is generated and launched.
- Mission 2 provides rearm and repair facilities, representing full resupply access after Mission 1.
- Mission 3 provides rearm and repair facilities, representing full resupply access after Mission 2.
- Mission 4 is back-to-back after Mission 3 and does not grant automatic logistics reset.
- Mission 5 provides rearm and repair facilities after the campaign time gap.
- Mission 6 provides rearm and repair facilities after the post-Kirov consolidation.
- Mission 8 provides rearm and repair facilities at the Sandakan forward staging base.

Future storage-management gameplay is parked for later: command points could eventually be spent to partially restore ammunition, while specific base missions can still grant automatic full restock.

## Localization Rules

Task Force Mode strings live under `[TaskForceMode]` in each `language_*/ui.ini`.

Use bare source keys:

```ini
[TaskForceMode]
aircraft=Aircraft
assignsquadron=Assign Squadron
tooltipcheckout=Commit this task force before launching the mission.
```

Do not include `taskforcemode` in the source key inside the section. The runtime/XAML lookup already namespaces the section into keys such as `taskforcemodeaircraft`. A key like `taskforcemodeaircraft` inside `[TaskForceMode]` becomes `taskforcemodetaskforcemodeaircraft`, which is exactly the duplicated form we are eliminating.

## Flagship Survival Pattern

For Task Force Mode missions, only the flagship should be required to survive.

Recommended objective text:

```ini
Objective_FlagshipMustSurvive=Your flagship must survive
```

Recommended failure trigger pattern:

```ini
[Trigger2]
Name=Flagship must survive
Condition_Condition1_Type=UnitDestroyed
Condition_Condition1_Units=Taskforce1Vessel1
ConditionsCompleted=<Condition1>
Action_Taskforce1_Message=Taskforce1FlagshipmustsurviveMessage
Action_EndMission=True
Action_Victory=Taskforce2
Action_ObjectivesFailed=FlagshipMustSurvive
Action_ObjectivesCancel=ReachObjectiveArea
```

Recommended failure message:

```ini
Taskforce1FlagshipmustsurviveMessage=MISSION FAILED|Defeat! Your flagship has been sunk!|Exit mission
```

Remove any survival objectives/triggers for named escorts or other fixed mission ships unless they are explicitly not part of the player task force and the mission design requires it.

## Crew Skill Progression

Current desired progression:

- Start: `Trained`
- 1 successful deployed mission: `Seasoned`
- 3 successful deployed missions: `Veterans`
- 5 successful deployed missions: `Ultra`

Post-mission progression should be summarized in the campaign-view popup after returning from the mission. The same popup also reports command point rewards and command cap increases. If there are no promotions and no point/cap changes, no popup is shown.

## Points and Unit Valuation Rules

Ship points were assessed manually rather than relying purely on `UnitScoreValue`.

Rules used:

- Ships with SH-2F/SH-3H-capable helicopter facilities are worth more than QH-50-only ships.
- Two helicopter slots are worth more than one.
- SAM count matters.
- Long-range SM-2ER capability matters heavily.
- Harpoon count matters.
- Towed array sonar matters.
- Modern radar matters, especially 3D radar.
- 127mm gun count matters.
- CIWS matters.
- Late/reward loadouts do not increase purchase cost.
- ABL Tomahawk Spruance variants receive a significant cost premium.
- E-2C and E-3A are high-value force multipliers and should be no less than 50 points.
- EA-6B is a highly capable unit and was increased to 25 points.
- SH-2F is 5 points.
- SH-3H is 7 points.
- Modified Barnegat-class Philippine frigate is 10 points as a Harpoon platform.
- Mahan NTU test ship is 38 points as a high-end SM-2ER Block II/Harpoon D AAW unit.
- Ticonderoga-class USS Valley Forge CG-50 is 65 points minimum as a high-end Aegis cruiser.

## Loadout Decisions

Basic aircraft loadouts:

- `AirToAir`
- `Default`
- `Ferry`
- `ASW`
- `AntiShip`
- `Strike`
- all long-range variants
- `EW`
- `SEAD`
- `Recon`

Locked aircraft loadouts:

- `AntiShipHeavy`
- `AirToAirIntercept`
- `SEADAdvanced`
- `AntiShipPrecision`

Ship loadout variants:

- Basic: `Default`, `Early`
- Reward/unlocked: `Late`

## Parked Items

- Dynamic flagship text tokens for generated mission INIs.
  - Let mission authors write placeholders such as `{TaskForceFlagshipName}`, `{TaskForceName}`, and `{TaskForceFlagshipFullName}` in messages, objectives, labels, and localized strings.
  - Replace those tokens during Task Force Mode mission generation.
  - Example:

```ini
Taskforce1FlagshipmustsurviveMessage=MISSION FAILED|Defeat! {TaskForceFlagshipName} has been sunk!|Exit mission
Objective_FlagshipMustSurvive={TaskForceFlagshipName} must survive
```

- Full land-based aircraft management mode.
  - Mission 5 and later should expose a campaign airfield such as Broom Airbase.
  - Aircraft not assigned to a carrier or ship should be assigned to the active campaign airfield when available.
  - Missions without an available campaign airfield should keep those aircraft parked in campaign save state.

- Per-mission task force snapshot/replay handling.
  - Replaying earlier missions should probably use the task force state that was available at that mission, not a later stronger force.
  - This prevents advancing the campaign, buying stronger units, then replaying earlier missions for easier progression.

- Aircraft serial number support.
  - Aircraft and helicopters have serial number indices in mission files.
  - We discussed reflecting exact serials in the builder and generated missions.
  - This was explicitly deferred to avoid object loader changes.

- More advanced post-mission summary.
  - Current campaign-view popup is the home for MVP post-mission Task Force Mode results.
  - Later versions could summarize damaged units, repairs, aircraft losses, unlocked loadouts, and logistics outcomes in the same popup.
  - AAR/debrief integration is parked.

- More refined Task Force Builder UI.
- Dedicated Task Force Status tile UI for fast at-a-glance review.
- Dedicated loaner unit tile-selection UI for side missions.
- Dedicated land-based air UI if the combined Aircraft tab becomes too crowded.
- More polished class dropdowns and filtering.

- Loaner unit difficulty ratings and variable rewards.
  - Some command-assigned loaner units may be better or worse suited to a restricted side mission.
  - Future authoring could show difficulty stars per loaner and grant different rewards for completing the same mission with weaker or stronger loaners.

- Optional no-flagship Task Force Mode.
  - Current MVP assumes a flagship and treats flagship survival as the player task force survival condition.
  - Later, `RequireFlagship=True/False` could let campaign makers build Task Force Mode campaigns without flagship requirements.
  - If disabled, runtime should skip flagship checks and the builder should hide flagship UI and context menu actions.

## Notes For Future Campaign Makers

- Keep authored missions generic where possible.
- Use a single player task force anchor.
- Place player submarine anchors in areas where their presence does not totally break pacing or balance (e.g. don't place them right on top of enemy units)
- Avoid hardcoding specific player ship names in objective text.
- Prefer `Your flagship must survive` until dynamic flagship tokens are implemented.
- Do not author mandatory survival objectives for generated escorts.
- Use a mission-defined submarine spawn anchor to control the player submarine operating area and preserve pacing.
- Put all costs and availability in the roster file.
- Use campaign mission sections for rewards, cap increases, and unlocks.
- Test each mission with a cheap/minimum task force and a high-cap task force to make sure balance still holds.
