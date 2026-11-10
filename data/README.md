# Data

## `farm_part_id.npy`

The four management parts of 845 Avonside on the 3 m grid, as an integer raster
over the catchment window:

| value | part | area |
|---|---|---|
| -1 | outside the property | |
| 0 | Top farm | 11.6 ha |
| 1 | Upper forest | 10.2 ha |
| 2 | Northerly woodland | 11.8 ha |
| 3 | Lower forest flats | 12.7 ha |

Boundaries follow terrain features rather than straight cuts, and were derived in
[`pylidarterrain`](https://github.com/jborevitz/pylidarterrain): a slope break, the
southern drainage creek, and the eastern ridge crest. Total 46.3 ha.

Measured in the farm's own axis frame (long axis bearing 108 degrees), the parts
span 600-715 m along the axis and 220-304 m across, which is what makes a 150 m
working front and a 25 m strip depth feasible on the two open parts.

The elevation surface itself is not duplicated here; build it from the ELVIS
LiDAR tiles with `pylidarterrain`.
