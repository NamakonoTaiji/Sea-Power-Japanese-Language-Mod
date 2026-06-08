# Task Force Mode Save / Load

## Summary
Task Force Mode uses `[PersistentData]` for the campaign-level master values: `CurrentTaskForce`, `TaskForceAvailablePoints`, and `TaskForcePointCap`. Canonical `TaskForceMode_*` sections preserve detailed snapshots for Status, launch, replay, debrief, and native mission bridge state. Native `Mission_*` unit state sections still exist, but they are treated as a bridge for the existing mission loader and post-mission state capture.

This document records the intended save/load behavior, section schema, launch flow, post-mission flow, and known validation checkpoints.

## Important Files
- `Assets/Scripts/UI/Linear Campaign/ViewModels/LinearCampaignTaskForceSaveManager.cs`
  - Owns Task Force Mode persistence.
- `Assets/Scripts/UI/Linear Campaign/LinearCampaign.cs`
  - Copies canonical sections into/out of campaign saves and captures post-mission native state.
- `Assets/Scripts/UI/Linear Campaign/ViewModels/LinearCampaignMapViewModel.cs`
  - Prepares launch snapshots, opens Builder from persistent points/roster, opens Status from current snapshot state, and applies completion rewards.
- `Assets/Scripts/UI/Linear Campaign/ViewModels/LinearCampaignTaskForceMissionGenerator.cs`
  - Generates launch mission INI from a prepared snapshot; it should not own canonical save logic.

## Canonical Sections

### Persistent Campaign State
`[PersistentData]`

Master Task Force Mode keys:
- `CurrentTaskForce`: campaign-level roster list, serialized as `UnitType,VariantReference` for ships/subs or `UnitType,VariantReference,Quantity` for air pools.
- `TaskForceAvailablePoints`: spendable campaign points.
- `TaskForcePointCap`: current campaign points cap.

The Task Force Builder reads and writes these keys directly. It does not scan mission snapshots, inherit old mission snapshots, or rebuild points from purchased item costs.

### Active Snapshot
`[TaskForceMode_Active]`

Detailed state cache for the current campaign task force. `CurrentTaskForce` is the master roster list; active snapshot rows preserve instance ids, loadouts, aircraft assignments, crew skill, and other detail needed by Status and launch.

Important keys:
- `TaskForceName`
- `NumberOfUnits`
- `AvailablePoints`
- `CapPoints`

Unit rows:
- `[TaskForceMode_Active_Unit1]`
- `[TaskForceMode_Active_Unit2]`
- etc.

Unit keys:
- `Type`
- `Folder`
- `Category`
- `InstanceId`
- `VariantReference`
- `LoadoutVariant`
- `Cost`
- `CostPerUnit`
- `BaseCost`
- `BaseCostPerUnit`
- `QuantityPurchased`
- `QuantityAvailable`
- `IsFlagshipEligible`
- `AircraftAssignments`, for air units only, format: `<hostInstanceId>:<count>|<hostInstanceId>:<count>`

### Mission Launch Snapshot
`[TaskForceMode_MissionN_Launch]`

Frozen task force snapshot used to launch or replay mission `N`.

Unit rows:
- `[TaskForceMode_MissionN_Launch_Unit1]`
- `[TaskForceMode_MissionN_Launch_Unit2]`
- etc.

Launch snapshots are important because replaying a completed mission should use the task force as it existed when that mission originally launched, not the current campaign task force.

### Mission Post Snapshot
`[TaskForceMode_MissionN_Post]`

Task force snapshot committed after mission `N` completes. This becomes the basis for active state and later missions.

Important extra key:
- `CompletionRewardsApplied=True`

This guard prevents mission completion rewards from being applied twice during load-time reconciliation.

### Frozen Launch Unit State
`[TaskForceMode_MissionN_LaunchState_<InstanceId>]`

Exact saved unit state used at mission launch. Child system sections use the same prefix:

```ini
[TaskForceMode_Mission4_LaunchState_abc123]
[TaskForceMode_Mission4_LaunchState_abc123SensorSystemRadar3]
[TaskForceMode_Mission4_LaunchState_abc123WeaponSystemLauncher12]
```

These sections preserve damage, flooding, ammunition, flight deck state, system integrity, and other native unit state values at launch.

### Committed Post Unit State
`[TaskForceMode_MissionN_PostState_<InstanceId>]`

Exact saved unit state captured after mission completion. This is the canonical source for future damage/ammo display and for the next mission launch.

## Native Bridge Sections
Native campaign state sections still look like:

```ini
[Mission_3_<InstanceId>]
[Mission_3_<InstanceId>SensorSystemRadar3]
```

These are not the Task Force Mode source of truth. They are used because the existing native loader expects `Mission_<previousMissionIndex>_<CampaignTag>` sections when loading campaign unit state.

During mission generation, Task Force units are generated with:

```ini
CampaignTag=<InstanceId>
```

This lets native save/load map exact tactical state back to the purchased unit instance, including duplicate-safe ships.

## Normal Launch Flow
For an incomplete mission:

1. `LinearCampaignMapViewModel` gates launch:
   - Task Force Mode enabled.
   - Commander committed.
   - Task force exists.
   - Flagship exists when required.
2. Pending Status edits are written to `TaskForceMode_Active` and `CurrentTaskForce`. Builder purchases are already committed through `[PersistentData]`.
3. `LinearCampaignTaskForceSaveManager.PrepareLaunchSnapshot(missionIndex, isReplay:false)` runs.
4. The launch snapshot is read from `CurrentTaskForce`, hydrated by active snapshot detail.
5. The snapshot is written to `TaskForceMode_MissionN_Launch`.
6. Latest unit state is frozen into `TaskForceMode_MissionN_LaunchState_<InstanceId>*`.
7. Launch state is materialized into native bridge sections:
   - `Mission_(N-1)_<InstanceId>*`
8. Mission generator creates the temporary generated mission INI.
9. Native loader uses generated unit `CampaignTag=<InstanceId>` plus the materialized native bridge state.

Important: incomplete future missions should prefer `TaskForceMode_Active` over inherited old mission snapshots. This prevents stale points/cap or task force state from being reused.

## Replay Launch Flow
For a completed mission:

1. `PrepareLaunchSnapshot(missionIndex, isReplay:true)` reads `TaskForceMode_MissionN_Launch`.
2. It materializes that frozen launch state into native bridge sections.
3. The generated mission is marked as replay:

```ini
[File]
TaskForceModeReplay=True
```

4. `SavePersistentDataFromMission()` ignores replay missions, so replay does not mutate canonical campaign state.

## Post-Mission Flow
On normal mission completion:

1. `LinearCampaign.SavePersistentDataFromMission()` writes native tactical state into temp:
   - `Mission_N_<CampaignTag>*`
2. It writes normal native ammunition and unit state fields.
3. Destroyed campaign tags are marked with `IsDestroyed=True`.
4. `LinearCampaignTaskForceSaveManager.CommitMissionPostState(missionIndex, result)` runs.
5. It reads `TaskForceMode_MissionN_Launch`.
6. For each launched ship/sub, it copies native `Mission_N_<InstanceId>*` into `TaskForceMode_MissionN_PostState_<InstanceId>*`.
7. Destroyed ships/subs are removed from the surviving task force.
8. Aircraft assigned to destroyed hosts are returned by removing those host assignments.
9. `TaskForceMode_MissionN_Post` is written.
10. `CurrentTaskForce` and `TaskForceMode_Active` are updated to the post snapshot.
11. Completion rewards are applied by `LinearCampaignMapViewModel.ApplyTaskForceCompletionRewards`.

## Completion Rewards
Campaign rewards come from each mission section in `campaign.ini`:

```ini
TaskForceModeCompletionPoints=8
TaskForceModeCompletionCapPoints=5
```

Reward behavior:
- Add points to `TaskForceAvailablePoints`.
- Add cap to `TaskForcePointCap`.
- Clamp available points to cap.
- Write rewarded values to `[PersistentData]`, `TaskForceMode_Active`, and `TaskForceMode_MissionN_Post`.
- Stamp `CompletionRewardsApplied=True`.

Load-time reconciliation exists because old/in-progress saves may have post snapshots without the guard. It checks whether rewards are already reflected before applying them. For completed missions with cap rewards, the check compares the post cap against the previous post cap plus the mission reward.

Fresh-save checkpoints that should hold:

```text
After Mission 1 complete: Active = 8/55, Mission3_Post = 8/55
Before Mission 2 launch: Mission4_Launch = 8/55
After Mission 2 complete: Active = 16/60, Mission4_Post = 16/60
Before Mission 3 launch: Mission5_Launch = 16/60
After Mission 3 complete: Active = 24/65, Mission5_Post = 24/65
```

Mission indices are campaign event indices. In Pacific Strike Task Force, Mission 1 gameplay is `[Mission3]`.

## Campaign Save / Load

### Loading a Campaign Save
When a saved campaign file is loaded:

1. Existing native `Mission_*` sections are copied into the temp campaign file.
2. `LinearCampaignTaskForceSaveManager.LoadCanonicalSectionsFromCampaignSave(sourceIni, tempIni)` copies all `TaskForceMode_*` sections into temp.

### Saving from Campaign View
When saving from campaign view:

1. Mission unlock/complete flags and persistent data are written.
2. `LinearCampaignTaskForceSaveManager.SaveCanonicalSectionsToCampaignSave(newIni)` copies canonical `TaskForceMode_*` sections into the save.
3. Native temp sections are copied too, but `TaskForceMode_*` sections are skipped in the generic copy to avoid duplicates.

### In-Mission Manual Saves
In-mission manual saves are tactical saves. They should not commit Task Force Mode post-mission canonical state.

### Exit / Abort
Exiting or aborting a mission has no Task Force Mode consequence. Restarting should be like starting fresh from the mission launch snapshot/state.

## Repair / Rearm Interaction

### Rearm
`TaskForceModeRearm=True` means generated Task Force units should enter the mission rearmed. Rearm is separate from repair and currently does not cost points.

TF Status may show a `Rearmed` badge for the selected upcoming mission, but the saved post-mission ammo state remains the previous mission end-state until launch rearm logic applies.

### Repair
Repairs are paid, immediate campaign actions. Repair:

- Spends `AvailablePoints`.
- Does not change `CapPoints`.
- Clears saved campaign damage state for the selected unit.
- Preserves ammunition and flight deck ammo.
- Uses `LinearCampaignTaskForceSaveManager.TryRepairUnit(instanceId, missionIndex, cost)`.

Repair clears:
- `IsDestroyed`
- `IsSinking`
- `FloodingCompartments`
- `SystemCompartments`
- `CompartmentsDelayedSinkTime`
- child section `CurrentIntegrity`
- child section `Inoperable`

Heavy damage remains locked behind Decommission.

## Damage / Ammo Display
TF Status reads previous end-state, not future launch rearm/repair state.

Damage and ammo come from latest canonical/native state found by:

1. Previous mission post state.
2. Previous mission launch state.
3. Previous native bridge state.

The helper is:

```csharp
LinearCampaignTaskForceSaveManager.FindLatestPriorUnitStateSection(campaignStateIni, instanceId, targetMissionIndex)
```

Normal in-mission damage control may slightly improve damage during a mission. That improved end-state is expected to persist after mission completion unless campaign repair is used.

## Destroyed Units
Destroyed ships/subs:

- Are detected from native post-mission state via `IsDestroyed=True`.
- Are removed from the post snapshot and active snapshot.
- Stop appearing in Builder, Status, and future launch generation.
- Cause assigned aircraft host references to be removed, returning aircraft to the unassigned pool.

Aircraft do not use repair/damage persistence rules.

## Known Good Validation
Fresh-save validation after recent fixes:

- Post-Mission 1:
  - Coontz/Preble: Light damage.
  - Adelaide/Canberra: Moderate damage.
  - Active: `8/55`.
- Mission 2 launch:
  - Damage carried exactly from Mission 1 post to Mission 2 launch.
  - In-mission Damage Control showed Canberra as Moderate with Slight flooding.
- Post-Mission 2:
  - Active and `TaskForceMode_Mission4_Post`: `16/60`.
  - Coontz/Preble stayed Light.
  - Adelaide/Canberra stayed Moderate.
  - Takatsuki stayed No Damage.
  - Mission 1 post to Mission 2 launch state was byte-identical except timestamps.

## Common Bug Patterns

### Future Mission Uses Stale Points
Symptom: after Mission 2, Palawan launch starts from `8/55` instead of `16/60`.

Likely cause:
- Builder or Status opened for an incomplete future mission seeded state from an old mission launch/post snapshot instead of `CurrentTaskForce`.

Expected behavior:
- Incomplete future missions prefer `[PersistentData] CurrentTaskForce`, hydrated by active snapshot detail.
- Completed mission replay uses frozen `TaskForceMode_MissionN_Launch`.

### Rewards Apply Twice
Symptom: Active points/cap jump too high on load.

Likely cause:
- A post snapshot already reflected rewards but lacked `CompletionRewardsApplied=True`.

Expected behavior:
- Load-time reconciliation detects already-reflected rewards, stamps the guard, and does not add again.

### Rewards Do Not Apply
Symptom: post snapshot and active remain at old points/cap after mission complete.

Likely cause:
- Reconciliation thinks rewards are already reflected because it compares against the wrong launch snapshot.

Expected behavior:
- For cap rewards, compare post cap against previous post cap plus reward.

### Damage Shows Too Severe
Symptom: lightly/moderately damaged units show Heavy.

Likely cause:
- Child systems with `CurrentIntegrity=0` are being allowed to dominate overall row summary.

Expected behavior:
- Overall Status damage should primarily reflect compartment/flooding summary. Destroyed child systems can appear in a future Damage Report, but should not always make the whole hull Heavy.

### Repaired Unit Still Damaged
Symptom: repair spends points but launch still shows old damage.

Likely cause:
- Repair cleared only temp or only save file state, or launch materialized an older canonical/native state.

Expected behavior:
- Repair clears latest canonical/native damage state in temp and the campaign save file when present.

## Debug Queries
Useful PowerShell checks:

```powershell
Select-String -Path "$env:USERPROFILE\AppData\LocalLow\Triassic Games\Sea Power\saves\campaigns\001.sav" `
  -Pattern "^\[TaskForceMode_|^\[Mission[0-9]+\]|AvailablePoints|CapPoints|CompletionRewardsApplied|FloodingCompartments|CurrentIntegrity|IsDestroyed"
```

```powershell
Select-String -Path "$env:USERPROFILE\AppData\LocalLow\Triassic Games\Sea Power\saves\campaigns\001.sav" `
  -Pattern "^\[TaskForceMode_Mission[345]_(Launch|Post|LaunchState_|PostState_)|AvailablePoints|CapPoints|FloodingCompartments|LoadedAmmunitions"
```

## Design Rules
- `[PersistentData] TaskForceAvailablePoints` is the spendable master value.
- `[PersistentData] TaskForcePointCap` limits how high available points can rise from rewards.
- Do not introduce alternate point-credit counters.
- Do not use `CampaignRepair`; Task Force Mode repair is paid and explicit.
- Keep campaign roster and point masters under `[PersistentData]`; keep detailed snapshots under `TaskForceMode_*`.
- Treat native `Mission_*` sections as compatibility bridge state only.
- Completed mission replay is read-only.
- Exit/abort has no campaign consequence.
