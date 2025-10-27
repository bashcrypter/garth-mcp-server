# Phase 1 – API Response Optimization Discovery

Goal: document how current tools talk to Garmin Connect, identify the highest
volume responses, and outline guardrails for later phases so the LLM context
remains manageable.

## 1. Endpoint Surface Area Review

### Stats-based tools (single `connectapi` request per page)

- All `daily_*` stats helpers (`DailySteps`, `DailyHydration`,
  `DailyIntensityMinutes`, `DailySleep`, `DailyStress`) use
  `garth.stats._base.Stats.list` with `_page_size = 28` days. When callers pass
  more than 28 days the helper automatically performs multiple API calls and
  concatenates results in memory, so a `days=90` request becomes four HTTP
  calls and roughly 90 returned objects.
- Weekly counterparts (`WeeklySteps`, `WeeklyStress`, `WeeklyIntensityMinutes`)
  use `_page_size = 52`. Asking for “two years” therefore triggers multiple
  long responses (~104 objects) and nothing clamps the `weeks` parameter in
  `src/garth_mcp_server/__init__.py`.
- `DailyHRV.list` overrides the base logic but still defaults to 28 days and
  paginates recursively when necessary.

### Data-based tools (per-day requests, multi-kilobyte payloads)

- Classes under `garth.data.*` (for example `DailyBodyBatteryStress`,
  `SleepData`, `HRVData`) inherit from `garth.data._base.Data`. `Data.list`
  dispatches one HTTP request per day via a `ThreadPoolExecutor(max_workers=10)`.
  `daily_body_battery(days=30)` therefore issues 30 authenticated GETs and
  returns 30 large objects that contain high-resolution arrays (stress values,
  body battery readings, HRV samples, sleep movement).
- `nightly_sleep` shows the precedent for trimming: it removes `sleep_movement`
  unless the user requests it. Similar trimming does not exist for other data
  tools yet.

### Direct `connectapi` wrappers

- `get_activities` only exposes `start_date` and `limit`, appending them to the
  path. The official/unofficial `garminconnect` client confirms the endpoint
  also supports `start` (offset), `endDate`, `activityType`, and `sortOrder`.
  Using those server-side filters before hitting Garmin would shrink payloads
  significantly.
- `snapshot` (`mobile-gateway/snapshot/detail/v2/{from}/{to}`) returns every
  tracked metric for each day in the range. We currently accept any span, so a
  user can accidentally fetch months of dense JSON.
- `get_connectapi_endpoint` is a raw escape hatch with no byte limits or
  filtering. Any large download immediately flows into the LLM prompt.

## 2. Data-Shaping & Guardrail Design

These defaults will be implemented in Phase 2 to keep responses bounded while
still allowing power users to opt in to verbatim data.

1. **Granularity negotiation for stats endpoints.** Add `max_points` (default
   30) and `granularity` arguments to every `daily_*` tool. When
   `granularity="auto"` and `days > 14`, switch to the weekly tool (or
   aggregate locally if no weekly data exists). Clamp `weeks` requests to 52
   unless explicitly overridden.
2. **Downsampling for data endpoints.** Add `max_points` and `include_samples`
   flags to high-volume tools such as `daily_body_battery`, `hrv_data`, and
   `daily_sleep`. Implement `_downsample_sequence` helpers that average windowed
   data or pick every Nth element so responses never exceed `max_points` unless
   callers opt in. Reuse the `sleep_movement` trimming approach for other bulky
   fields.
3. **Safer `get_activities` and `snapshot` wrappers.** Accept Garmin’s filters
   (`start`, `startDate`, `endDate`, `activityType`, `sortOrder`, `max_results`).
   Default to pagination windows of 20 activities (matching the web UI) and
   reject limits above 400 without confirmation. For `snapshot`, split ranges
   longer than seven days, aggregate results per chunk, and include metadata so
   consumers know the data is summarized.
4. **Generic endpoint guardrails.** Extend `get_connectapi_endpoint` with
   `params`, `method`, and `max_bytes`. Stream responses and stop with an
   actionable error if the payload would exceed ~256 KB unless the user raises
   the cap.
5. **Documentation + metadata.** Update tool docstrings and the README to spell
   out defaults (max days/weeks, downsampling policy, verbose flags) so clients
   can plan prompts accordingly.

## 3. Open Questions / Items to Validate

- Should the max-point defaults be environment-configurable (for example
  `GARTH_MCP_MAX_POINTS`)?
- Which summary statistics (mean, min/max, totals) provide the most value for
  aggregated responses? We will confirm once we inspect real payload samples.
- How should we describe chunk metadata so downstream tools can reconcile
  partial responses (for example `{ "granularity": "weekly" }` wrappers)?

These findings unblock Phase 2, where we will build helper utilities and wire
them into each MCP tool.
