# Linear Campaign Save Flow

This note explains how the linear campaign save path works in Sea Power, including the temporary campaign state file and the campaign `.sav` files.

## Active Campaign Path

`SaveLoadManager.GetGlobalCampaignSaveFilePath()` points at the currently active linear campaign file.

At campaign start, this is usually the authored campaign file, for example:

```ini
Assets/StreamingAssets/original/campaigns/pacific-strike-task-force/campaign.ini
```

After the player saves the campaign, this can point at a writable campaign `.sav` file under the user save folder.

## Runtime Temp File

During campaign play, the game uses a temporary campaign state file:

```text
<Application.persistentDataPath>/saves/campaigns/linear_campaign.tmp
```

This path is stored in:

```csharp
Globals._campaignTempFilePath
```

The temp file stores runtime mission/campaign carry-forward state, especially sections named like:

```ini
[Mission_3]
[Mission_3_<CampaignTag>]
[Mission_3_CampaignVariables]
[Mission_3_TaskForce]
[Mission_3_TaskForceUnit1]
```

Task Force Mode mission snapshots are written into this temp file before they are folded into an actual campaign save.

## Loading A Campaign

When a linear campaign loads:

1. `LinearCampaign` parses the current campaign file from `SaveLoadManager.GetGlobalCampaignSaveFilePath()`.
2. It clears/recreates `linear_campaign.tmp`.
3. If the loaded campaign file is a save, detected by `[File] Version`, it copies all sections beginning with `Mission_` from the save into `linear_campaign.tmp`.

This means the temp file is rebuilt from the durable `.sav` when loading a saved campaign.

## Saving From The Campaign Map

`LinearCampaign.SaveCampaign(filePath)` writes a durable campaign save file.

It writes campaign metadata:

```ini
[File]
Version=...
CreationTime=...
```

It writes event state:

```ini
[Mission1]
IsUnlocked=True
IsComplete=True
```

It writes campaign-scoped persistent data:

```ini
[PersistentData]
TaskForceCommanderCommitted=True
TaskForceCommanderNation=Japan
TaskForceCommanderName=Hiroshi Watanabe
```

Then it opens `linear_campaign.tmp` and copies every temp section into the save file. This makes the `.sav` the durable version of:

- mission unlock/completion state
- campaign persistent data
- carried mission unit state
- Task Force Mode snapshots
- campaign variables and tags

## Saving During A Tactical Mission

When saving during a tactical mission, the normal mission save is written first.

If a linear campaign is active, the mission save also creates a companion campaign save file:

```text
<mission_save_name>_campaign.ini
```

The mission save stores the companion campaign save reference:

```ini
[File]
LinearCampaignSavePath=<campaign companion filename>
LinearCampaignEventName=<current campaign event>
```

So an in-mission save has two related files:

- the tactical mission `.sav`
- a companion linear campaign save file beside it

The tactical mission save points back to the companion campaign save through `LinearCampaignSavePath`.

## Persistent Data

`LinearCampaign.Instance.PersistentData` is in-memory campaign-scoped data.

When saved, it is written into:

```ini
[PersistentData]
```

When the campaign loads, `[PersistentData]` is read back into `LinearCampaign.Instance.PersistentData`.

Task Force Mode uses this for commander setup fields such as:

```ini
TaskForceCommanderCommitted=True
TaskForceCommanderNation=Japan
TaskForceCommanderFirstName=Hiroshi
TaskForceCommanderLastName=Watanabe
TaskForceCommanderName=Hiroshi Watanabe
TaskForceCommanderRankLevel=5
```

Task Force Mode also stores the master campaign task-force values here:

```ini
CurrentTaskForce=usn_dd_spruance,Variant2|usn_ff_knox,Default
TaskForceAvailablePoints=34
TaskForcePointCap=50
```

`CurrentTaskForce` is the campaign roster source. `TaskForceMode_*` sections keep the detailed snapshots used by Status, launch, replay, and debrief; Builder points come only from the persistent point keys.

## Short Version

- `campaign.ini` is the authored/base campaign definition.
- `linear_campaign.tmp` is the live scratchpad for mission and task-force carry-forward state.
- campaign `.sav` is the durable campaign save that copies in `PersistentData`, mission unlock/completion state, and temp `Mission_` sections.
- tactical mission `.sav` can create a companion campaign save and reference it with `LinearCampaignSavePath`.
