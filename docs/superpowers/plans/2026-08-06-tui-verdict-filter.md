# TUI Verdict Filter Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> superpowers:subagent-driven-development (recommended) or
> superpowers:executing-plans to implement this plan task-by-task. Steps use
> checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a composable TUI queue filter that cycles between all, failing,
and passing review verdicts.

**Architecture:** Add an optional `verdict=fail|pass` query to the daemon jobs
endpoint and translate it to a persisted `reviews.verdict_bool` storage filter.
The TUI owns a three-state filter, sends it on initial and paginated requests,
and integrates it with the existing filter stack, title chips, help, local
visibility checks, and control socket.

**Tech Stack:** Go, SQLite, Huma, Bubble Tea, testify, generated OpenAPI client,
Zensical Markdown.

## Global Constraints

- `H` cycles `all -> FAIL -> PASS -> all` in the queue view.
- `h` and `H` remain independent; together they show open failing results.
- Jobs without a review verdict match neither `PASS` nor `FAIL`.
- Verdict filtering is server-side and composes conjunctively with every
  existing jobs filter.
- Queue header stats are verdict-scoped but retain the existing behavior of
  ignoring hide-closed so completed, closed, and open totals remain visible.
- The optional query must not change callers that omit it.
- Do not add a schema migration or compatibility fallback.
- Do not invoke `roborev review`, touch the live daemon/data directory, install
  a development binary, change branches, amend commits, or merge a pull
  request.

---

## File Map

- `internal/storage/jobs.go`: storage option and SQL predicate for persisted
  verdicts.
- `internal/storage/db_filter_test.go`: storage behavior and verdict/open
  composition coverage.
- `internal/daemon/types.go`: public jobs query definition.
- `internal/daemon/server.go`: query-to-storage option translation for rows and
  stats.
- `internal/daemon/server_jobs_test.go`: endpoint validation, filtering,
  composition, and stats coverage.
- `pkg/client/openapi.yaml` and `pkg/client/generated/*.go`: generated public
  API schema and Go client.
- `cmd/roborev/tui/tui.go`: active verdict state.
- `cmd/roborev/tui/helpers.go`: filter and verdict constants.
- `cmd/roborev/tui/filter.go`: local visibility and Escape-stack behavior.
- `cmd/roborev/tui/fetch.go`: initial and paginated request parameters.
- `cmd/roborev/tui/handlers.go` and `handlers_queue.go`: `H` dispatch and state
  cycle.
- `cmd/roborev/tui/render_queue.go` and `render_log.go`: title, empty state,
  compact footer, and full help copy.
- `cmd/roborev/tui/control_types.go` and `control_handlers.go`: expose, set, and
  clear the filter through the TUI control socket.
- `cmd/roborev/tui/filter_stack_test.go`, `fetch_test.go`, `status_line_test.go`,
  `helpers_test.go`, and `control_test.go`: focused TUI behavior coverage.
- `docs/integrations/tui.md` and `docs/changelog.md`: user documentation and
  unreleased note.

---

### Task 1: Persisted Verdict Storage Filter

**Files:**

- Modify: `internal/storage/db_filter_test.go`
- Modify: `internal/storage/jobs.go`

**Interfaces:**

- Produces: `storage.WithVerdict(pass bool) ListJobsOption`
- Consumes: existing `ListJobs` and `CountJobStats` option plumbing

- [ ] **Step 1: Write the failing storage test**

Add a focused test that creates one passing result, one open failing result,
one closed failing result, and one queued job. Set `reviews.verdict_bool`
explicitly after completing the terminal jobs so this test isolates query logic
from verdict parsing.

```go
func TestListJobsVerdictFilter(t *testing.T) {
	db := openTestDB(t)
	defer db.Close()

	repo, err := db.GetOrCreateRepo("/tmp/verdict-filter")
	require.NoError(t, err)

	var passID, openFailID, closedFailID int64
	for i, pass := range []bool{true, false, false} {
		sha := fmt.Sprintf("verdict-filter-%d", i)
		commit := createCommit(t, db, repo.ID, sha)
		job := enqueueJob(t, db, repo.ID, commit.ID, sha)
		claimed, claimErr := db.ClaimJob("verdict-filter-worker")
		require.NoError(t, claimErr)
		require.Equal(t, job.ID, claimed.ID)
		require.NoError(t, db.CompleteJob(job.ID, "test", "prompt", "result"))
		verdict := 0
		if pass {
			verdict = 1
		}
		_, err = db.Exec(`UPDATE reviews SET verdict_bool = ? WHERE job_id = ?`, verdict, job.ID)
		require.NoError(t, err)
		switch i {
		case 0:
			passID = job.ID
		case 1:
			openFailID = job.ID
		case 2:
			closedFailID = job.ID
			require.NoError(t, db.MarkReviewClosedByJobID(job.ID, true))
		}
	}

	queuedCommit := createCommit(t, db, repo.ID, "verdict-filter-queued")
	enqueueJob(t, db, repo.ID, queuedCommit.ID, "verdict-filter-queued")

	passing, err := db.ListJobs("", "", 50, 0, WithVerdict(true))
	require.NoError(t, err)
	require.Len(t, passing, 1)
	assert.Equal(t, passID, passing[0].ID)

	failing, err := db.ListJobs("", "", 50, 0, WithVerdict(false))
	require.NoError(t, err)
	require.Len(t, failing, 2)
	assert.ElementsMatch(t, []int64{openFailID, closedFailID}, []int64{failing[0].ID, failing[1].ID})

	openFailing, err := db.ListJobs("", "", 50, 0, WithVerdict(false), WithClosed(false))
	require.NoError(t, err)
	require.Len(t, openFailing, 1)
	assert.Equal(t, openFailID, openFailing[0].ID)

	stats, err := db.CountJobStats("", WithVerdict(false))
	require.NoError(t, err)
	assert.Equal(t, JobStats{Done: 2, Closed: 1, Open: 1}, stats)
}
```

- [ ] **Step 2: Run the test and verify RED**

Run:

```bash
go test ./internal/storage -run '^TestListJobsVerdictFilter$'
```

Expected: compilation fails because `WithVerdict` does not exist.

- [ ] **Step 3: Implement the minimal storage option and predicate**

Add the field to `listJobsOptions` and define the option:

```go
verdict *bool

func WithVerdict(pass bool) ListJobsOption {
	return func(o *listJobsOptions) { o.verdict = &pass }
}
```

Add the persisted predicate in `buildJobFilterClause`:

```go
if o.verdict != nil {
	conditions = append(conditions, "rv.verdict_bool = ?")
	if *o.verdict {
		args = append(args, 1)
	} else {
		args = append(args, 0)
	}
}
```

- [ ] **Step 4: Run storage tests and verify GREEN**

Run:

```bash
go test ./internal/storage -run 'TestListJobsVerdictFilter|TestListJobsWithBranchAndClosedFilters'
```

Expected: PASS.

- [ ] **Step 5: Commit the storage slice**

Use the mandatory commit skill, review the diff/status, and commit the two
files with a rationale-first message such as:

```bash
git add internal/storage/jobs.go internal/storage/db_filter_test.go
git commit -m "feat: filter jobs by review verdict"
```

---

### Task 2: Daemon Jobs API and Generated Client

**Files:**

- Modify: `internal/daemon/server_jobs_test.go`
- Modify: `internal/daemon/types.go`
- Modify: `internal/daemon/server.go`
- Regenerate: `pkg/client/openapi.yaml`
- Regenerate: `pkg/client/generated/*.go`

**Interfaces:**

- Consumes: `storage.WithVerdict(pass bool)` from Task 1
- Produces: optional `GET /api/jobs?verdict=pass|fail`
- Produces: generated `ListJobsQuery.Verdict *ListJobsQueryVerdict`

- [ ] **Step 1: Write failing daemon endpoint tests**

Add a test that seeds PASS, open FAIL, and closed FAIL reviews, then verifies
the valid filters, open/fail composition, verdict-scoped stats, and invalid enum
handling.

```go
func TestHandleListJobsVerdictFilter(t *testing.T) {
	server, db, _ := newTestServer(t)
	repo, err := db.GetOrCreateRepo("/test/verdict-filter")
	require.NoError(t, err)

	var openFailID int64
	for i, verdict := range []int{1, 0, 0} {
		job, enqueueErr := db.EnqueueJob(storage.EnqueueOpts{
			RepoID: repo.ID, GitRef: fmt.Sprintf("verdict-%d", i), Agent: "test",
		})
		require.NoError(t, enqueueErr)
		claimed, claimErr := db.ClaimJob("verdict-worker")
		require.NoError(t, claimErr)
		require.Equal(t, job.ID, claimed.ID)
		require.NoError(t, db.CompleteJob(job.ID, "test", "prompt", "result"))
		_, err = db.Exec(`UPDATE reviews SET verdict_bool = ? WHERE job_id = ?`, verdict, job.ID)
		require.NoError(t, err)
		if i == 1 {
			openFailID = job.ID
		}
		if i == 2 {
			require.NoError(t, db.MarkReviewClosedByJobID(job.ID, true))
		}
	}

	failing := fetchJobs(t, server, "verdict=fail")
	assert.Len(t, failing.Jobs, 2)
	assert.Equal(t, storage.JobStats{Done: 2, Closed: 1, Open: 1}, failing.Stats)

	openFailing := fetchJobs(t, server, "verdict=fail&closed=false")
	require.Len(t, openFailing.Jobs, 1)
	assert.Equal(t, openFailID, openFailing.Jobs[0].ID)
	assert.Equal(t, storage.JobStats{Done: 2, Closed: 1, Open: 1}, openFailing.Stats)

	passing := fetchJobs(t, server, "verdict=pass")
	assert.Len(t, passing.Jobs, 1)

	req := httptest.NewRequest(http.MethodGet, "/api/jobs?verdict=maybe", nil)
	w := httptest.NewRecorder()
	server.httpServer.Handler.ServeHTTP(w, req)
	assert.Equal(t, http.StatusUnprocessableEntity, w.Code)
}
```

- [ ] **Step 2: Run the daemon test and verify RED**

Run:

```bash
go test ./internal/daemon -run '^TestHandleListJobsVerdictFilter$'
```

Expected: valid requests ignore `verdict`, so the result counts are wrong and
the invalid value is accepted.

- [ ] **Step 3: Add the typed query and storage translation**

Add the optional Huma query field:

```go
Verdict string `query:"verdict" doc:"Filter by review verdict" enum:"pass,fail,"`
```

Apply it to both listing and stats options:

```go
switch input.Verdict {
case "pass":
	listOpts = append(listOpts, storage.WithVerdict(true))
case "fail":
	listOpts = append(listOpts, storage.WithVerdict(false))
}
```

Repeat the same switch for `statsOpts`, preserving the existing rule that
stats ignore only the closed filter.

- [ ] **Step 4: Regenerate the public OpenAPI client**

Run:

```bash
make api-generate
```

Expected: the checked-in OpenAPI schema and generated query/enum/client files
gain the optional verdict filter. Do not hand-edit generated files.

- [ ] **Step 5: Run API and generated-client tests**

Run:

```bash
go test ./internal/daemon -run 'TestHandleListJobsVerdictFilter|TestHumaOpenAPISpec'
```

If the package patterns select no generated-client tests, also run:

```bash
go test ./pkg/client/generated
```

Expected: PASS.

- [ ] **Step 6: Commit the API slice**

Use the mandatory commit skill, stage the daemon and generated API files, and
commit with a rationale-first message such as:

```bash
git add internal/daemon/types.go internal/daemon/server.go \
  internal/daemon/server_jobs_test.go pkg/client/openapi.yaml \
  pkg/client/generated
git commit -m "feat: expose verdict job filtering"
```

---

### Task 3: TUI Verdict State, Requests, Rendering, and Control

**Files:**

- Modify: `cmd/roborev/tui/tui.go`
- Modify: `cmd/roborev/tui/helpers.go`
- Modify: `cmd/roborev/tui/filter.go`
- Modify: `cmd/roborev/tui/fetch.go`
- Modify: `cmd/roborev/tui/handlers.go`
- Modify: `cmd/roborev/tui/handlers_queue.go`
- Modify: `cmd/roborev/tui/render_queue.go`
- Modify: `cmd/roborev/tui/render_log.go`
- Modify: `cmd/roborev/tui/control_types.go`
- Modify: `cmd/roborev/tui/control_handlers.go`
- Test: `cmd/roborev/tui/filter_stack_test.go`
- Test: `cmd/roborev/tui/fetch_test.go`
- Test: `cmd/roborev/tui/status_line_test.go`
- Test: `cmd/roborev/tui/helpers_test.go`
- Test: `cmd/roborev/tui/control_test.go`

**Interfaces:**

- Consumes: generated `ListJobsQueryVerdict` and daemon `verdict` query
- Produces: `activeVerdictFilter` with values `""`, `"fail"`, or `"pass"`
- Produces: control-socket `verdict_filter` state and `set-filter` /
  `clear-filter` verdict parameters

- [ ] **Step 1: Write failing key-cycle and Escape-stack tests**

Add tests that press uppercase `H`, assert the cycle, and prove last-applied
stack behavior with repo and hide-closed filters.

```go
func TestTUIVerdictFilterCycles(t *testing.T) {
	m := newModel(localhostEndpoint, withExternalIODisabled())
	m.currentView = viewQueue

	failing, failCmd := pressKey(m, 'H')
	assert.Equal(t, verdictFilterFail, failing.activeVerdictFilter)
	assert.Equal(t, []string{filterTypeVerdict}, failing.filterStack)
	assert.NotNil(t, failCmd)

	passing, passCmd := pressKey(failing, 'H')
	assert.Equal(t, verdictFilterPass, passing.activeVerdictFilter)
	assert.Equal(t, []string{filterTypeVerdict}, passing.filterStack)
	assert.NotNil(t, passCmd)

	all, allCmd := pressKey(passing, 'H')
	assert.Empty(t, all.activeVerdictFilter)
	assert.Empty(t, all.filterStack)
	assert.NotNil(t, allCmd)
}

func TestTUIVerdictFilterEscapesBeforeHideClosed(t *testing.T) {
	m := newModel(localhostEndpoint, withExternalIODisabled())
	m.currentView = viewQueue
	m.hideClosed = true
	m.activeVerdictFilter = verdictFilterFail
	m.filterStack = []string{filterTypeVerdict}

	withoutVerdict, _ := pressSpecial(m, tea.KeyEscape)
	assert.Empty(t, withoutVerdict.activeVerdictFilter)
	assert.True(t, withoutVerdict.hideClosed)

	withoutHideClosed, _ := pressSpecial(withoutVerdict, tea.KeyEscape)
	assert.False(t, withoutHideClosed.hideClosed)
}
```

- [ ] **Step 2: Write failing request-composition tests**

Add an HTTP recorder test in `fetch_test.go` that runs both `fetchJobs` and
`fetchMoreJobs` with `activeVerdictFilter=fail` and `hideClosed=true`, then
asserts both requests carry `verdict=fail` and `closed=false`. Also add a direct
`listJobsQuery` assertion for the generated typed verdict value.

```go
assert.Equal(t, "fail", jobsQuery.Get("verdict"))
assert.Equal(t, "false", jobsQuery.Get("closed"))
assert.Equal(t, "fail", moreQuery.Get("verdict"))
assert.Equal(t, "false", moreQuery.Get("closed"))
```

- [ ] **Step 3: Write failing render, visibility, and control tests**

Add focused assertions that:

```go
assert.Contains(t, stripTestANSI(m.renderQueueTitle()), "[H: FAIL]")
assert.Contains(t, stripTestANSI(m.renderHelpView()), "H")
assert.Contains(t, stripTestANSI(m.renderHelpView()), "Cycle verdict all/fail/pass")
```

Extend control tests so `get-state` and `get-filter` report `verdict_filter`,
`set-filter` accepts `"fail"`/`"pass"` and rejects other values, and
`clear-filter` removes it. Add a local visibility test proving a job whose
verdict no longer matches is hidden before the next fetch completes.

- [ ] **Step 4: Run the focused TUI tests and verify RED**

Run:

```bash
go test ./cmd/roborev/tui -run 'VerdictFilter|HelpRows|RenderQueueTitle|Control.*Filter'
```

Expected: compilation failures for the new state/constants followed by missing
request, render, and control behavior.

- [ ] **Step 5: Implement the three-state filter and stack behavior**

Add constants and model state:

```go
const (
	filterTypeRepo    = "repo"
	filterTypeBranch  = "branch"
	filterTypeVerdict = "verdict"

	verdictFilterFail = "fail"
	verdictFilterPass = "pass"
)
```

```go
activeVerdictFilter string // Empty = all, otherwise fail or pass
```

Dispatch uppercase `H` only from the queue and implement the cycle:

```go
func (m model) handleVerdictFilterKey() (tea.Model, tea.Cmd) {
	if m.currentView != viewQueue {
		return m, nil
	}
	switch m.activeVerdictFilter {
	case "":
		m.activeVerdictFilter = verdictFilterFail
		m.pushFilter(filterTypeVerdict)
	case verdictFilterFail:
		m.activeVerdictFilter = verdictFilterPass
		m.pushFilter(filterTypeVerdict)
	default:
		m.activeVerdictFilter = ""
		m.removeFilterFromStack(filterTypeVerdict)
	}
	m.resetQueueForFilterChange()
	return m, m.fetchJobs()
}
```

Teach `popFilter` and `handleEscKey` to clear and refetch for
`filterTypeVerdict`. Add the same local predicate used during optimistic state
changes:

```go
if m.activeVerdictFilter != "" {
	if job.Verdict == nil {
		return false
	}
	want := "F"
	if m.activeVerdictFilter == verdictFilterPass {
		want = "P"
	}
	if *job.Verdict != want {
		return false
	}
}
```

- [ ] **Step 6: Send verdict on every paginated queue request**

In `fetchJobs` and `fetchMoreJobs`:

```go
if m.activeVerdictFilter != "" {
	params.Set("verdict", m.activeVerdictFilter)
}
```

In `listJobsQuery`, convert the string to the generated enum and set
`query.Verdict`. Do not add the filter to task or panel-member requests.

- [ ] **Step 7: Render and expose the active state**

Add `[H: FAIL]` or `[H: PASS]` in `titleFilters` at its filter-stack position,
include verdict in all “filters active” and empty-state checks, add `H` to the
compact queue footer and full help overlay, and keep the existing `h` copy.

Extend the control state/filter responses and mutations:

```go
VerdictFilter string `json:"verdict_filter"`
```

Add `Verdict *string` to the `set-filter` params and validate before mutating
any filter:

```go
if params.Verdict != nil && *params.Verdict != "" &&
	*params.Verdict != verdictFilterFail &&
	*params.Verdict != verdictFilterPass {
	return m, controlResponse{Error: "verdict filter must be fail or pass"}, nil
}
```

Apply or remove it with the same stack functions as the keyboard path:

```go
if params.Verdict != nil {
	m.activeVerdictFilter = *params.Verdict
	if *params.Verdict == "" {
		m.removeFilterFromStack(filterTypeVerdict)
	} else {
		m.pushFilter(filterTypeVerdict)
	}
}
```

Add `Verdict bool` to `clear-filter`; when true, clear
`activeVerdictFilter` and remove `filterTypeVerdict` before the shared reset and
fetch.

- [ ] **Step 8: Run focused and package TUI tests and verify GREEN**

Run:

```bash
go test ./cmd/roborev/tui -run 'VerdictFilter|HelpRows|RenderQueueTitle|Control.*Filter'
go test ./cmd/roborev/tui
```

Expected: PASS with no warnings.

- [ ] **Step 9: Commit the TUI slice**

Use the mandatory commit skill, stage every TUI file related to the feature,
and commit with a rationale-first message such as:

```bash
git add cmd/roborev/tui
git commit -m "feat: filter TUI reviews by verdict"
```

---

### Task 4: User Documentation and Full Verification

**Files:**

- Modify: `docs/integrations/tui.md`
- Modify: `docs/changelog.md`

**Interfaces:**

- Documents: `H`, title state, `h` composition, Escape behavior, API query,
  and control-socket fields

- [ ] **Step 1: Update the TUI guide and changelog**

Add `H` to the key table and explain the cycle with a concrete example:

```text
Press `H` once for failing verdicts, twice for passing verdicts, and a third
time to show every verdict. Combine `H` on its failing state with `h` to show
only open failing reviews.
```

Update the control-socket command descriptions and examples for
`verdict_filter`. Add one concise Unreleased improvement describing the new
composable filter.

- [ ] **Step 2: Format and validate Markdown**

Run:

```bash
make markdown
make markdown-ci
```

Expected: both commands succeed; retain any formatter changes in the docs.

- [ ] **Step 3: Run focused cross-layer tests**

Run:

```bash
go test ./internal/storage ./internal/daemon ./cmd/roborev/tui ./pkg/client/generated
```

Expected: PASS.

- [ ] **Step 4: Run repository quality gates**

Run without live-data overrides or integration tags:

```bash
go test ./...
go build ./...
mise x golangci-lint@2.12.2 -- make lint-ci
prek run --all-files
```

Expected: all commands succeed. If inherited GitHub or PostgreSQL environment
variables cause unrelated integration-style failures, rerun after unsetting
only those specific variables; do not replace `HOME` or point tests at the live
roborev data directory.

- [ ] **Step 5: Review the complete diff and privacy surface**

Run:

```bash
git diff origin/main...HEAD --check
git diff origin/main...HEAD --stat
git status --short
```

Use the privacy-scrub skill before anything leaves the private checkout. Verify
that docs, generated schema, tests, commit messages, and diffs contain only
synthetic paths, IDs, and review data.

- [ ] **Step 6: Commit documentation and generated formatting changes**

Use the mandatory commit skill, review the status/diff, and commit all remaining
feature files with a rationale-first message such as:

```bash
git add docs/integrations/tui.md docs/changelog.md
git commit -m "docs: explain TUI verdict filters"
```

- [ ] **Step 7: Push and verify the remote state**

Follow the repository completion contract without changing branches or
amending history:

```bash
git pull --rebase
git push
git status --short --branch
```

Expected: the current branch reports up to date with its upstream and the
working tree is clean. Do not open or merge a pull request unless separately
requested.
