# World Simulation

A browser-based sandbox for configuring and observing an evolving world represented on an interactive globe.

## Language

**World**:
The complete evolving state of one simulation, including its countries, global settings, relationships, conflicts, and simulated time.
_Avoid_: Game state, campaign

**Globe**:
The navigable three-dimensional geographic representation through which the World Operator inspects the World.
_Avoid_: World, map

**World Operator**:
The human who observes the entire simulation, configures world and country state, controls simulated time, and may intervene anywhere without representing a particular country or civilization.
_Avoid_: Player country, national leader, ruler

**Country**:
An autonomous simulated actor with geographic territory, configurable state, relationships, and behavior. A Country may remain at peace, initiate conflict, or have its territory changed by conflict.
_Avoid_: Player, faction

**Country Trait**:
An operator-configurable quality that influences a Country's autonomous behavior, such as its aggressiveness or inclination toward peace.
_Avoid_: Country action, fixed rule

**Geographic Region**:
A stable, high-resolution area whose ownership can transfer between Countries. Adjacent regions with the same owner are rendered as a smooth combined border.
_Avoid_: Runtime polygon cut, country

**Conflict**:
A strategic contest between Countries that unfolds over continuous simulated time and may transfer Geographic Regions. A Country may participate in several Conflicts simultaneously.
_Avoid_: Battle, tactical match

**Scenario**:
The initial configuration from which a World begins, including world-level settings and country-level starting values.
_Avoid_: Save, world state

**Intervention**:
A direct change the World Operator makes after a World has begun. Interventions occur while simulated time is paused.
_Avoid_: Turn, country action
