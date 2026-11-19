# CLAUDE.md (fork branch `review-plan` only)

This checkout is a fork of `github.com/rfdrew/terraform-provider-pbs`. The branch `review-plan` holds the review notes
and this file. It never goes upstream.

The current work is `fork-notes/plans/2026-11-19-review-and-upstream-fixes.md`. Read it first. Results go in
`fork-notes/review/`.

## Rules

- `upstream` is `rfdrew`. `origin` is the user's fork.
- Fix branches start from `upstream/main` in their own worktree. Never commit `fork-notes/` or this file to one.
- One fix per pull request, with the test that proves it. Keep the change as small as possible.
- Ask the user before opening any pull request, issue, or comment upstream.
- Acceptance tests run only against the throwaway PBS from `make pbs-up`. Never against a real server.
- No secrets in commits, pull requests, or logs.
- Run Go and Terraform inside `nix shell nixpkgs#go nixpkgs#terraform nixpkgs#golangci-lint nixpkgs#shellcheck`.
- Writing: no em-dashes, no emoji, short active sentences.
