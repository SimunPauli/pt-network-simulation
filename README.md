# pt-network-simulation

Public transport choice sets from OpenTripPlanner (OTP). For a trip (origin, destination, departure time) it
generates a set of distinct public transport itineraries: a plain OTP search, plus re-queries with stops or routes
excluded, with other generalized costs and with only some transit modes. Too many alternatives is intended; pruning
is left to the model. Written for route choice modelling on the Danish National Travel Survey (TU), but it takes
plain trips, so any travel survey can use it through its own adapter.

## Requirements

- Python 3.11+, pandas, numpy, requests.
- A running OTP built from the modified OTP 2.8.1 fork
  ([MiserableArmy/otp_nationalsurvey_gtfs](https://github.com/MiserableArmy/otp_nationalsurvey_gtfs)), with a
  graph covering the dates queried. Upstream OTP does not work: every query asks for the fork's `S_TRAIN` transit
  mode (`otp.client.DEFAULT_TRANSIT_MODES`), and the `exclude_stops` rounds use the fork's
  `excludeStops`/`excludeTrips`.

## Install

```bash
pip install "pt-network-simulation @ git+https://github.com/<owner>/pt-network-simulation@v0.1.0"
```

While developing it alongside a survey adapter: `pip install -e ../pt-network-simulation`.

## Usage

```python
import json
import pandas as pd
from pt_network_simulation import otp, trip_choice_set, generate_choice_sets, choice_set_summary

with open("config.json", encoding="utf-8") as file:
	config = json.load(file)  # see config.example.json

trip = dict(
	trip_id=1,
	origin=otp.client.coordinate_location(55.6655, 12.5130),
	destination=otp.client.coordinate_location(55.6828, 12.5735),
	depart_dt=pd.Timestamp("2025-09-16 07:30", tz="Europe/Copenhagen"),
	access_mode="WALK",
	egress_mode="WALK",
)
choice_set = trip_choice_set(**trip, config=config)

# Several trips, 8 at a time
choice_sets = generate_choice_sets([trip, ...], config, max_workers=8)
print(choice_set_summary(choice_sets))
```

The package is a library with no command line: each survey writes its own adapter that turns its trips (and
chosen itineraries, if any) into `trip_choice_set` arguments.

### `trip_choice_set(trip_id, origin, destination, depart_dt, access_mode, egress_mode, config, chosen_legs=None)`

| Argument | Description |
|---|---|
| `trip_id` | The trip's id, written to the output; with each random kind's `seed`, it seeds that kind's draws |
| `origin`, `destination` | OTP `PlanLocationInput`: `otp.client.coordinate_location(lat, lon)` or `otp.client.stop_location(stop_gtfs_id)` |
| `depart_dt` | Timezone-aware `pd.Timestamp`; itineraries are searched within ± `choice_set.search_window` of it |
| `access_mode`, `egress_mode` | OTP `PlanAccessMode`/`PlanEgressMode`, e.g. `"WALK"` |
| `config` | See "Configuration" |
| `chosen_legs` | Optional: the chosen itinerary, one row per leg (see below) |

### `chosen_legs`

The chosen itinerary becomes `alt_id` 0. Columns:

- Always: `leg_id` (leg order), `mode` (OTP mode), and `start_trip` (the itinerary's start: alternatives are
  ranked by their gap to it instead of to `depart_dt`).
- If any leg is transit (a mode not in `otp.client.STREET_MODES`): `iteration_id` (one value for all its legs)
  and the `alternative_key` columns: `route_short_name`; for `routes_stops` also `from` and `to`; for `trip`
  also `start_leg`. Candidates sharing its key are dropped (the chosen itinerary is that alternative), and a
  `coverage` column is added.
- Optional: `route_gtfs_id`, `from_gtfs_id` and `to_gtfs_id`, added to the pools the `exclude_routes` and
  `exclude_stops` rounds draw from.

Other output columns are kept if present and left empty otherwise (e.g. a car trip's routes). Without
`chosen_legs`, `alt_id`s start at 1 and there is no `coverage` column.

### Output

One row per leg; alternatives in order of their score (minutes between their start and the chosen start, or
`depart_dt` without one, plus their waiting). Itineraries walking at least as far as the direct walk are dropped.

| Column | Description |
|---|---|
| `trip_id` | As given |
| `direct_walk_km` | The direct walk's distance; empty if OTP found none |
| `alt_id` | 0 = chosen, then 1, 2, … in score order |
| `is_chosen` | 1 for the chosen itinerary |
| `coverage` | Only with a transit chosen itinerary: 1 if the search found its `alternative_key` or another alternative of its `line_group`, else 0 |
| `line_group` | Lowest `alt_id` with the same transit legs on the `alternative_key` columns plus `from`/`to`, where bus routes are compared by stops only, metro sibling lines (M1/M2, M3/M4) count as one, and all S-train lines but F count as one |
| `route_group` | Lowest `alt_id` with the same sequence of transit modes and route names |
| `search_round`, `search_variation` | The search round that found it (0 = the plain query) and what it varied (excluded ids, costs, modes) |
| `leg_id`, `mode`, `route_short_name`, `route_gtfs_id`, `trip_short_name`, `trip_gtfs_id`, `bikes_allowed`, `from`, `to`, `from_gtfs_id`, `to_gtfs_id`, `generalized_cost`, `system_notice_tag` | From OTP's legs; `from`/`to` are stop names |
| `start_trip`, `end_trip` | The itinerary's start and end (ISO 8601 with UTC offset) |
| `start_leg`, `end_leg` | The leg's start and end (ms since epoch) |
| `distance_km`, `crow_distance_km`, `duration_min` | Leg distance along the route and as the crow flies; duration in whole minutes |
| `leg_geometry` | Encoded polyline |

OTP's stay-seated (interlined) legs are merged into one leg.

### `generate_choice_sets(trips, config, max_workers=1)`

`trips` is a list of dicts of `trip_choice_set`'s arguments except `config`. Runs `max_workers` trips at a time
(threads; the work is OTP requests) and returns their choice sets concatenated in trip order. Each trip's
output (its choice set and alternative count) is printed in trip order, not interleaved. A trip that raises is
printed as skipped and left out. Each trip's OTP requests carry the correlation ID `trip_id=<trip_id>`.

### `choice_set_summary(choice_sets, groups=None)`

One row for all trips plus, given `groups` (a Series `trip_id` → label), one per label: `trips`,
`covered`/`coverage_share` (empty without a `coverage` column), `no_otp_itinerary` (trips with no OTP
transit itinerary), and the mean and 0–100th percentiles of alternatives besides the chosen per trip.

## Configuration

A JSON config with two sections; `config.example.json` has the settings used for TU. All keys are required
except `request.print_query`.

| Key | Description |
|-----|-------------|
| `request.otp_url` | GraphQL endpoint of the running OTP instance |
| `request.max_itinerary_candidates` | Maximum number of OTP itineraries to fetch per query |
| `request.request_timeout` | HTTP timeout in seconds for OTP requests |
| `request.print_query` | Optional, default `false`: print every query and its variables, ready to paste into GraphiQL |
| `choice_set.search_window` | ISO 8601 duration: time window searched around `depart_dt` |
| `choice_set.alternative_key` | What makes two itineraries the same alternative, compared over transit legs in order: `routes` (mode + route name), `routes_stops` (also boarding/alighting stop names), `trip` (also departure: every departure is its own alternative). Lines sharing track are separate alternatives, grouped in `line_group` (see "Output"). Of each `alternative_key` group the itinerary with the least gap to the chosen start (or `depart_dt`) plus waiting is kept; the chosen itinerary's group is the chosen itinerary itself |
| `choice_set.exclude_stops` | Extra alternatives from re-querying with stops excluded: `steps` run in order, each `rounds` queries excluding `stops_per_round` stop ids (no boarding/alighting, passing through allowed) drawn at random from the boarding/alighting stops of the chosen itinerary and all candidates so far, drawn anew each round but never a set of stops an earlier round excluded (across steps); a round with no such set left is skipped. Draws are seeded per trip by `seed` and `trip_id`, so runs are reproducible. `steps: []` disables it |
| `choice_set.exclude_routes` | As `exclude_stops`, with routes, run after the stop rounds: `steps` of `rounds` queries, each excluding `routes_per_round` route ids (their trips are not used at all) drawn from the routes of the chosen itinerary and all candidates so far. `steps: []` disables it |
| `choice_set.fixed_costs` | Extra alternatives from re-querying with chosen generalized-cost parameters, run after the exclude rounds: one query (no exclusions) per entry, each setting any of `walk_reluctance`, `wait_reluctance`, `transit_reluctance` (one value for every transit mode, or `{mode: value}`; modes left out get 1) and `transfer_cost` (seconds). Left-out keys keep the plain query's values (`walk_reluctance` 2, the rest OTP's defaults). `[]` disables it |
| `choice_set.random_costs` | Extra alternatives from re-querying with random generalized-cost parameters, run after the `fixed_costs` rounds: `rounds` queries (no exclusions), each with `walk_reluctance` drawn uniform over the integers in its `[min, max]` (integer bounds; few distinct values, so OTP's transfer cache is reused), `wait_reluctance` and one `transit_reluctance` per transit mode drawn log-uniform from their `[min, max]` (min > 0), and `transfer_cost` (seconds, on every boarding after the first) drawn uniform from its `[min, max]`. Draws are seeded per trip by `seed` and `trip_id`. `rounds: 0` disables it |
| `choice_set.only_modes` | Extra alternatives from re-querying with only some transit modes, run after the `random_costs` rounds: one query (no exclusions, plain costs) per entry, a list of OTP transit modes (`BUS`, `SUBWAY`, `S_TRAIN`, `RAIL`, `TRAM`) it may use, e.g. `[["BUS"], ["SUBWAY", "S_TRAIN", "RAIL", "TRAM"]]`: bus only, then everything but bus. `[]` disables it |
| `choice_set.exclude_targeted` | Extra alternatives from peeling off the fastest routes, run last: per chain, that many queries, each adding route ids of the previous round's fastest itinerary (the first round's: of the plain query) to the routes the chain excludes, so slower routes that the faster ones dominate (e.g. a bus along a metro line) come out. `all` peels all its transit legs' routes; `first`/`last` only the first/last leg's, keeping the trunk line so other feeder/distributor buses to it come out. Deterministic; a chain stops early once a round finds nothing, and a round excluding the same routes as an earlier one reuses its itineraries without querying. `0` disables a chain |

## OTP client

`pt_network_simulation.otp.client` queries OTP's `planConnection` GraphQL API and can be used on its own:

- `load_all_candidates(origin, destination, depart_dt, ...)`: pages forward and backward from `depart_dt` until
  `max_itinerary_candidates` itineraries per direction or ± `search_window`, then drops duplicates and
  itineraries starting outside the window (OTP widens sparse pages). Returns one row per leg
  (`otp.parser.json_to_df`). Besides the access/egress modes it takes:
  - `modes_json`: transit modes, as `[{"mode": "BUS"}, ...]` (default `DEFAULT_TRANSIT_MODES`)
  - `route_short_name` (include) and `via_stopids`
  - `exclude_stopids`: no boarding/alighting at those stops; trips may pass through (the fork's `excludeStops`)
  - `exclude_tripids`: those trips are not used (the fork's `excludeTrips`)
  - `exclude_routeids`: route `gtfsId`s whose trips are not used (OTP's `filters` exclude, ANDed with the
    `route_short_name` include)
  - `walk_reluctance` (default 2), `car_reluctance`, `transit_reluctances` (`{mode: value}`, others 1),
    `wait_reluctance` and `transfer_cost` (seconds on every boarding after the first; left out, OTP's defaults
    apply: 1 and 0)
  - `print_query`
- `request_direct_leg(origin, destination, direct_mode, depart_dt, ...)`: street-only itineraries.
- `get_all_routes_for_mode(mode, url)` and `get_stops_by_bbox_query(lat, lon, bbox_buffer_m, otp_url)`.
- `trace.id`: a per-thread ID sent as `X-Correlation-ID`, so OTP's log lines name their trip. Needs
  `server.traceParameters` in OTP's `router-config.json`.

Ids are feed-scoped as in OTP's graph (e.g. a leg's `trip_gtfs_id`).

## Danish network assumptions

Making these configurable is planned; for now:

- `DEFAULT_TRANSIT_MODES` is `SUBWAY`, `BUS`, `TRAM`, `RAIL`, `S_TRAIN`: other modes (e.g. ferries) are not queried.
- `line_group` merges metro lines M1/M2 and M3/M4 and all S-train lines but F, on any network.
- Printed leg times (`print_choice_set`) are in `Europe/Copenhagen`.

## Example

`examples/synthetic_od_check.py` is a recall check on 12 Danish origin-destination pairs: per pair it compares
the choice set with an oracle that forces each route, or pair of routes, serving stops near the origin and
destination, and reports which plausible alternatives the choice set misses.

```bash
python examples/synthetic_od_check.py config.json <out_dir>
```

## License

MIT, see `LICENSE`.
