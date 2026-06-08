# Task Force Mode Repairs

Task Force Mode repairs are paid campaign-management actions for ships and submarines. They let the player spend task force points to clear survivable damage before a mission when repair facilities are available.

## Player Behavior

- Light and Moderate damage can be repaired.
- Heavy damage cannot be repaired in theater and should use Decommission instead.
- Destroyed units are removed from the task force.
- Repair spends `[PersistentData] TaskForceAvailablePoints`.
- Repair does not change `[PersistentData] TaskForcePointCap`.
- Repair immediately changes Task Force Status damage to `No Damage`.

## Campaign Configuration

Repair rules live in the campaign `campaign.ini` `[TaskForceMode]` section:

```ini
; REPAIR RULES
DamageToAllowRepair=Light,Moderate
DamageToDisallowRepair=Heavy
RepairPointsCost=Light,0.1|Moderate,0.25
```

`DamageToAllowRepair` lists damage states that can be repaired if repair facilities are available.

`DamageToDisallowRepair` lists damage states that cannot be repaired. For Pacific Strike Task Force Mode, Heavy damage represents damage that requires large shipyard facilities outside the campaign area.

`RepairPointsCost` maps damage state to a fraction of the unit's original point cost.

## Cost Math

Repair cost is calculated as:

```text
Ceiling(original unit cost * damage modifier)
```

The minimum repair cost is `1` point.

Original unit cost uses `BaseCost` when present. If `BaseCost` is unavailable, repair uses `Cost`.

Examples with the default Pacific Strike rules:

- A 25-point unit with Light damage costs `Ceiling(25 * 0.1) = 3` points.
- A 25-point unit with Moderate damage costs `Ceiling(25 * 0.25) = 7` points.

## Saved State Behavior

Repair edits the saved campaign-state section for the selected unit. The unit is identified by its task force `InstanceId`, which is written to generated mission units as `CampaignTag`.

The latest prior state section is located by searching backward from the selected mission:

```text
Mission_<missionIndex>_<InstanceId>
```

Repair preserves ammunition and flight deck ammunition state. It only clears damage-related state.

The unit state section clears these keys when present:

- `IsDestroyed`
- `IsSinking`
- `FloodingCompartments`
- `SystemCompartments`
- `CompartmentsDelayedSinkTime`

Child system state sections matching the unit state prefix clear these keys when present:

- `CurrentIntegrity`
- `Inoperable`

After repair, Task Force Status should show the unit as `No Damage`. It should not show a `Repaired` badge while still displaying damage.

## Interactions

`TaskForceModeRepair=True` on a mission means repair facilities are available before that mission. It does not mean repairs are free.

`TaskForceModeRepair=False` means repair facilities are not available before that mission, so repair actions should be disabled.

`TaskForceModeRearm=True` remains separate from repair. Rearm does not cost points in the current economy.

Aircraft and helicopters do not use repair rules.

Dismissal and Decommission are separate management actions:

- Dismissal removes a unit from the task force and returns a configured fraction of its original cost.
- Decommission is the heavy-damage management path for ships and submarines that cannot be repaired in theater.
