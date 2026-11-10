# pylandgraze

**Where the herd should be, hour by hour and season by season, computed from the
landscape rather than from a timetable.**

> Status: **concept stage.** This README is the specification and the evidence
> behind it. No allocation code yet — the geometry and scheduling layers are
> buildable now, the forage layer is blocked on a soil water balance.

Virtual fencing removed the labour cost of moving a fence. It did not answer the
question that matters: *where should the fence be, and when?* Collar platforms let
you draw a line and set a pace. Deciding the line is still done by eye.

That decision has a terrain answer, and it changes through the day. The ridge that
holds frost at dawn is the one carrying dry fibre; the north slope at peak
afternoon sugar is also the one at peak radiation and peak water demand; the
sheltered gully that protects stock overnight is measurably the least windy ground
on the property. **Each shift in a daily rotation is an argmax over a layer we
already compute.** That is the thesis of this package.

## Where this sits

| package | reads | gives you |
|---|---|---|
| [pylidarterrain](https://github.com/jborevitz/pylidarterrain) | classified airborne LiDAR | the ground surface: slope, aspect, sky view |
| [pyterrainsol](https://github.com/jborevitz/pyterrainsol) | that surface + daily weather | light and evaporative demand per cell per day |
| [pyterrainwind](https://github.com/jborevitz/pyterrainwind) | that surface + reanalysis wind | wind exposure, and the demand it carries |
| [pychannelform](https://github.com/jborevitz/pychannelform) | that surface | catchments, channels, long profiles |
| [pylandstore](https://github.com/jborevitz/pylandstore) | channels + demand | where to hold water and what it costs to lift |
| **pylandgraze** (this one) | all of the above + a herd | **where the animals should be, and when** |

## What already exists, honestly

Worth stating plainly so nobody here rebuilds it:

- **Commercial collar platforms** — Halter, eShepherd, Nofence, Vence — ship the
  hardware, the containment behaviour and the app. eShepherd already advances a
  fence automatically and offers strip grazing and back-fence modes at a pace the
  operator sets. **Automated strip advance is a solved, shipping product feature
  and is not our contribution.**
- **OpenFence** (GPL-3) is an open-source GPS collar and web interface. It is an
  electrical-engineering project about containment, not an allocation model.
- The **virtual-fencing research literature** is healthy and mostly about animal
  welfare, learning and whether containment works — not about where to put the line.

What is missing, as far as a literature search can establish, is anything that
computes the plan **from the landscape**. The platforms execute a fence; they do not
tell you what the pace should be, which part of the farm to be on at 07:00 versus
14:30, or how deep today's strip should be given what the grass did last week. The
microclimate models that could answer that are built for ecology, where terrain and
vegetation are fixed boundary conditions rather than things a manager changes.

**So the seam is: platforms own execution, ecology owns the physics, and nobody owns
the decision.**

## The core relationship

Everything reduces to one equation:

```
return interval (days) = length of the part along its grazing axis / strip depth
```

Strip depth is the control variable. It sets how long the ground rests, and rest
is what regrowth needs. Measured on the Avonside parts, with a 150 m working front:

| part | area | along axis | across | strips at 25 m |
|---|---|---|---|---|
| Top farm | 11.6 ha | 688 m | 220 m | 27.5 |
| Upper forest | 10.2 ha | 654 m | 249 m | 26.1 |
| Northerly woodland | 11.8 ha | 715 m | 279 m | 28.6 |
| Lower forest flats | 12.7 ha | 600 m | 304 m | 24.0 |

A 25 m strip gives a ~27-day return interval on the two open parts — within a
couple of days of a typical spring regrowth cycle, which nobody tuned.

## Two seasons, two different problems

This is the thing most grazing schedulers get wrong, and it follows directly from
the equation above.

**Growing season** is a *rotation*. Allocate against the regrowth rate; the binding
constraint is that the return interval must not be shorter than recovery.

**Dormant season is not a rotation at all.** It is single-pass depletion of standing
stockpile, and the binding constraint is total remaining feed divided by daily
demand. Deepening the strips to "make the feed last" does the opposite — it shortens
the return interval. On a 688 m part, a 75-day return would need 9 m strips, and a
9 x 150 m strip would have to carry 3,370 kg DM/ha to feed a hundred head, which
dormant hill country does not.

Same geometry engine, different objective function. Implementing winter as a flag
on the summer model hides the distinction.

## The diurnal layer

The part of this that is genuinely new. Each shift is a query against an existing
raster, not a position on a clock:

| shift | what the animal needs | the layer that answers it |
|---|---|---|
| dawn | dry fibre, rumen fill | exposure and first-sun timing on the ridge |
| afternoon | peak soluble sugars | peak radiation on the north-facing slope |
| night | thermal shelter | minimum wind exposure |
| midday | rest, no soil damage | slope exclusion; wide communal polygons, never strips |

At Avonside these resolve to different parts, and the wind field picks the same
night-shelter ground a person would choose by eye: the upper forest is the most
sheltered part on the property and carries 27 mm/yr *less* evaporative demand than
the farm mean, while the exposed top farm carries 11 mm/yr more.

## Constraints that are not negotiable

- **Steep ground is never strip-grazed.** Concentrating stock on 40%+ slopes
  terraces and erodes them. Rest zones there stay wide and communal.
- **Residual is not waste.** Grazing to bare ground on 29% median grade with gully
  walls at 100% is how erosion starts. Any "harvest it all" objective must be
  bounded by a minimum residual.
- **A plan is proposed, not executed.** Virtual fencing for cattle became legal in
  NSW in December 2025 under welfare conditions that require training and ongoing
  monitoring — explicitly not set-and-forget. This package emits a plan for a human
  to approve.
- **Virtual fencing acts only on collared stock.** It does nothing to wild deer or
  any other competitor, and no polygon geometry changes that.

## Herd size is an output, not an input

A fixed herd is the wrong starting point: carrying capacity depends on actual
rainfall, on how much of the block is trees versus grass, and on where water is
moved to — all of which are management decisions rather than constants. The budget
arithmetic therefore runs backwards, from what the land grows to what it carries.

For scale, on the 23.5 ha of currently open ground at 537 mm/yr, year-round capacity
ranges from about 21 head at 8 kg DM/ha/mm of rain-use efficiency to 53 head at 20.
That span is the point: the efficiency term is the management lever, not the herd.

## Planned shape

```
parts.py      polygon classes from terrain, or supplied
forage.py     standing biomass and regrowth      <- blocked on the water balance
strips.py     advance a front across a polygon, honouring exclusions
diurnal.py    the argmax layer: which polygon serves each shift
budget.py     demand against supply, in both seasonal modes
export.py     timestamped GeoJSON / KML
```

`export.py` targets a **file format, not a vendor API**. No collar platform
publishes an open polygon-ingest endpoint, and writing to an interchange format
keeps the output portable across Halter, eShepherd, Nofence and Vence.

`forage.py` is the honest blocker. Atmospheric demand is finished; supply needs the
soil water balance, which is the TOPMODEL work still outstanding.

## Data

`data/farm_part_id.npy` — the four Avonside management parts on the 3 m grid, as an
integer raster (-1 outside the property, 0-3 for Top farm, Upper forest, Northerly
woodland, Lower forest flats). Terrain itself comes from
[`pylidarterrain`](https://github.com/jborevitz/pylidarterrain).

## Methods

Allocation arithmetic, the seasonal-mode distinction and the traps are in
**[`docs/methods.md`](docs/methods.md)**.
