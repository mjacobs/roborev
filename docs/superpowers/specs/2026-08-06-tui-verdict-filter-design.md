# TUI Verdict Filter Design

## Goal

Let users narrow the TUI queue to passing or failing review verdicts while
retaining every existing filter. In particular, combining the new filter with
`h` must show only open reviews with a `FAIL` verdict.

## User Experience

In the queue view, `H` cycles through three states:

```text
all -> FAIL -> PASS -> all
```

The active verdict appears in the title alongside the existing repo and branch
filter chips:

```text
roborev [f: repo] [b: main] [H: FAIL] [hiding closed]
```

The help view describes the related controls without confusing a failed agent
run with a failing review verdict:

```text
h     Toggle hide closed/failed/canceled
H     Cycle verdict all/fail/pass
esc   Clear filters (one at a time)
```

Jobs without a verdict do not match either `PASS` or `FAIL`. This excludes
queued, running, canceled, and agent-error jobs from a verdict-filtered view.

## Filter Composition

The verdict is an ordinary queue filter. It composes with repo, branch,
hide-closed, and classify-row filtering. Applying or changing it resets queue
pagination, invalidates stale fetches, and fetches the first matching page.

The verdict participates in the existing last-applied filter stack used by
`Esc`. If verdict was the most recently applied stacked filter, `Esc` clears it
first. The existing `h` state remains separate: after stacked filters are
cleared, another `Esc` disables hide-closed, preserving current behavior.

Cycling an already active verdict moves it to the top of the filter stack. The
third `H` press returns to the unfiltered verdict state and removes the verdict
entry from the stack.

## Data Flow

Verdict filtering happens server-side so pagination and queue counts use the
same verdict scope as the rows shown on screen. As in the existing TUI, queue
header counts ignore the hide-closed toggle: within the selected verdict, they
continue to show completed, closed, and open totals while `h` narrows the rows.

The TUI sends a verdict query value with initial and paginated job requests.
The daemon validates the supported values and passes the selection into the
storage job-list query. Storage filters on the persisted review verdict rather
than reparsing review prose. Existing repo, branch, closed-state, job-type, and
classify filters remain conjunctive, so `closed=false` plus `verdict=fail`
returns only open failing review results.

The new query is optional. Callers that omit it retain current behavior.

## Error Handling

Unsupported verdict query values receive the same invalid-request treatment as
other invalid daemon job-list filters. A valid filter with no matches returns
an empty page and zero scoped counts, not an error.

The TUI keeps the existing stale-response generation guard. Rapidly cycling
`H` cannot replace the newest filter state with an older response.

## Testing

Focused tests will prove:

- storage returns only the requested persisted verdict and combines verdict
  with open/closed selection;
- the daemon accepts valid verdict filters and rejects invalid values;
- TUI requests include the verdict for initial fetches and pagination;
- `H` cycles all, fail, pass, and all while updating the filter stack;
- `h` and `H` compose to request open failing results;
- `Esc` clears verdict according to last-applied filter order;
- the active title chip and help text expose the new control.

The implementation will also run the relevant package tests followed by the
repository's standard Go, lint, and Markdown quality gates.
