# Task Force Builder Radar Stats Shelf

Date shelved: 2026-05-27

This note preserves the Task Force Builder statistics/radar-chart work that was removed after art direction review. The feature was functional, but the visual style did not fit the game.

## What It Did

- Added a compact `Statistics` panel in Task Force Builder Unit Details.
- Displayed six 1-10 capability values as both a list of bars and a six-axis radar polygon.
- Read author-facing values from the unit INI `[AI]` block rather than inferring values from weapons at runtime.
- Used Noesis-supported `Path` geometry for the radar polygon, after `Polygon` failed in Noesis XAML.
- Bound `ProgressBar.Value` with `Mode=OneWay` because the stat percent property was read-only.
- Gave zero-valued radar axes a visual `1/10` floor so sparse profiles did not collapse into a spike, while still displaying `0` in the numeric list.

## C# Shape

The implementation lived in:

- `Assets/Scripts/UI/Linear Campaign/ViewModels/LinearCampaignTaskForceUnitLookup.cs`
- `Assets/Scripts/UI/Linear Campaign/ViewModels/LinearCampaignTaskForceBuilderViewModel.cs`

It added these view-model types/properties:

```csharp
public class LinearCampaignTaskForceCapabilityStat
{
  public string Label { get; }
  public int Value { get; }
  public double PercentValue { get; }
  public string Summary { get; }
}

public class LinearCampaignTaskForceCapabilityProfile
{
  public ReadOnlyCollection<LinearCampaignTaskForceCapabilityStat> Stats { get; }
  public string RadarPathData { get; }
}
```

`LinearCampaignTaskForceBuilderRosterItem` exposed:

```csharp
public LinearCampaignTaskForceCapabilityProfile CapabilityProfile { get; }
public ReadOnlyCollection<LinearCampaignTaskForceCapabilityStat> CapabilityStats => CapabilityProfile?.Stats;
public string CapabilityRadarPathData => CapabilityProfile?.RadarPathData ?? string.Empty;
```

The profile was resolved with:

```csharp
CapabilityProfile = LinearCampaignTaskForceUnitLookup.GetCapabilityProfile(unitFolder, unitType);
```

The radar path used a six-point `M... L... Z` path string:

```csharp
double angle = -Math.PI / 2.0 + i * 2.0 * Math.PI / stats.Count;
double chartValue = stats[i].Value <= 0 ? 1.0 : stats[i].Value;
double statRadius = radius * Math.Max(0, Math.Min(10, chartValue)) / 10.0;
```

## Stat Categories

Surface ships:

- `ASW_Capability`
- `ASuW_Capability`
- `AAW_Capability`
- `PointDefense_Capability`
- `LandAttack_Capability`
- `AirComplement_Capability`

Aircraft and helicopters:

- `SurfaceRadar_Capability`
- `AirRadar_Capability`
- `EW_Capability`
- `ASW_Capability`
- `ASuW_Capability`
- `LandAttack_Capability`, displayed as `Strike`

Submarines:

- `Stealth_Capability`
- `Sonar_Capability`
- `ASW_Capability`
- `ASuW_Capability`
- `LandAttack_Capability`
- `Endurance_Capability`

## XAML Shape

The XAML lived in:

- `Assets/Scripts/UI/Linear Campaign/Views/LinearCampaignTaskForceBuilderView.xaml`

It added:

- `TaskForceBuilderCapabilityStatTemplate`
- A right-column `Statistics` bordered panel above the selected unit description
- A 136 x 124 `Canvas`
- Six radial `Line` elements
- Outer and inner hex-grid `Path` elements
- One filled `Path Data="{Binding CapabilityRadarPathData}"`
- An `ItemsControl ItemsSource="{Binding CapabilityStats}"`

Important binding detail:

```xml
<ProgressBar Value="{Binding PercentValue, Mode=OneWay}" />
```

## Data Pass Notes

A bulk aircraft data pass inserted missing capability keys into actual aircraft, helicopter, and VTOL unit definition files under:

```text
Assets/StreamingAssets/original/aircraft
```

It skipped:

- `*_squadrons.ini`
- `shared_settings.ini`

It did not overwrite existing authored values. Preexisting `TBD` values were left intact.

Notable authored/support-aircraft examples:

```ini
# usn_e-2c
SurfaceRadar_Capability=5
AirRadar_Capability=9
EW_Capability=3
ASW_Capability=0
ASuW_Capability=0
LandAttack_Capability=0

# usaf_e-3a
SurfaceRadar_Capability=4
AirRadar_Capability=10
EW_Capability=4
ASW_Capability=0
ASuW_Capability=0
LandAttack_Capability=0

# usn_ea-6b
SurfaceRadar_Capability=1
AirRadar_Capability=2
EW_Capability=9
ASW_Capability=0
ASuW_Capability=0
LandAttack_Capability=2
```

After review, generic helicopters were corrected back to:

```ini
EW_Capability=0
```

unless explicitly `EW` or `ESM` role aircraft.

## Localization Keys

The feature added these `TaskForceMode` UI keys across `language_*` files:

```ini
CapabilityStatistics=Statistics
CapabilityASW=ASW
CapabilityASuW=ASuW
CapabilityAAW=AAW
CapabilitySurfaceRadar=Surface Radar
CapabilityAirRadar=Air Radar
CapabilityEW=EW
CapabilityStrike=Strike
CapabilityStealth=Stealth
CapabilitySonar=Sonar
CapabilityEndurance=Endurance
CapabilityPointDefense=Point Defense
CapabilityLandAttack=Land Attack
CapabilityAirComplement=Air Complement
```

## Verification Before Shelving

Before removal, `dotnet build Seapower-Scripts.csproj --no-restore -v:minimal` succeeded with `0 Error(s)` and the repo's existing warning set.
