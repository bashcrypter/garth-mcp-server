# Plan: Reduce API Payload Size And LLM Context Usage

## Goal

Add guardrails and summarized endpoints so Garmin MCP responses stay within the
LLM context window while preserving utility.

## Deliverables

1. Shared helper utilities for request clamping, data aggregation, and payload
   trimming.
2. Extended tool signatures (for example `get_activities`, `snapshot`,
   `daily_*`) that accept filters/limits and automatically downsample when
   callers request large ranges.
3. Documentation describing the new options and recommended usage.
4. Regression tests covering aggregation, clamping, and opt-in verbose modes.

## Work Breakdown

### 1. Discovery & Design

- [x] Inspect Garmin endpoints (activity list, snapshot, wellness metrics) to
  confirm available query parameters and aggregation endpoints. See
  `docs/phase-1-discovery.md` for notes.
- [x] Draft data-shaping rules (max days before switching to weekly, default
  `max_points`, fields to strip) and document the plan in
  `docs/phase-1-discovery.md`.

### 2. Helper Infrastructure

- [ ] Add private helpers in `src/garth_mcp_server/__init__.py`:
  - `_clamp_range(requested_points: int, *, max_points: int = 30) -> int`
  - `_maybe_use_weekly(days: int, *, threshold: int = 14)` returning
    `(granularity, fetch_fn)`
  - `_trim_payload(obj: Any, include_samples: bool) -> Any` to remove bulky
    fields recursively
  - `_downsample_sequence(data: Sequence[T], max_points: int,
    reducer: Callable[[Sequence[T]], T] | None)`
- [ ] Ensure helpers are unit-testable (pure functions) and typed.

### 3. Tool Updates

- [ ] `get_activities`: accept `end_date`, `activity_type`, `event_type`,
  `max_results`; default `max_results` to 100 and enforce a server-side upper
  bound. Refuse `limit > max_allowed` with guidance.
- [ ] `snapshot`: auto-split ranges longer than 7 days, fetch chunked data, and
  aggregate totals/averages per week before returning combined response (include
  metadata about the grouping).
- [ ] `daily_*` tools: accept `max_points` and `granularity`
  (`"auto" | "daily" | "weekly"`). On `auto`, call weekly endpoints or
  downsample so `len(data) <= max_points`.
- [ ] High-detail endpoints (`hrv_data`, `nightly_sleep`,
  `get_activity_details`): add `include_samples` flag and run `_trim_payload`
  unless explicitly true.
- [ ] `get_connectapi_endpoint`: allow optional `params`, `method`, and
  `max_bytes`; stream response and error if the payload exceeds `max_bytes`
  (default 256 KB) with guidance to narrow the request.

### 4. Documentation & Examples

- [ ] Update `README.md` tool descriptions to highlight new parameters and the
  automatic summarization behavior.
- [ ] Provide examples showing how to request weekly summaries vs. raw data and
  how to opt in to verbose outputs.

### 5. Testing & Validation

- [ ] Add pytest coverage for helper utilities and representative tools (mock
  `garth` responses) ensuring clamping, downsampling, and field trimming behave
  deterministically.
- [ ] If UI/interactive surfaces exist later, prepare Playwright end-to-end
  coverage per AGENTS.md.
- [ ] Run `uv run pytest`, `make lint`, and `make all` before opening a pull
  request.

## Open Questions

- Should we expose server-configurable defaults (for example environment
  variables for `MAX_POINTS`)? Decide during implementation.
- How should we merge metadata (chunk boundaries, aggregation notes) into
  responses to keep them LLM-friendly yet informative?
