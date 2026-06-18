# Contributing to BimRoss

Org-wide conventions that apply across every repo. Repo-specific rules live in each repo's own `CONTRIBUTING.md` or `README.md`.

## Pagination discipline

Most paged APIs we touch (Slack, GitHub, GA4, Search Console, Cloudflare) silently cap a single call. A one-shot read against a paged endpoint looks right until the cap bites, then numbers under-report without an error. Pagination is the default approach when reading a paged API, not an afterthought.

Two shapes to keep straight:

- **Probe / sample.** "Top 5 most recent PRs." Explicit `--limit N` (or equivalent) is the right answer. No loop. The cap is the intent.
- **Walk / audit.** "Every PR in this repo since X." Cursor or page-number loop until the endpoint says "no more rows." Cap with a caller-supplied `max_rows` so a runaway loop can't burn quota.

If a walk truncates at `max_rows` without seeing the end-of-data signal, surface that to the operator. Silent truncation reads as "covered everything" when it didn't.

### GitHub (`gh` CLI)

- *Bounded probe / sample.* Explicit `--limit N` is correct. Don't add `--paginate`. Examples: `gh pr list --limit 1` to probe for an open PR, `gh pr list --state merged --limit 5` to sample merge-message style.
- *Walk every PR / issue / run.* Use `gh ... --paginate`, not a hand-rolled cursor loop. `--paginate` can be slow on big repos. Cap the date range with `--search 'created:>YYYY-MM-DD'` when only recent activity matters.
- *Programmatic use across many repos.* If `google/go-github` shows up, write a `Paginate[T]` helper (cousin of `slackpaginate`). Don't reach for ad-hoc cursor loops.
- *Search endpoints.* `/search/issues` and `/search/repos` have a hard 1000-result ceiling. Paginating past page 10 doesn't help. Filter the query instead.

### Slack

See [BimRoss/claude-code-joanne#116](https://github.com/BimRoss/claude-code-joanne/issues/116) and the `slackpaginate` helper (`cmd/joanne/slackpaginate/paginate.go`). Use it for every `conversations.history`, `conversations.replies`, `users.list`, and `conversations.list` call. Walks return a `MaxPages` truncation signal so the caller can tell the difference between "no more data" and "we stopped early."

### GA4, Search Console, Cloudflare

Per-surface tickets in `BimRoss/claude-code-ross`:

- GA4: [#249](https://github.com/BimRoss/claude-code-ross/issues/249)
- Search Console: [#250](https://github.com/BimRoss/claude-code-ross/issues/250)
- Cloudflare: tracked alongside the others

Each of those wraps the same probe vs walk discipline against its native paging mechanism (`nextPageToken`, `startRow`, `result_info.cursors`). Read the per-surface ticket before adding a new call site.

## Releasing

See [`RELEASING.md`](./RELEASING.md) for the org-wide release flow, the reusable `gitops-release` workflow, and per-repo deploy profiles.
