# gh-pr — GitHub PR VS Code Plugin: Todo & Plan

A high-performance GitHub Pull Requests & Issues VS Code extension with **exact feature parity** to the official
[GitHub Pull Requests and Issues](https://marketplace.visualstudio.com/items?itemName=GitHub.vscode-pull-request-github)
extension, engineered from the ground up to be significantly faster.

---

## Table of Contents

1. [Goals](#goals)
2. [Feature Checklist](#feature-checklist)
3. [Performance Strategy](#performance-strategy)
4. [Architecture](#architecture)
5. [Implementation Phases](#implementation-phases)
6. [Benchmarks / Definition of "Fast"](#benchmarks--definition-of-fast)

---

## Goals

| Goal | Description |
|------|-------------|
| **Feature parity** | Every user-visible feature present in the official extension is covered |
| **Performance** | PR list, diffs, and checkout are measurably faster (see benchmarks section) |
| **No code yet** | This document is planning only — implementation begins after this plan is approved |

---

## Feature Checklist

### Authentication & Connection

- [ ] GitHub.com OAuth / Personal Access Token (PAT) authentication
- [ ] GitHub Enterprise Server (GHES) support
- [ ] Multiple GitHub account support
- [ ] Secure token storage via VS Code `SecretStorage` API
- [ ] Graceful re-authentication on token expiry

### Pull Request — Browse & Filter

- [ ] Sidebar tree view: list open/closed PRs grouped by query
- [ ] Customisable saved queries (e.g. "Created by me", "Assigned to me", "Review requested")
- [ ] Filter by label, milestone, author, assignee, review state
- [ ] Sort by created date, updated date, or comment count
- [ ] PR status badges (open, closed, draft, merged)
- [ ] Paginate large result sets with "Load more" nodes
- [ ] Search / type-to-filter across visible PRs

### Pull Request — Create

- [ ] "Create Pull Request" command & button
- [ ] Select base branch and compare branch
- [ ] Populate title & description with smart defaults (first commit message, issue reference)
- [ ] Link to issue (auto-close keywords)
- [ ] Set reviewers, assignees, labels, milestone
- [ ] Toggle draft mode
- [ ] Preview diff before submit
- [ ] Template support (`.github/PULL_REQUEST_TEMPLATE.md` and multi-template chooser)

### Pull Request — Review

- [ ] Open PR diff in editor (full file or changed-only view)
- [ ] Side-by-side and inline diff modes
- [ ] Comment on any diff line (single-line and multi-line)
- [ ] Reply to existing comment threads
- [ ] Resolve / unresolve comment threads
- [ ] Suggest changes (code suggestion blocks)
- [ ] Apply a suggestion with one click
- [ ] Submit review: Comment / Approve / Request Changes
- [ ] View all conversations tab (list all unresolved threads)
- [ ] Emoji reactions on comments
- [ ] Edit and delete own comments
- [ ] View PR description in a dedicated webview panel
- [ ] View CI / status checks per commit
- [ ] View requested reviewers and their review state

### Pull Request — Checkout & Merge

- [ ] Checkout PR branch locally
- [ ] Fast sparse-checkout for PRs touching only a subset of files
- [ ] "Exit review mode" — restore previous branch
- [ ] Merge PR (merge commit / squash / rebase)
- [ ] Delete head branch after merge (optional)

### Issue Management

- [ ] List, browse, and filter issues in sidebar
- [ ] Create new issue from VS Code
- [ ] "Start working on issue" — auto-create branch and check out
- [ ] Create issue from a TODO comment in code (code action)
- [ ] Close issue with a commit message keyword
- [ ] Assign, label, and milestone issues

### Editor Integration

- [ ] `@mention` autocomplete for collaborators in comment inputs
- [ ] `#number` autocomplete for issues/PRs in comment inputs
- [ ] Hover cards on `@user` mentions and `#issue` references in documents
- [ ] CodeLens showing open PR count per file
- [ ] Status bar item: current PR / branch indicator

### Notifications & Refresh

- [ ] Periodic background refresh of PR/issue data
- [ ] ETag / `If-None-Match` conditional requests to skip unchanged data
- [ ] Activity badge on sidebar when new reviews or comments arrive
- [ ] Configurable refresh interval

### Settings & Customisation

- [ ] Configure multiple remotes (origin + upstream)
- [ ] Customisable sidebar query definitions
- [ ] Toggle features on/off (issues panel, CodeLens, hover cards, etc.)
- [ ] Configure default merge strategy
- [ ] Proxy support via VS Code HTTP settings

---

## Performance Strategy

This is the core differentiator. The following techniques target the known bottlenecks of the official extension.

### 1. GraphQL over REST — Batch All Reads

| Problem (official) | Solution |
|--------------------|----------|
| N+1 REST calls to populate PR list (one call per PR for details) | Single GraphQL query fetches all visible PRs with fields in one round-trip |
| Separate calls for reviews, comments, status checks | All retrieved in the same batched GraphQL query |

### 2. Persistent Disk Cache

- Cache PR metadata, diff data, and comment threads to an SQLite database in the extension's global storage directory.
- Cache key: `{owner}/{repo}/pr/{number}/v{etag}`
- On startup, serve from cache immediately and refresh in the background.
- Cache size cap with LRU eviction to stay under a configurable limit (default 50 MB).

### 3. Incremental / Delta Refresh

- Use GitHub GraphQL `updatedAt` cursor to fetch only PRs that changed since last poll.
- For comments, use the `since` parameter on the REST comments endpoint, or GraphQL `updatedAt` filter.
- Only re-render tree nodes that actually changed.

### 4. Lazy Loading & Virtualisation

- Tree view shows only the first page (30 items) immediately.
- Remaining pages load on demand (user scrolls / expands).
- Diff files are streamed — the webview renders visible files first.

### 5. Sparse / Partial Checkout for PR Review

- When checking out a PR branch, use `git sparse-checkout` to fetch only files changed in the PR, not the entire working tree.
- Fall back to full checkout only if the user browses outside the changed file set.

### 6. Precomputed Diff Cache

- After fetching a PR, immediately compute and cache the file diff summary in the background.
- Opening a diff file uses the cached computation; no blocking wait.

### 7. Parallel API Fan-out

- Multiple independent GitHub API calls (e.g. label list + assignee list + milestone list for PR creation form) are dispatched in parallel with `Promise.all`, not sequentially.

### 8. Optimistic UI Updates

- When the user submits a comment, adds a reaction, or applies a suggestion, update the UI immediately and reconcile with the server response asynchronously.

### 9. ETag Conditional Requests

- Every GitHub REST request stores the response `ETag`.
- Subsequent requests include `If-None-Match`; a `304 Not Modified` response costs 0 rate-limit and returns quickly.

### 10. Rate-Limit Awareness

- Track remaining `X-RateLimit-Remaining` on every response.
- Pause non-critical background requests when below a configurable threshold (default: 50 remaining).
- Prefer GraphQL (5000 points/hour) for bulk reads and REST only where required.

---

## Architecture

```
gh-pr/
├── src/
│   ├── api/
│   │   ├── graphql/          # GraphQL queries & fragments
│   │   ├── rest/             # REST wrappers (where GraphQL unavailable)
│   │   ├── cache/            # SQLite cache layer
│   │   └── rateLimiter.ts    # Rate-limit tracking & backpressure
│   ├── auth/
│   │   ├── githubAuthProvider.ts
│   │   └── tokenStore.ts
│   ├── views/
│   │   ├── pullRequestsTree/  # Sidebar tree view
│   │   ├── issuesTree/        # Issues sidebar tree view
│   │   └── prWebview/         # Review webview panel
│   ├── commands/              # VS Code command registrations
│   ├── git/
│   │   ├── sparseCheckout.ts  # Sparse-checkout helper
│   │   └── gitHelper.ts
│   ├── notifications/
│   │   └── backgroundPoller.ts
│   ├── statusBar/
│   │   └── prStatusBar.ts
│   └── extension.ts           # Entry point
├── package.json
├── tsconfig.json
├── README.md
└── TODO.md
```

### Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **TypeScript** | Same language as the official extension; mature VS Code extension ecosystem |
| **SQLite (via `better-sqlite3`)** | Synchronous, fast, zero-config disk cache; avoids async overhead for cache hits |
| **GraphQL-first** | Dramatically reduces API round-trips and byte transfer for list views |
| **No React/Vue in webview** | Use lightweight vanilla TS + VS Code webview toolkit to reduce bundle size and startup time |
| **VS Code TreeView API** | Native virtualisation, no custom scroll implementation needed |

---

## Implementation Phases

### Phase 0 — Project Scaffolding *(this plan)*

- [x] Create `TODO.md` with full plan
- [ ] Scaffold extension with `yo code` (TypeScript, ESLint)
- [ ] Set up CI (GitHub Actions: lint, build, test)
- [ ] Configure `esbuild` for fast bundling (replaces webpack)

### Phase 1 — Authentication & API Layer

- [ ] GitHub OAuth via VS Code built-in `github` auth provider
- [ ] PAT fallback
- [ ] GraphQL client with batching support
- [ ] REST client with ETag caching
- [ ] SQLite cache layer
- [ ] Rate-limit manager

### Phase 2 — Pull Request List (Read-Only)

- [ ] Sidebar tree view with default queries
- [ ] Customisable queries via settings
- [ ] PR status badges
- [ ] Background polling with delta refresh
- [ ] "Load more" pagination nodes

### Phase 3 — Pull Request Detail & Review

- [ ] PR description webview
- [ ] Changed files tree
- [ ] Diff editor integration (using VS Code `workspace.openTextDocument` + virtual FS)
- [ ] Inline comment threads
- [ ] Review submission (approve / request changes / comment)
- [ ] Suggest changes & apply suggestion

### Phase 4 — PR Checkout & Merge

- [ ] Checkout PR branch (sparse if possible)
- [ ] Exit review mode
- [ ] Merge with strategy selection

### Phase 5 — Issue Management

- [ ] Issues sidebar tree view
- [ ] Create issue command
- [ ] "Start working on issue" workflow
- [ ] Create issue from TODO code action

### Phase 6 — Editor Integration

- [ ] `@mention` and `#issue` autocomplete
- [ ] Hover cards
- [ ] CodeLens
- [ ] Status bar item

### Phase 7 — Polish & Parity Audit

- [ ] Side-by-side comparison against official extension feature list
- [ ] Performance benchmarks (see below)
- [ ] Accessibility audit
- [ ] Localisation / i18n groundwork

---

## Benchmarks / Definition of "Fast"

The following measurements will be taken against the official extension on the same machine and network:

| Metric | Official (baseline) | Target |
|--------|---------------------|--------|
| PR list initial load (30 PRs) | ~3–6 s | **< 1 s** (from cache); **< 2 s** (cold) |
| PR diff open (first file) | ~1–3 s | **< 500 ms** |
| PR checkout (sparse) | N/A | **50 % faster** than full checkout |
| Memory usage (idle, 100 PRs loaded) | ~120 MB | **< 60 MB** |
| API calls to render PR list (30 items) | ~30–60 REST calls | **1 GraphQL call** |
| Background refresh (no changes) | Full re-fetch | **0 bytes** transferred (ETag 304) |

Benchmarks will be automated using VS Code's `extensionTelemetry` hooks and a local mock GitHub server.
