# pylandgraze — methods

Conventions, arithmetic and traps. Findings from the block this was developed on
are in the [README](../README.md).

## The allocation arithmetic

Three quantities, and every scheduling decision is a relation between them.

- **Return interval** = part length along the grazing axis / strip depth.
- **Daily area** = strip depth x working front width.
- **Required standing biomass** = herd daily demand / daily area.

The third is the one that catches people. Choosing a strip depth fixes both how
long the ground rests *and* how much biomass must be standing for the allocation to
feed the herd, and those pull in opposite directions: deeper strips rest the ground
less but demand less standing feed per hectare. There is no depth that optimises
both, which is why depth is a decision rather than a parameter to fit.

Worked, on a 150 m front and a 688 m part, for 35% of a 1,300 kg DM/day herd demand:

| depth | return interval | standing biomass required |
|---|---|---|
| 9 m | 76 days | 3,370 kg DM/ha |
| 15 m | 46 days | 2,022 kg DM/ha |
| 25 m | 28 days | 1,213 kg DM/ha |
| 40 m | 17 days | 758 kg DM/ha |

## Growing season and dormant season are different objective functions

Do not implement the dormant season as the growing season with a longer cycle.

- **Growing season — a rotation.** Allocate against regrowth rate. Constraint:
  return interval >= recovery time. Strip depth is chosen to make the rotation fit
  the recovery period.
- **Dormant season — depletion.** Nothing is regrowing, so there is no rotation to
  close. Allocate against standing stock. Constraint: remaining feed / daily demand
  = days until empty. Strip depth is chosen to meet daily intake, and the plan ends
  when the stockpile does.

The trap: "expand the strips in winter" is intuitive and backwards. Deepening
strips shortens the return interval, because the part has a fixed length. It only
makes sense once you realise winter grazing is single-pass, at which point the
return interval is not a constraint at all.

## Rain-use efficiency is the management lever

Carrying capacity should be derived, never assumed. The chain is:

```
annual rainfall x rain-use efficiency x grazeable area = annual dry matter
annual dry matter / (intake per head per day x 365) = year-round head
```

Rain-use efficiency is where tree-to-grass conversion, water movement, soil
amendment and grazing management all land. Reporting a stocking rate without the
efficiency term it assumed is reporting an opinion.

## Diurnal allocation is a query, not a timetable

Resist hard-coding clock times to places. Each shift is defined by what the animal
needs and resolved against a raster:

- fibre at dawn -> exposure and the timing of first direct sun
- sugars in the afternoon -> peak radiation on the day's solar field
- shelter at night -> minimum wind exposure
- rest at midday -> slope exclusion mask

The same four queries run anywhere with a DEM and a climate record. The timetable
is the *output*, which is what makes the method site-independent.

## Constraints to enforce in code, not in documentation

- **Slope exclusion.** Strip grazing must be impossible above a configurable grade.
  Rest polygons on steep ground stay wide and communal.
- **Minimum residual.** Any objective that maximises harvested fraction must be
  bounded below by a residual floor, or it will recommend bare ground.
- **Approve, do not act.** Emit a proposed plan with shift times. Jurisdictions
  permitting virtual fencing generally require training and ongoing monitoring;
  an autonomous overnight controller is the pattern those rules exist to prevent.
- **Collared stock only.** Virtual fencing influences the animals wearing collars
  and nothing else. Any feature premised on steering wildlife is mis-specified.

## Export, not integration

Write timestamped polygon sequences to an interchange format (GeoJSON or KML).
No collar vendor publishes an open polygon-ingest API, so an integration would be
to a private interface that may not exist; a file is portable across vendors and
costs nothing if an endpoint appears later.

## What this does not do

- **No animal behaviour model.** Where animals actually go within a polygon,
  how fast they learn a boundary, and their welfare response are empirical
  questions this package does not answer.
- **No forage growth model yet.** Standing biomass and regrowth require a soil
  water balance that does not exist in this stack. Until it does, any biomass
  figure here is a placeholder, and anything downstream of it is geometry.
- **No animal nutrition model.** Diet quality targets are taken as inputs.
- **No wildlife management.** Out of scope, and partly out of scope legally.
