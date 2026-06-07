# Parallel GPX Trigger Handling — Method Description v1

## Prior Art Notice

This document is a public disclosure of the described method and architecture.

The purpose of publication is to document the existence and operation of the system as prior art.

No patent rights are claimed by this publication.

---

## Purpose

This document describes the design and operating principles of the parallel GPX trigger mechanism used in the T-Guide system.

**Version:** 1.0  
**System:** T-Guide route-based guided audio compiler  
**Companion document:** [T-Guide Audio Stitcher — Method Description](stitcher-method-v1.md)

---

## 1. Problem Statement

A GPS-triggered audio playback system places trigger zones — fixed polyline segments on a route — each associated with exactly one audio file in the runtime manifest. This one-to-one relationship is insufficient when a single geographic position must select among multiple audio variants based on a condition that is only known at the moment of the passenger's journey, such as the current ambient temperature, the calendar season, or the day of the year.

Compiling a separate trigger zone for each variant and activating the relevant one at runtime is not viable because:

(a) The device cannot know in advance which file will be selected, so it cannot pre-load or pre-position a specific trigger zone.  
(b) Multiple overlapping trigger zones at the same position, each referencing a different file, create ambiguous dispatch: the device would fire all of them simultaneously.

### Solution

The **parallel GPX trigger** mechanism resolves this by splitting compile-time and runtime responsibilities:

1. The compiler emits a single trigger zone in the primary manifest, referencing a **sentinel key** rather than a real audio filename.
2. All candidate files and their selection conditions are stored in a separate **parallel manifest**.
3. At runtime, when the device detects the sentinel key, it queries the appropriate condition source, evaluates the selection rule, and plays the matching file.

The primary manifest remains a simple sequential list. Conditional complexity is entirely contained within the parallel manifest and the runtime selection logic.

---

## 2. Assumptions

- All candidate files for a condition group share an identical trigger zone: the same route-aligned polyline segment. The device plays exactly one of them per traversal.
- The condition is evaluated at the moment the trigger threshold is crossed. The result must be available within a short latency window relative to the time the device traverses the trigger zone.
- Condition sources are queried at runtime: a weather API for temperature conditions, or the device calendar for date-based conditions. The set of candidate files is fixed at compile time; no variant is downloaded or generated at runtime.
- The sentinel key is a synthetic string that does not correspond to any real audio file on disk.

---

## 3. Terminology

| Term | Definition |
|---|---|
| Trigger zone | A GPS polyline segment on the route. When the device's GPS position enters the zone, playback is initiated. |
| Condition group | A set of audio files sharing a trigger position that are mutually exclusive at runtime. |
| Condition key | The dimension on which runtime selection is made: `temp_c`, `season`, or `day_of_year`. |
| Condition value | The specific value a candidate file matches (e.g. `"14"`, `"summer"`, `"135_319"`). |
| Sentinel key | The `sound_file` value written to the primary manifest for a condition group. Format: `{theme}_mech`. |
| Primary manifest | The guiding CSV consumed by the runtime device; one row per audio trigger. |
| Parallel manifest | Secondary CSV listing all conditional candidates with their selection metadata. |
| Branch A | Placement mode: the entire theme is a condition group — all files share one trigger position. |
| Branch B | Placement mode: a sequential theme where one sequence position contains a parallel mini-group of conditional variants, while other positions are unconditional. |

---

## 4. Compile-Time Processing

### 4.1 Audio file classification

The compiler identifies conditional files by their filename pattern. Each filename is matched against a priority-ordered rule set (described fully in the companion document, §5.1). A matched file produces a classification record with four fields:

- `theme`: normalised string identifier
- `seq`: integer sequence position within the theme
- `condition_key`: `"temp_c"` | `"season"` | `"day_of_year"` | `""` (empty = unconditional)
- `condition_value`: string representation of the matched condition value

Condition metadata is encoded entirely in the filename. No sidecar files are required.

### 4.2 Condition group formation

**Branch A — Full parallel theme**

A theme is declared as a full parallel group via a condition configuration CSV. The entry specifies the theme identifier, condition key, and condition data source. During placement, all files in the theme are assigned to a single trigger position: all share the same `trigger_start` and `trigger_end` values on the route. Each file receives its own record in the parallel manifest; the group is identified by `(theme, trigger_start)`.

**Branch B — Sequential theme with conditional mini-group**

The theme is not declared as a full parallel group. Files are placed in ascending sequence number order. When consecutive files share the same sequence number and all carry a non-empty condition key, they form a **parallel mini-group**:

- All files in the mini-group are assigned the same trigger position.
- Unconditional files at other sequence positions are placed as individual sequential blocks, each with its own trigger zone.
- The condition source for mini-group files is inferred from the condition key rather than from the condition configuration CSV.

This branch allows a theme to contain both a fixed unconditional component (an intro that always plays) and a conditional component (an outro selected at runtime by condition), placed at different positions on the route.

### 4.3 Sentinel row generation

The manifest writer iterates placed blocks in trigger-position order. On encountering the first block from a condition group — identified by a non-empty condition key — it:

1. Emits one row to the primary manifest with `sound_file = "{theme}_mech"`.
2. Records that the sentinel for this theme has been written.
3. Suppresses all subsequent blocks from the same theme.

The trigger GPX reference in that row points to the shared trigger zone (since all variants in the group share the same zone, any is equivalent).

Unconditional blocks (empty condition key) are emitted as normal rows referencing their actual audio filename.

### 4.4 Trigger GPX generation

One trigger zone GPX file is written per placed block, including every variant in a condition group. For conditional blocks, the GPX file's metadata description carries the condition key and condition value as structured annotations:

```
condition_key=temp_c|condition_value=14
```

For Branch A, all variant GPX files are geometrically identical (same polyline segment, same position on route). They differ only in filename and description metadata. The runtime device does not load per-variant GPX files directly; the primary manifest references only the sentinel, and the parallel manifest carries the candidate list.

### 4.5 Parallel manifest generation

The parallel manifest CSV is written after trigger zone generation. Every placed block with a non-empty condition key is included.

**Schema:**

| Column | Type | Description |
|---|---|---|
| group_id | integer | Shared by all files in the same condition group |
| section_name | string | Route segment identifier |
| mp3 | string | Audio filename (basename) |
| condition_key | string | `temp_c` \| `season` \| `day_of_year` |
| condition_value | string | Matched condition value |
| condition_source | string | Data source the device queries: `weather_api` \| `calendar` |
| trigger_s | float | Route distance (metres) to trigger zone start |
| trigger_end_s | float | Route distance (metres) to trigger zone end |
| audio_end_s | float | Route distance (metres) where audio finishes playing |
| duration_s | float | Audio clip length in seconds |
| trigger_gpx | string | Filename of trigger zone GPX |

`group_id` is assigned by enumerating unique `(theme, trigger_s)` pairs in row-sorted order. Rows are sorted by trigger position, then theme, then condition value.

---

## 5. Runtime Processing

### 5.1 Manifest loading

On session initialisation, the runtime device reads the primary manifest and partitions rows into two pools:

- **Direct rows:** `sound_file` does not end with `_mech`. Loaded as standard trigger zones; playback is initiated unconditionally on zone entry.
- **Sentinel rows:** `sound_file` ends with `_mech`. The device loads the referenced trigger zone GPX and registers it with the conditional trigger handler.

For each sentinel row, the device reads the parallel manifest and extracts all rows whose theme matches (derived by removing the `_mech` suffix from the sentinel key). This produces the candidate list for runtime selection.

Multiple sentinel polylines may be active simultaneously on the same route segment. Each is tracked independently.

### 5.2 Trigger detection

The conditional trigger handler maintains a traversal window for each sentinel polyline: a sliding buffer of recent GPS fixes. A trigger fires when a sufficient number of consecutive fixes project onto the trigger polyline within a threshold approach distance. This windowed detection reduces false positives caused by GPS position jitter.

When a trigger fires, the index of the matching sentinel polyline is recorded so that the correct theme and candidate list are dispatched.

### 5.3 Condition evaluation

When a sentinel trigger fires, the handler dispatches based on the `condition_key` associated with the sentinel's candidates:

---

**`temp_c` — Ambient temperature**

1. An HTTP request is sent to the configured weather API endpoint.
2. The response provides a temperature value in degrees Celsius (floating point).
3. The value is rounded to the nearest integer.
4. The candidate list is searched for the best match using the following rules:

   - Exact integer values are matched by numeric distance: `|queried_temp − candidate_int|`.
   - Below-range boundary values (encoded `sub_N`) match when the queried temperature is below the boundary integer `N`.
   - Above-range boundary values (encoded `N_up`) match when the queried temperature is above the integer `N`.
   - When candidates tie on distance, the lower value is preferred.

---

**`season` — Calendar season**

1. The device reads the current calendar date.
2. The month is mapped to a season string according to the hemisphere configured for the route segment. Supported values: `summer`, `winter`, `spring`, `autumn`, `rainy`, `dry`.
3. Exact string match is performed against each candidate's `condition_value`.

---

**`day_of_year` — Day-of-year range**

1. The current Julian day number (1–365) is derived from the device calendar.
2. Each candidate's `condition_value` is parsed as `"START_END"` (two integers separated by `_`).
3. The candidate whose range `[START, END)` contains the current day is selected.
4. If no range matches, no audio is played for this trigger.

---

### 5.4 Fallback behaviour

If the parallel manifest lookup fails or returns no candidates for a sentinel (manifest absent, network failure, format mismatch), the implementation should log a diagnostic and suppress playback for that trigger. Playback of an incorrect or arbitrary file in place of a missing result is not acceptable behaviour.

### 5.5 Playback

The resolved audio filename is used to load the corresponding file from the device's asset bundle. The file plays once. A reference to the active player is retained so that playback can be interrupted if a higher-priority event occurs (e.g. navigation instruction, route exit).

---

## 6. Condition Types — Reference

### 6.1 `temp_c` — Ambient temperature

- **Condition source:** Configurable weather API, returning current temperature in degrees Celsius.
- **Selection unit:** Integer degrees Celsius after rounding.
- **Filename conventions:**
  - `g_THEME_N.mp3` — matches temperature value N (single-chunk themes)
  - `g_THEME_sub_N.mp3` — matches temperatures below N
  - `g_THEME_N_up.mp3` — matches temperatures above N
  - `g_THEME_SEQ_N.mp3` — seq=SEQ, matches N (multi-chunk themes)
  - `g_THEME_SEQ_sub_N.mp3` — seq=SEQ, matches below N
  - `g_THEME_SEQ_N_up.mp3` — seq=SEQ, matches above N
- **Manifest value format:** plain integer string (`"14"`), `"sub_N"`, `"N_up"`.

### 6.2 `season` — Calendar season

- **Condition source:** Device calendar with hemisphere configuration.
- **Supported values:** `summer`, `winter`, `spring`, `autumn`, `rainy`, `dry`.
- **Filename convention:** `g_THEME_SEASON.mp3`
- **Manifest value format:** lowercase season keyword string.
- **Selection:** Exact string match.

### 6.3 `day_of_year` — Day-of-year range

- **Condition source:** Device calendar (Julian day number).
- **Filename convention:** `g_THEME_SEQ_START_END.mp3` — three trailing integers (sequence, range start, range end).
- **Manifest value format:** `"START_END"` e.g. `"135_319"`.
- **Selection:** The file whose declared range contains the current Julian day. Ranges are assumed non-overlapping.

---

## 7. Filename Pattern Disambiguation

Several filename patterns share structural similarity (trailing integers). The parser applies rules in strict priority order to avoid misclassification.

**Key ambiguities and their resolution:**

| Pattern A | Pattern B | Resolution |
|---|---|---|
| `g_THEME_SEQ_TEMPVAL.mp3` — 2 trailing integers | `g_THEME_SEQ_START_END.mp3` — 3 trailing integers | The day-of-year rule (3 integers) is checked before the temperature-chunk rule (2 integers). |
| `g_THEME_SEQ_sub_N.mp3` — chunked boundary sub | `g_THEME_sub_N.mp3` — plain boundary sub (SEQ absorbed into theme name) | Chunked boundary rules are checked before plain boundary rules. |
| `g_THEME_N.mp3` — single trailing integer, seq=N | `g_THEME_SEQ_TEMPVAL.mp3` — two trailing integers | A file with a single trailing integer matches the plain numbered rule; two trailing integers match the temperature-chunk rule. These are structurally unambiguous. |

**Complete priority order:**

1. Season keyword: `g_THEME_SEASON.mp3`
2. Chunked below-range: `g_THEME_SEQ_sub_N.mp3`
3. Chunked above-range: `g_THEME_SEQ_N_up.mp3`
4. Plain below-range: `g_THEME_sub_N.mp3`
5. Plain above-range: `g_THEME_N_up.mp3`
6. Day-of-year range: `g_THEME_SEQ_START_END.mp3` (3 trailing integers)
7. Temperature-chunk: `g_THEME_SEQ_TEMPVAL.mp3` (2 trailing integers)
8. Numbered sequential: `g_THEME_N.mp3` (1 trailing integer)
9. Simple (no condition): `g_THEME.mp3`

---

## 8. Examples

### Example 1 — Full parallel theme (Branch A)

**Scenario:** Theme `weather_commentary` is declared in the condition group configuration with `condition_key=temp_c`. The audio library contains one file per integer degree Celsius across an expected temperature range, plus below-range and above-range boundary files — 22 files total.

**Compile-time result:**
- All 22 files are assigned to a single trigger zone at position T1 on the route.
- The primary manifest contains one row:
  ```
  ..., t1_zone.gpx, weather_commentary_mech, ,
  ```
- The parallel manifest contains 22 rows, all with `group_id=0`, each with a distinct `condition_value`.

**Runtime at measured temperature 14 °C:**
1. Device enters T1 trigger zone → sentinel fires.
2. Weather API returns 14.2 °C → rounded to 14.
3. Parallel manifest searched: nearest match is the file with `condition_value="14"`.
4. That file plays.

---

### Example 2 — Sequential theme with conditional mini-group (Branch B)

**Scenario:** Theme `contextual_commentary` is not in the condition group configuration. The audio library contains:

```
g_contextual_commentary_1.mp3       seq=1, no condition
g_contextual_commentary_2_13.mp3    seq=2, condition_value="13"
g_contextual_commentary_2_14.mp3    seq=2, condition_value="14"
g_contextual_commentary_2_15.mp3    seq=2, condition_value="15"
g_contextual_commentary_2_16.mp3    seq=2, condition_value="16"
```

**Compile-time result:**
- Trigger zone T1: `g_contextual_commentary_1.mp3` — placed as a standard sequential block.
- Trigger zone T2 (later on route): four `seq=2` variants as a parallel mini-group, all sharing T2 geometry.
- Primary manifest rows:
  ```
  ..., t1_zone.gpx, g_contextual_commentary_1, ,
  ..., t2_zone.gpx, contextual_commentary_mech, ,
  ```
- Parallel manifest: 4 rows, all `group_id=1`, `condition_key=temp_c`.

**Runtime at 15 °C:**
1. T1 entered → `g_contextual_commentary_1.mp3` plays (unconditional).
2. T2 entered → sentinel fires → weather API returns 15.1 → rounds to 15 → `g_contextual_commentary_2_15.mp3` plays.

The passenger hears the unconditional intro at T1, then the temperature-matched outro at T2.

---

## 9. Limitations

- **Single condition dimension per trigger:** A trigger zone carries exactly one `condition_key`. Compound conditions (e.g. temperature AND season simultaneously) are not supported by the current manifest schema.
- **Nearest-match tie-breaking for `temp_c`:** When the queried temperature falls equidistant between two candidate values, the lower value is selected. This behaviour is fixed and not configurable per theme.
- **Gaps in temperature coverage:** If the measured temperature falls within the defined range but no candidate matches the rounded value and no boundary variant is defined, no audio plays. Coverage of the expected range is the operator's responsibility.
- **Sentinel suffix convention:** The runtime handler identifies sentinel rows by the `_mech` suffix on the `sound_file` value. An audio file with a name ending in `_mech` would be incorrectly intercepted as a sentinel. The suffix is reserved for system use.
- **Year-boundary day-of-year ranges:** A range spanning 31 December (where `START > END`) is not handled. The range produces no match. Ranges must be expressed within a single calendar year.
- **Condition API availability:** Temperature conditions require a successful network request at trigger time. If the network is unavailable when the trigger fires, no audio plays for that sentinel. There is no offline temperature fallback in the general case.
- **Condition evaluated once per traversal:** The condition source is queried when the traversal detection threshold is crossed. The condition is not re-evaluated if the vehicle decelerates or pauses within the trigger zone after the threshold has been passed.

---

## Revision History

| Version | Description |
|---------|-------------|
| 1.0 | Initial public disclosure |
