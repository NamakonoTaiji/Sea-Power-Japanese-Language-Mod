# Task Force Mode Parked Items

- Dynamic flagship text tokens for generated mission INIs.
  - Allow mission authors to write placeholders such as `{TaskForceFlagshipName}`, `{TaskForceName}`, and `{TaskForceFlagshipFullName}` in mission messages, objectives, labels, and localized text.
  - Replace those tokens at Task Force Mode mission-generation time, after copying the authored mission INI to the temporary generated mission INI and before launch.
  - Intended use:
    - `Taskforce1FlagshipmustsurviveMessage=MISSION FAILED|Defeat! {TaskForceFlagshipName} has been sunk!|Exit mission`
    - `Objective_FlagshipMustSurvive={TaskForceFlagshipName} must survive`
  - Keep this campaign-side only; do not require SceneCreator or general mission loader changes.

- Briefing XAML task force substitution.
  - The prototype that rewrote briefing XAML at campaign-map runtime has been removed from the active MVP.
  - It experimented with replacing the `TO` field with the current flagship and replacing authored player-unit summary boxes with generated task force boxes.
  - Revisit later only if we design a proper author-facing token system for briefings, preferably sharing the same token replacement approach as mission objectives/messages.
  - Avoid hard-coded text matching such as `CTG 77.3 USS SCOTT DDG-995`; any future version should be data-driven and campaign-author friendly.

- AAR / debrief Task Force report.
  - Parked for now.
  - Keep Task Force Mode post-mission results inside the campaign-view popup for MVP.
  - The popup is the intended home for crew proficiency changes, command point rewards, command cap increases, and later logistics/damage summaries.
  - A future AAR/debrief version could add a dedicated Task Force Report section, but should not be needed for the current campaign-side implementation.

- Optional no-flagship Task Force Mode.
  - Current Pacific Strike Task Force Mode requires a flagship and uses flagship survival as the core player-force survival gate.
  - A future campaign option could add `RequireFlagship=True/False` under `[TaskForceMode]`.
  - If false, campaign runtime should skip flagship validation, auto-selection, briefing substitution, flagship-only repair/decommission restrictions, and flagship survival launch checks.
  - The Task Force Builder should also hide flagship-specific UI: flagship summary text, flagship stars, "Switch to flagship", and flagship-facilities messaging.

- Loaner unit difficulty ratings and variable rewards.
  - Some command-assigned loaner units may be much better or worse for a restricted side mission.
  - Future mission authoring could give each loaner a difficulty/star rating and optionally different completion rewards.
  - This would let the campaign present a fun risk/reward choice: take the rougher boat for a bigger payoff, or pick the safer unit for a smaller reward.

- Dedicated Task Force Status tile UI.
  - Current `Task Force Status` opens the builder/status management surface.
  - A future window should provide a faster, read-only tile view of the active task force: hulls, submarines, aircraft pools, damage, repairs, flagship, and assigned aircraft.
  - The status UI should not feel like a purchase screen. It should be a calm operational dashboard: current force, readiness, damage, aviation pools, and upcoming mission eligibility.

- Service Record expansion.
  - The first Service Record onboarding window now owns commander nation/name setup and one-way commit state.
  - Future work should add the fuller record view: ribbons, mission history, kills, losses, promotion progress, and profile imagery.
  - Mission kill/loss export should be implemented in a new dedicated class rather than folded into existing mission or debrief files.

- Dedicated loaner unit selection tile UI.
  - Loaner unit runtime and UI support has been stripped from the active MVP until the design is clearer.
  - A future window should present mission-local loaners as clear tiles with profile images, flags, capability summaries, and later difficulty/reward ratings.
  - Loaners should come from the broader game database, not from `player_task_force_roster.ini`.
  - The UI should make it clear that loaners are temporary command assets and will not join the persistent task force.

- Campaign bottom bar / command surface.
  - The campaign map is accumulating several related UI surfaces: event timeline, mission modal, Task Force Builder, future Task Force Status, service record, event log, formation access, and air operations.
  - Fresh-branch status: the first bottom bar shell is being introduced in the active linear campaign view with empty `Task Force Status`, `Task Force Builder`, and `Service Record` buttons.
  - Free-event overlays now leave spacing above the bottom command bar, and the unused `LinearCampaignMapScreen` view has been removed so future work targets the active campaign view.
  - Parked work is now the real command wiring and any expanded command surface beyond those placeholder buttons.
  - Likely later buttons: Timeline toggle, Air Ops, Event Log, formation access, and selected-mission briefing/intel.
  - The mission modal should stay focused on mission-specific details and launch state, while the bottom bar owns campaign-wide management tools.

- Localization review.
  - Task Force Builder strings now have `[TaskForceMode]` localization keys across the shipped language folders, including button tooltips and debug labels.
  - Non-English values are machine translated for now.
  - Parked work is native-speaker review, consistent military terminology, and checking text fit in all supported languages after the Builder/Status UI settles.

- Dedicated land-based aviation management.
  - Current MVP keeps aviation in the pooled Aircraft/Aviation tabs.
  - Later missions with a campaign airfield, such as Broom Airbase, may need a separate view for land-based aviation assignment, readiness, and transfer.
  - Aircraft not assigned to carriers or flight-deck hosts should appear under the active campaign airfield when one exists.

- Storage and ammunition management.
  - Current MVP uses mission-section flags for repair/rearm/reset behavior.
  - Future versions could make ammunition and repair a task force logistics problem, spending command points to partially rearm or repair units between missions.
  - Base/refit missions could still grant full automatic replenishment.

## Morning UI Concepts

These are starting points for the next design pass.

### Task Force Status UI

Purpose: quick operational readout after the force has been committed. This is not the builder and should not look like a shopping cart.

Suggested layout:

- Header:
  - task force name
  - current command points and cap
  - flagship line with star
  - next mission eligibility warning if relevant
- Left/main tile grid:
  - one tile per ship/submarine
  - flag, name, hull/class, profile image
  - readiness badge: Ready, Repairing, Damaged, Destroyed
  - crew proficiency
  - hangar summary if applicable
  - assigned aviation mini-icons under the host ship
- Right side summary rail:
  - Aviation Pool: squadron rows with Available / Assigned counts
  - Repairs: units unavailable this mission
  - Damage: units needing repair/decommission action
  - Loadout Unlocks: currently unlocked reward loadouts
- Footer:
  - Manage Task Force opens the builder if purchases or repair actions are available
  - Close

Design notes:

- Use compact cards rather than large inspector panels.
- Make damaged/repair states visually obvious without overwhelming the map.
- Flagship should always sort first and carry the white star beside the name.
- Keep aircraft assignment details visible under ships, because that is where players expect to verify deck usage.
- This UI should be safe to open at any time, including after checkout and on missions where purchases are disabled.

### Parked Loaner Unit Selection UI

Purpose: let a side mission proceed when the player lacks a qualifying owned unit.

Suggested layout:

- Header:
  - mission title
  - requirement text, such as `Requires: Submarine` or `Requires: Vessel - Cruiser, Destroyer, or Frigate`
  - explanation that command can assign a temporary unit for this mission only
- Tile list:
  - one tile per loaner
  - flag, name, class/type, profile image
  - short capability line, e.g. torpedoes, missiles, sonar/radar, helicopter capacity
  - crew proficiency fixed to mission default or campaign-authored value
  - temporary/loaner badge
- Selected loaner preview:
  - larger profile image
  - class/type
  - loadouts available for this mission
  - warning: does not join persistent task force, does not gain campaign progression
- Footer:
  - Select Loaner
  - Back

Design notes:

- Do not reference points or purchase costs unless we later add difficulty/reward tradeoffs.
- Keep this separate from the player roster to reinforce that loaners are not owned.
- Later difficulty ratings can add stars and adjusted rewards, but that is parked for now.
