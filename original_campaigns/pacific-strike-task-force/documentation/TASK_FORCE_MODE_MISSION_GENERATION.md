# Task Force Mode Mission Generation

This document describes the launch-time mission generation rules implemented by `LinearCampaignTaskForceMissionGenerator`.

Task Force Mode does not overwrite authored mission files. When the player launches a mission, the selected mission INI is copied to a temporary generated INI, player task-force placeholders are replaced, and the game loads that generated file. Red, neutral, and non-player task forces are left as authored.

## Launch Gates

`LinearCampaignMission.PlayCommand` asks `LinearCampaignMapViewModel.TryPrepareTaskForceForMissionLaunch(...)` for the mission path to load.

For non-Task-Force campaigns, the original mission path is returned unchanged.

For Task Force Mode campaigns, launch is blocked in this order:

1. Commander must be committed.
2. A current task force must exist in `[PersistentData] CurrentTaskForce`, with active snapshot detail available for launch hydration.
3. If `[TaskForceMode] FlagshipRequired=True`, the snapshot must contain at least one purchased surface ship.
4. `LinearCampaignTaskForceMissionGenerator.TryGenerate(...)` must successfully create the temporary mission INI.

Completed missions are treated as replays. Replay launch uses the exact task force snapshot saved for that mission index and does not inherit the current/latest task force state.

Blocked launches show warning dialogs:

- Missing commander: opens Service Record.
- Missing task force: opens Task Force Builder when Builder is available.
- Missing flagship: opens Task Force Builder when Builder is available.
- Generation failure: shows a generic mission generation warning.

## Generation Entry Point

The generator API is:

```csharp
LinearCampaignTaskForceMissionGenerator.TryGenerate(
  LinearCampaignMission mission,
  string sourceMissionPath,
  LinearCampaignTaskForceSnapshot snapshot,
  out string generatedMissionPath,
  out string failureHeader,
  out string failureMessage)
```

Inputs:

- `mission`: selected linear campaign mission.
- `sourceMissionPath`: resolved authored mission INI path.
- `snapshot`: active task force snapshot, including purchased ships, submarines, aircraft pools, assignments, task force name, available points, and cap points.

Outputs:

- `generatedMissionPath`: temporary INI path to load through `Globals.currentMissionFilePath`.
- `failureHeader` and `failureMessage`: user-facing dialog text if generation fails.

The generated file is written under the campaign temp/save directory as:

```text
task_force_mode_mission_<missionIndex>.ini
```

The generated INI writes `[File] BaseFile` to the original mission path so debrief and progress attribution still apply to the selected campaign mission.

## Mission Logistics Rules

Task Force Mode logistics are read from the selected mission section in `campaign.ini`:

```ini
TaskForceModeRearm=True
TaskForceModeRepair=True
```

If either key is missing, it defaults to `True`.

The launch generator uses `LinearCampaignTaskForceLogistics` to translate rearm into the legacy per-unit mission key expected by campaign persistence:

```ini
CampaignRearm=True
```

Generated task-force ships and submarines receive `CampaignRearm=True` when rearm is enabled. `TaskForceModeRepair=True` means repair facilities are available before that mission; it does not write `CampaignRepair=True` or repair units for free. Paid repairs are applied immediately in Task Force Status before launch. Aircraft do not receive `CampaignRearm` or `CampaignRepair`; those logistics keys do not apply to aircraft.

Exiting or aborting a mission has no logistics consequence. Restarting an incomplete mission regenerates from the same task-force snapshot and mission logistics rules as a fresh launch.

## Player Task Force Resolution

The linear campaign map shows mission nodes only. Task-force, aircraft, submarine, airbase/contact, and threat-axis foreground markers are intentionally not rendered on the campaign map because they duplicate mission briefing information and can imply map-level unit control. Mission content is communicated through the briefing and through launch-time mission generation instead.

The mission modal button label changes from `Play Mission` to `Replay Mission` when the selected mission is complete.

Replay still generates a temporary mission INI, but it uses the original mission snapshot rather than the post-mission campaign task force.

The player task force prefix comes from:

```ini
[Mission]
PlayerTaskforce=Taskforce1
```

If the key is missing, the generator uses `Taskforce1`.

The surface anchor is resolved by scanning player vessel sections for:

```ini
TaskForceModeAnchor=True
```

If no explicit anchor exists, the fallback is:

```text
<PlayerTaskforce>Vessel1
```

The surface anchor's `RelativePositionInNM` is the spawn origin for the generated surface task force and unassigned aircraft.

The submarine anchor is:

```text
<PlayerTaskforce>Submarine1
```

If that section is missing or has no valid `RelativePositionInNM`, submarine generation falls back to the surface anchor with `shallow` depth.

## Section Replacement Rules

Generation removes and rebuilds only player-side placeholder sections with these prefixes:

- `<PlayerTaskforce>VesselN`
- `<PlayerTaskforce>SubmarineN`

Matching child sections are also removed, such as:

```text
Taskforce1Vessel1_WeaponSystem1
```

Enemy, neutral, and non-player sections are not removed or rewritten.

Authored player aircraft and helicopter sections are preserved. Generated task-force aircraft, when needed, are appended after the authored player aircraft/helicopter counts.

Player formations are rebuilt as follows:

- Existing player formations that reference generated surface/submarine units are removed.
- Existing player aircraft/helicopter formations are preserved.
- Preserved formations are written first.
- Generated task-force formations are appended after preserved formations.
- `<PlayerTaskforce>_NumberOfFormations` is updated to the final count.

## Surface Ship Rules

Surface ships are snapshot items where:

```text
Category == "Ship"
```

Generation writes:

```ini
[Mission]
NumberOf<PlayerTaskforce>Vessels=<purchased ship count>
```

Each purchased ship becomes:

```text
<PlayerTaskforce>Vessel1
<PlayerTaskforce>Vessel2
...
```

`<PlayerTaskforce>Vessel1` is always the task force anchor and keeps `TaskForceModeAnchor=True`.

Generated ship fields:

```ini
Type=<snapshot UnitType>
VariantReference=<snapshot VariantReference or Default>
LoadoutVariant=<snapshot SelectedLoadoutVariant or Default>
CrewSkill=Trained
RelativePositionInNM=<generated position>
Telegraph=1
TaskForceModeAnchor=True ; only on Vessel1
```

For `<PlayerTaskforce>Vessel1`, `Telegraph` and `Heading` are copied from the authored surface anchor section before replacement. If the authored anchor has no `Heading`, no generated `Heading` key is written. Other generated ships use the baseline `Telegraph=1` unless later rules override them.

Every generated surface ship also receives:

```ini
CustomAirGroup=True
```

This intentionally overrides the default aircraft carried by the ship/unit variant.

If the Task Force Status snapshot has aircraft assigned to that ship instance, the generated ship section writes one custom air group line per assigned aircraft type:

```ini
<aircraft UnitType>=<SquadronReference>,<assigned count>
```

If more than one assigned aircraft pool uses the same aircraft type, entries are joined with `|`:

```ini
usn_sh-2f=Squadron7,1|Squadron8,1
```

If no aircraft are assigned to the ship, `CustomAirGroup=True` is still written and no aircraft type lines are written. This clears any default embarked air group from the ship variant.

Positioning:

- Ship 1 uses the authored surface anchor position.
- Additional ships are placed around the anchor on a 1.5 nm radial circle.

Generated surface formation:

```ini
<PlayerTaskforce>_FormationN=<all generated vessels>|<snapshot task force name>|Circle|1.5|OverrideSpawnPositions
```

If the snapshot task force name is blank, the formation name falls back to `Task Force`.

## Submarine Rules

Submarines are snapshot items where:

```text
Category == "Submarine"
```

Generation writes:

```ini
[Mission]
NumberOf<PlayerTaskforce>Submarines=<purchased submarine count>
```

Each purchased submarine becomes:

```text
<PlayerTaskforce>Submarine1
<PlayerTaskforce>Submarine2
...
```

Generated submarine fields:

```ini
Type=<snapshot UnitType>
VariantReference=<snapshot VariantReference or Default>
LoadoutVariant=<snapshot SelectedLoadoutVariant or Default>
CrewSkill=Trained
RelativePositionInNM=<generated position>
Telegraph=1
```

Positioning:

- Submarine 1 uses the authored `<PlayerTaskforce>Submarine1` anchor position.
- Additional submarines are placed around that anchor at 2 nm radial offsets.
- The anchor depth token is preserved, such as `shallow`; if no submarine anchor exists, depth defaults to `shallow`.

If the task force has no purchased submarines, player submarine placeholders are removed and:

```ini
NumberOf<PlayerTaskforce>Submarines=0
```

## Aircraft Rules

Aircraft generation only handles aircraft that are in the task force snapshot but not assigned to a ship.

Unassigned count per aircraft pool:

```text
QuantityPurchased - sum(AircraftAssignments)
```

Ship-assigned aircraft are not emitted as independent mission aircraft. They are written into the assigned ship's `CustomAirGroup`.

Unassigned aircraft are generated only when the player side has no authored airbase or airfield. The generator checks player land units:

```text
<PlayerTaskforce>LandUnitN
```

If any player land unit `Type` contains `airbase` or `airfield`, unassigned aircraft are not generated and authored player aircraft/helicopter placeholders are preserved.

Authored player aircraft/helicopter sections remain in the mission. When unassigned task-force aircraft are generated, they are appended after those authored counts:

```ini
NumberOf<PlayerTaskforce>Aircraft=<authored fixed-wing count + generated fixed-wing count>
NumberOf<PlayerTaskforce>Helicopters=<authored helicopter count + generated helicopter count>
```

Generated aircraft/helicopter fields:

```ini
Type=<snapshot UnitType>
SquadronReference=<snapshot VariantReference or Default>
LoadoutVariant=<snapshot SelectedLoadoutVariant or Default>
CrewSkill=Trained
RelativePositionInNM=<surface anchor offset>,3000,<surface anchor offset>
Telegraph=3
UnlimitedFuel=True
```

Positioning:

- Aircraft spawn at 3000 ft.
- Aircraft are offset 5 nm from the surface anchor.
- Multiple aircraft are distributed radially around the surface anchor.

Fixed-wing formation rules:

- Fixed-wing aircraft are grouped by purchased aircraft pool.
- Groups have a maximum size of 2.
- Formation name format is:

```text
N x <aircraft display name>
```

- Formation shape is:

```ini
<PlayerTaskforce>_FormationN=<aircraft sections>|N x <aircraft display name>|Vic|0.1|OverrideSpawnPositions
```

Helicopter rules:

- Helicopters always spawn solo.
- Helicopters are never grouped with other helicopters.
- No helicopter formation is generated for this baseline.

## Values Currently Hardcoded

These values are fixed in the baseline generator:

```text
Surface ship circle radius: 1.5 nm
Additional submarine offset: 2 nm
Unassigned aircraft offset: 5 nm
Unassigned aircraft altitude: 3000 ft
Generated crew skill: Trained
Generated ship telegraph: 1
Generated aircraft/helicopter telegraph: 3
```

## Explicit Non-Goals

This baseline does not:

- Modify red or neutral units.
- Implement `TaskForceModeFreeAgent=True`.
- Implement `TaskForceModeNonTracked=True`.
- Generate ship-assigned aircraft as ready aircraft or flight-deck state.
- Persist generated mission INIs back into authored campaign content.
- Track dismissal, repair, decommission, damage, or post-mission survival state inside the mission generator.
