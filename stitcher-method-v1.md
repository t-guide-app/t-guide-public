# T-Guide Audio Stitcher — Method Description v1

## Prior Art Notice

This document is a public disclosure of the described method and architecture.

The purpose of publication is to document the existence and operation of the system as prior art.

No patent rights are claimed by this publication.

---

## Purpose

This document describes the architecture and operating principles of the T-Guide offline audio placement system.

**Version:** 1.0  
**System:** T-Guide route-based guided audio compiler  
**Companion document:** [Parallel GPX Trigger Handling](parallel-gpx-trigger-v1.md)

---

## 1. Problem Statement

A vehicle-based guided tour presents pre-recorded audio narration that must begin playing at specific positions along a fixed GPS route. Each audio clip has a known duration. The vehicle moves at a speed that varies by road segment. The combination of clip duration, vehicle speed, and route geometry constrains where each clip may be triggered so that the following rules hold simultaneously:

- No two audio clips overlap in playback time.
- A minimum silence gap separates consecutive clips within a thematic group.
- A minimum silence gap separates clips from different thematic groups.
- Zones reserved for navigation instructions cannot be intruded upon by content audio.
- Certain audio clips are anchored to exact geographic positions regardless of capacity constraints.

The compilation is performed entirely offline before the tour session begins. The runtime device receives a set of trigger zone files and a manifest CSV; it performs no placement logic at runtime.

---

## 2. Assumptions

- The route is fixed and known before compilation. The compiler is deterministic: identical inputs produce identical outputs.
- Vehicle speed is approximated by a piecewise-constant model. Each road segment is assigned a single speed value. Continuous acceleration or deceleration between segments is not modelled.
- All distance calculations follow the route polyline. Crow-flight distances are not used anywhere in the system.
- Audio playback is triggered when a device's GPS position enters a designated trigger zone on the route. Playback begins at that moment; the system does not pre-buffer based on approach.
- Trigger zones are polyline segments of configurable length, within a minimum and maximum bound. They are densified to a configurable vertex interval for reliable GPS detection.
- The operator-specified theme sequence is authoritative. The placement engine does not reorder themes to improve fit.
- All audio files exist on disk before compilation begins. The compiler does not synthesize or modify audio content.
- A **job** is the unit of compilation: one route segment with its associated audio library, speed data, and configuration.

---

## 3. System Inputs

### 3.1 Route geometry
A GPX file containing an ordered sequence of WGS-84 track points representing the route. The compiler converts this into a cumulative distance index: a list of `(latitude, longitude, metres_from_start)` tuples, built by summing haversine distances between consecutive points. All downstream placement calculations operate in this one-dimensional distance space.

### 3.2 Guide audio library
A directory organised by theme. Each theme subdirectory contains audio files named according to the conventions in §5.1. Files are parsed into structured records encoding theme name, sequence number, duration, and optional condition metadata (see §5.1 for the full naming grammar). Files that match no known pattern are skipped.

### 3.3 Navigation audio files
Audio files paired with GPS waypoints indicating the exact firing position on the route. Two subtypes:

- **NOW**: plays at the anchor point.
- **PRE**: plays a specified distance before the NOW anchor, with a configurable placement tolerance.

Navigation files define corridors (reserved road spans) that no content audio may enter.

### 3.4 Theme sequence configuration
An operator-authored CSV listing the ordered themes for this route segment and which themes are active. Sequence order controls the left-to-right placement priority.

### 3.5 Speed data
A piecewise speed limit dataset derived from a map API and projected onto the route polyline. Localized speed override zones (supplied as GPX polylines) can be applied afterward. If no speed data is provided, a configured default speed is used throughout.

### 3.6 Condition group configuration
A CSV identifying themes where all audio variants share a single trigger zone and the runtime device selects one based on an external condition (ambient temperature, calendar season, or day of year). See the companion document for the full parallel trigger specification.

### 3.7 Hard-placement anchors
GPX waypoint files paired with audio files. Each anchor is projected onto the route and placed at that exact position, independently of silence constraints. Conflicts with navigation corridors are flagged but not suppressed.

### 3.8 Duration override table
An optional CSV providing manual audio durations, used in environments where metadata extraction from audio files is unavailable.

---

## 4. Processing Pipeline

### Stage 1 — Route indexing

The route GPX is parsed into an ordered point list. The cumulative distance index is constructed. All subsequent positions are expressed as a scalar distance value `s` (metres from route start).

### Stage 2 — Speed model construction

A piecewise-constant speed model is constructed over the cumulative index, initialised with a default speed. Map-derived speed segments are projected onto the polyline by nearest-point projection and applied in order. Localized overrides are applied afterward in order of specificity. The resulting model provides a function that returns the route position reached after a given number of seconds of travel from any starting position.

### Stage 3 — Audio library scan and classification

Each audio file in the library is matched against a priority-ordered sequence of filename patterns (§5.1). Matches produce structured classification records. Each record carries: theme identifier, sequence position, duration, and optional condition metadata (condition key and condition value).

Content audio not permitted for the current route segment is filtered out via a per-route allowlist. If no allowlist exists, it is generated automatically from the theme sequence configuration.

### Stage 4 — Condition group configuration loading

Condition group configuration is loaded from a configuration file. If no configuration is present, all themes are treated as sequential. The result maps theme identifiers to their condition key and condition data source.

### Stage 5 — Duration resolution

Audio durations are read from file metadata or from the duration override table. Files with unresolvable durations produce a warning; they are placed but contribute no audio-end distance to the model.

### Stage 6 — Theme sequence parsing

The theme sequence CSV is parsed into an ordered list of `(priority, theme_name)` pairs. Pre-flight cross-validation confirms that every active theme exists in both the allowlist and the audio library. A theme present in the allowlist but absent from the library is a hard error that aborts compilation.

### Stage 7 — Hard-placement anchor parsing

Each anchor's GPS waypoint is projected onto the route polyline. The result is a `(s_anchor, theme, audio_filename)` record. Anchor positions persist through all subsequent placement phases and are excluded from the sequential placement pool.

### Stage 8 — Placement engine

The placement engine operates in four sequential phases.

#### Phase 1: Navigation corridor reservation

Each NOW anchor is projected onto the route. The playback distance of the NOW audio is computed from the speed model. All PRE anchors for the same NOW anchor are projected; each generates a placement position derived by walking back the specified distance from the NOW position. A **corridor** spans from the earliest PRE position (extended by the configurable PRE tolerance) to the end of NOW playback plus a safety buffer. No content audio may start or play within any corridor. Corridors from logically related navigation files are merged into a single blocked span.

#### Phase 2: Hard-placement anchors

Each hard-placement anchor is assigned a trigger zone starting at its projected route position. If an anchor overlaps a navigation corridor, a conflict error is recorded; the anchor is still placed.

#### Phase 3: Available window computation

Blocked spans (navigation corridors and hard-placement audio extents) are merged and padded by a configurable safety margin on each side. The intervals between blocked spans form the **available windows** in which content audio may be placed.

#### Phase 4: Guide block placement

Themes are processed in sequence-CSV order. Before placement begins, a global dry-run estimates whether all themes fit within the available windows when using the maximum permitted inter-theme silence gap. If not all themes fit at maximum gap, the minimum gap is used throughout. This decision is made once and applied uniformly; it is never reconsidered per window.

Themes are assigned to windows left-to-right. Within a window, content is packed from the earliest available position (left-justified). When a window fills and remaining themes still exist, the next window is used.

For each theme, one of two placement branches is applied:

**Branch A — Full parallel theme:**
All audio files in the theme are assigned to a single trigger position. Every file shares the same trigger zone geometry. Each file is registered individually in the parallel manifest (§5.3). The primary manifest receives a single sentinel row referencing a synthetic key rather than a real audio filename. At runtime, the device queries the configured condition source and selects one file from the parallel manifest. See the companion document for the full runtime specification.

**Branch B — Sequential theme:**
Audio files are placed in ascending sequence number order. A configurable minimum silence gap is enforced between consecutive blocks. If multiple files share the same sequence position and all carry a condition key, they form a **parallel mini-group**: all variants are assigned the same trigger position, and the sentinel mechanism applies to that group only. Unconditional files at other sequence positions are placed as normal sequential blocks with their own trigger zones.

#### Phase 5: Silence redistribution

After initial placement, the trailing silence — the gap between the last audio block and the navigation fence — is computed. If inter-theme gaps exist and trailing silence is positive, the surplus is distributed evenly across all inter-theme gaps. Blocks are shifted in place; no re-run of the placement engine is required.

### Stage 9 — Output generation

#### Trigger zone GPX files
One GPX file is written per placed block. The file contains the polyline segment between the trigger start and trigger end positions, densified to a configurable vertex interval. For conditional blocks, the GPX metadata carries the condition key and condition value as structured annotations.

#### Parallel manifest
If any placed blocks carry a condition key, a parallel manifest CSV is written. Schema is described in §5.3.

#### Primary manifest (guiding CSV)
One row per audio trigger. Conditional block groups are collapsed to a single sentinel row. Structure is described in §5.2.

#### Additional outputs
A navigation instruction override CSV, a human-readable placement report, and diagnostic visualisation files are written alongside the primary manifests.

### Stage 10 — Approval and deployment

After operator review of the placement output, the job is approved. Approval packages all trigger GPX files, audio files, and manifests into the runtime asset bundle. An approved job is protected from re-compilation until explicitly unlocked.

---

## 5. Data Formats

### 5.1 Audio file naming conventions

Conditions and sequence positions are encoded directly in the audio filename. No sidecar metadata files are required. The compiler classifies each file by matching against the following patterns in strict priority order. All matching is case-insensitive.

| Priority | Pattern | Produces |
|---|---|---|
| 1 | `g_THEME_SEASON.mp3` | condition_key=season; season ∈ {summer, winter, spring, autumn, rainy, dry} |
| 2 | `g_THEME_SEQ_sub_N.mp3` | seq=SEQ; condition_key=temp_c; value=sub_N (below-range bound, chunked theme) |
| 2 | `g_THEME_SEQ_N_up.mp3` | seq=SEQ; condition_key=temp_c; value=N_up (above-range bound, chunked theme) |
| 3 | `g_THEME_sub_N.mp3` | seq=0; condition_key=temp_c; value=sub_N (below-range bound) |
| 4 | `g_THEME_N_up.mp3` | seq=0; condition_key=temp_c; value=N_up (above-range bound) |
| 5 | `g_THEME_SEQ_START_END.mp3` | seq=SEQ; condition_key=day_of_year; value=START_END (three trailing integers) |
| 6 | `g_THEME_SEQ_TEMPVAL.mp3` | seq=SEQ; condition_key=temp_c; value=TEMPVAL (two trailing integers) |
| 7 | `g_THEME_N.mp3` | seq=N; no condition (one trailing integer) |
| 8 | `g_THEME.mp3` | seq=1; no condition |

Theme names are normalised to lowercase with hyphens converted to underscores.

The priority ordering resolves structural ambiguity: patterns with three trailing integers (day-of-year range) are matched before patterns with two (temperature-chunked), which are matched before patterns with one (numbered sequential). Chunked boundary variants (containing a keyword `sub` or `up`) are matched before the plain boundary variants to prevent the sequence number from being absorbed into the theme name.

### 5.2 Primary manifest (guiding CSV) columns

| Column | Type | Description |
|---|---|---|
| serial | integer | Monotonically increasing row identifier |
| section_name | string | Route segment identifier |
| trigger_gpx | string | Filename of the trigger zone GPX |
| sound_file | string | Audio basename (no extension) or sentinel key |
| date_min | string | First valid calendar date (ISO 8601), or empty |
| date_max | string | Last valid calendar date (ISO 8601), or empty |

For conditional (parallel) block groups, `sound_file` contains a sentinel key of the form `{theme}_mech`. The device recognises this suffix and delegates file selection to the parallel manifest at runtime.

### 5.3 Parallel manifest columns

| Column | Type | Description |
|---|---|---|
| group_id | integer | Shared by all candidates in one condition group |
| section_name | string | Route segment identifier |
| mp3 | string | Audio filename (basename) |
| condition_key | string | `temp_c` \| `season` \| `day_of_year` |
| condition_value | string | Matched condition value |
| condition_source | string | Data source the device must query: `weather_api` \| `calendar` |
| trigger_s | float | Route distance (metres) to trigger zone start |
| trigger_end_s | float | Route distance (metres) to trigger zone end |
| audio_end_s | float | Route distance (metres) where audio finishes |
| duration_s | float | Audio clip length in seconds |
| trigger_gpx | string | Path to trigger zone GPX |

`group_id` is assigned by enumerating unique `(theme, trigger_s)` pairs in sort order.

---

## 6. Validation and Diagnostics

The compiler classifies placement outcomes into errors and warnings:

| Category | Severity | Condition |
|---|---|---|
| Insufficient inter-block silence | Error | Gap between consecutive blocks is below the configured minimum |
| Insufficient inter-theme silence | Error | Gap between themes is below the configured minimum |
| Audio overlap | Error | Two audio blocks overlap in playback time |
| Navigation corridor violation | Error | Content block placed within a nav corridor |
| Hard-placement conflict | Error | Anchored block overlaps a nav corridor |
| Content overflow | Error | One or more themes could not be placed in any available window |
| Unresolved duration | Warning | Audio duration could not be determined; block placed with zero length |

Jobs with any errors do not produce approved output.

---

## 7. Examples

### Example A — Multi-theme sequential job

**Setup:** A route of several kilometres, default piecewise speed, seven thematic groups each containing three to four sequential audio blocks (total approximately 22 minutes of content), one navigation NOW+PRE pair positioned near the end of the route.

**Result:** All guide blocks are placed in the single available window preceding the navigation corridor. Navigation blocks are placed at the anchor position. Zero errors. A trailing silence buffer remains at the end of the window.

---

### Example B — Temperature-conditional theme (full parallel, Branch A)

**Setup:** Theme `weather_reads` is listed in the condition group configuration with `condition_key=temp_c`, `condition_source=weather_api`. The audio library contains one file per integer degree Celsius across the expected temperature range, plus below-range and above-range boundary files — approximately 20 files total.

**Placement result:** All 20 files are placed at a single trigger position. All share the same trigger zone geometry.

**Primary manifest row:**
```
..., weather_reads_trigger.gpx, weather_reads_mech, ,
```

**Parallel manifest rows (excerpt):**
```
group_id, section_name, mp3,                   condition_key, condition_value, condition_source
0,        route_alpha,  g_weather_reads_13.mp3, temp_c,        13,             weather_api
0,        route_alpha,  g_weather_reads_14.mp3, temp_c,        14,             weather_api
...
```

**Runtime behaviour at 14 °C:** The weather API returns 14.2 → rounded to 14 → nearest match is `g_weather_reads_14.mp3` → played.

---

### Example C — Chunked theme with conditional mini-group (Branch B)

**Setup:** Theme `seasonal_commentary` is not in the condition group configuration (sequential placement applies). The audio library contains:

```
g_seasonal_commentary_1.mp3          seq=1, no condition  — always plays
g_seasonal_commentary_2_13.mp3       seq=2, condition_value="13"
g_seasonal_commentary_2_14.mp3       seq=2, condition_value="14"
g_seasonal_commentary_2_15.mp3       seq=2, condition_value="15"
g_seasonal_commentary_2_16.mp3       seq=2, condition_value="16"
```

**Placement result:**
- Trigger zone T1: `g_seasonal_commentary_1.mp3` — unconditional block, plays at every temperature
- Trigger zone T2 (later on route): four `seq=2` variants as a parallel mini-group, all sharing T2 geometry

**Primary manifest rows:**
```
..., t1_trigger.gpx, g_seasonal_commentary_1, ,
..., t2_trigger.gpx, seasonal_commentary_mech, ,
```

**Runtime behaviour at 15 °C:** T1 fires → `g_seasonal_commentary_1.mp3` plays. T2 fires → weather API returns 15.1 → rounded to 15 → `g_seasonal_commentary_2_15.mp3` plays.

---

## 8. Limitations

- **Offline determinism:** The compiler has no knowledge of real-time traffic, vehicle position, or weather. All placement decisions are made at compile time using the configured speed model.
- **Piecewise speed model:** Instantaneous speed transitions are assumed at segment boundaries. Acceleration and deceleration profiles are not modelled.
- **No backtracking:** Themes are assigned to windows in sequence-CSV order with a single left-to-right pass. The engine does not attempt permutations or explore alternative window assignments.
- **Single connected route:** Branching routes (operator chooses between two roads) are not supported. Each job compiles a single linear polyline.
- **Trigger zone length assumptions:** The configured trigger zone bounds assume a minimum travel speed sufficient to traverse the zone in a number of seconds adequate for reliable GPS detection. At very low speeds or on foot, trigger zone length may need to be increased.
- **Single condition dimension per trigger:** A trigger zone carries exactly one `condition_key`. Compound conditions (e.g. temperature AND season simultaneously) are not supported.
- **Year-boundary day-of-year ranges:** A `day_of_year` condition range where the start day is numerically greater than the end day (spanning 31 December) is not handled; the range produces no match.
- **Condition evaluated once per traversal:** The condition source is queried at the moment the traversal threshold is crossed. Slow traversal (e.g. stationary traffic) does not re-evaluate the condition as conditions change.

---

## Revision History

| Version | Description |
|---------|-------------|
| 1.0 | Initial public disclosure |
