# Task Force Mode Damage Report

## Summary
Add a campaign-side Damage Report command for Task Force ships and submarines. The report opens from TF Status right-click menus and shows a read-only summary of saved unit damage from the canonical Task Force Mode save state.

## Intended Behavior
- Add `Damage Report` to ship/submarine context menus in TF Status.
- Do not show the command for aircraft or helicopters.
- Open a static report window, not the live in-mission Damage Control UI.
- Read damage from the latest saved Task Force unit state through `LinearCampaignTaskForceSaveManager`.
- Show unit identity, overall damage, hull/compartment damage, flooding, and damaged/destroyed systems.
- Show `No damage reported.` when no saved damage exists.
- Repair remains handled by TF Status repair actions, not inside this report.

## Notes
The live `DamageControlViewModel` expects an active `ObjectBase` and live `Compartments`, so reusing it directly on the campaign map is risky. The preferred implementation is a campaign-safe summary-only viewmodel/XAML that uses saved INI state.

## Test Cases
- Damaged ship opens a report with damaged systems listed.
- Undamaged ship opens a no-damage report.
- Repaired ship updates to no damage.
- Unrepaired damage persists across missions and remains visible.
- Aircraft/helicopter menus do not include Damage Report.
