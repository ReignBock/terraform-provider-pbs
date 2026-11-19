# Review rfdrew/pbs and Upstream Fixes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or
> superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Decide whether `rfdrew/pbs` can configure the PBS server in `infrastructure/hardware/truenas`. For each
serious defect, prove it with a test and send upstream one small fix per pull request.

**Architecture:** This checkout is a fork of `github.com/rfdrew/terraform-provider-pbs`. The branch `review-plan`
holds only `fork-notes/` and `CLAUDE.md`, and never goes upstream. Each fix lives on its own branch, cut from
`upstream/main` in its own git worktree, and becomes one pull request from the fork to `rfdrew/terraform-provider-pbs`
`main`. The review reads the code for the resources the truenas repo needs and runs the provider against a throwaway
PBS.

**Tech Stack:** Go 1.26.5 (from `go.mod`), terraform-plugin-framework, the `make` targets upstream ships, Docker for
the throwaway PBS (`make pbs-up`), OpenTofu 1.12 (`tofu`), Terraform 1.14 for the acceptance tests.

**Spec:** No separate spec. The requirements are in "What the truenas repo needs" below. They come from
`~/dev/infrastructure/hardware/truenas` (branch `pbs-on-truenas`): `src/configure.yml`, `src/group_vars/all.yml`,
`src/library/pbs_*.py`, `README.md`, and `docs/superpowers/specs/2026-11-18-pbs-on-truenas-design.md`.

## What the truenas repo needs

The truenas repo runs PBS 4.2.12 (`dockurr/proxmox-backup:4.2.12`) on TrueNAS. GitLab CI applies the configuration
with OpenTofu from `registry.powell.network/infrastructure/opentofu:latest`. CI authenticates with the API token
`root@pam!ci` (privilege separation off) and pins the certificate with the CI variable `PBS_FINGERPRINT`. The
provider must manage:

| Object | Settings the repo sets |
|---|---|
| Datastore `data` at `/datastore` (a ZFS dataset, 5 TiB quota) | `gc-schedule` `05:00`, `notification-mode` `notification-system` |
| Prune job `data-prune` on `data` | schedule `04:00`, keep-daily 7, keep-weekly 4, keep-monthly 2 |
| Verify job `data-verify` on `data` | schedule `sun 06:00`, ignore-verified true, outdated-after 30 |
| Sync jobs | none, and none may appear |
| Webhook notification endpoint `ntfy` | POST to the ntfy URL, headers `Authorization: Bearer {{ secrets.token }}` and `Title: {{ title }}`, body `{{ message }}`, secret `token` |
| Notification matcher `ntfy-alerts` | target `ntfy`, mode `any`, severities `error` and `warning` |
| User `pve@pbs` | comment `Proxmox VE backups` |
| ACL | `DatastoreBackup` for `pve@pbs` on `/datastore/data`, propagate true |

Objects the configuration does not manage must survive every apply and destroy: `root@pam`, tokens under it, token
ACLs (`user@realm!name`), built-in objects, namespaces, and backup content.

## Global Constraints

- One fix per pull request. A pull request changes the least code that fixes its one defect, plus the test that
  proves it.
- Every fix starts with a test that fails on `upstream/main` and passes with the fix. Prefer a unit test with the
  fake transport (`internal/pbsclient/fake_transport_test.go` style). Use an acceptance test only when the defect
  needs the real PBS API.
- Fix branches start from `upstream/main`, never from `review-plan`. `fork-notes/` and this branch's `CLAUDE.md` never
  appear in a fix branch.
- Follow upstream's conventions: commit subjects in the imperative with no prefix ("Fix ..."), branch names
  `fix/<slug>`, an `## [Unreleased]` entry in `CHANGELOG.md` under `### Fixed` (Keep a Changelog), `gofmt`,
  `go vet`, golangci-lint v2.15.0, and `make docs` output committed when a schema or description changes.
- Acceptance tests run only against the throwaway PBS from `make pbs-up`. Never point `PBS_ENDPOINT` at the truenas
  PBS (`10.140.10.40:8007`) or any other real server.
- No secrets in commits, pull request text, issue text, or logs. The throwaway PBS token is test data, but do not
  paste it anywhere either.
- Opening a pull request upstream is outward-facing. The user approves the findings list (Task 3) before any pull
  request opens. Pull requests are opened from the GitHub web page, because `gh` is not installed.
- Writing rules for commits, pull requests, and notes: no em-dashes, no emoji, short active sentences, one term per
  concept.

## Review Focus

The failure modes most likely to hurt the truenas repo. Task 2 checks each one, and a confirmed one becomes a finding.

1. Destroying or replacing `pbs_datastore` deletes backup data, or a plan shows a replace where an in-place update
   would do. Expected: data stays unless `destroy_data` is true, and changing `gc_schedule` updates in place.
2. Removing an optional attribute from the configuration leaves the old value on PBS. PBS clears a field only when
   the request names it in `delete`. Expected: the apply sends `delete=<field>` and the next plan is empty.
3. A secret shows up in plan output, in state, or in `TF_LOG=DEBUG` output: the API token, the webhook secret, or a
   header value. Expected: write-only or sensitive, and redacted in logs.
4. A request reaches PBS without the fingerprint check: a second connection, a redirect, a data source, the version
   probe, or the certificate-change path (`ChangeCertificate`). Expected: every TLS connection verifies the pin.
5. A value PBS normalizes causes a diff on every plan: schedules (`05:00` against `5:00`), severity order, booleans
   that PBS omits when they are false, or header and secret round-trips. Expected: a second apply plans no changes.

---

### Task 0: Fork, toolchain, and baseline

**Files:**
- Create: `fork-notes/review/baseline.md`

**Interfaces:**
- Produces: remotes `origin` (the user's fork) and `upstream` (`rfdrew`), a working `go` and `terraform`, and a
  baseline that later tasks compare against.

- [ ] **Step 1: The user forks the repository.** On github.com, the user opens `rfdrew/terraform-provider-pbs` and
  clicks Fork. The user reports the fork's URL. This step needs the user. The agent cannot do it.
- [ ] **Step 2: Trust GitHub's SSH host key.** `~/.ssh/known_hosts` has no entry for `github.com`. Compare the output
  of `ssh-keyscan -t ed25519 github.com | ssh-keygen -lf -` with the ED25519 fingerprint published at
  https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints. Append
  the key only if they match. Then `ssh -T git@github.com` must greet the user's account.
- [ ] **Step 3: Add the fork and push the plan branch.**

```bash
git remote add origin git@github.com:<the user's account>/terraform-provider-pbs.git
git push -u origin review-plan
```

- [ ] **Step 4: Get the toolchain.** Run every Go and Terraform command inside
  `nix shell nixpkgs#go nixpkgs#terraform nixpkgs#golangci-lint nixpkgs#shellcheck`. If the `go` from nixpkgs is
  older than 1.26.5, set `GOTOOLCHAIN=go1.26.5` so Go downloads the right one. Expected: `go version` reports 1.26.5
  or newer, and `terraform version` reports 1.14 or newer.
- [ ] **Step 5: Run the unit tests on `upstream/main`.** Run `make test`. Expected: all pass. Record the commit and
  the result in `fork-notes/review/baseline.md`.
- [ ] **Step 6: Start the throwaway PBS.** Run `make pbs-up`, which prints `PBS_*` exports. Export them in the shell
  only. Do not write them into any file. Note the PBS version it started. If it is not 4.2.x, check
  `scripts/bootstrap-test-pbs.sh` for a way to pick the 4.2 line, because the truenas repo runs 4.2.12.
- [ ] **Step 7: Run the acceptance tests for the resources in scope.**

```bash
TF_ACC=1 go test ./internal/provider/ -v -timeout 30m \
  -run 'Datastore|PruneJob|VerifyJob|SyncJob|NotificationEndpointWebhook|NotificationMatcher|User|ACL|Acl|APIToken|ApiToken'
```

  Check the real test names with `grep -h '^func TestAcc' internal/provider/*_test.go` first and adjust the
  pattern. Expected: all pass. Record failures in `baseline.md`. A failure here is either an upstream bug or a stand
  problem, so investigate before Task 2.
- [ ] **Step 8: Commit.**

```bash
git add fork-notes/review/baseline.md
git commit -m "Record the review baseline"
```

### Task 1: Supply chain and compatibility

**Files:**
- Create: `fork-notes/review/findings.md` (Task 2 appends to it)

**Interfaces:**
- Produces: `findings.md`, with one entry per finding in this shape:

```markdown
### F<n>: <one-line claim>
- Severity: blocking | serious | minor
- Where: <file>:<line>
- Evidence: <test name, command output, or code quote that proves it>
- Effect on truenas: <what breaks or leaks>
- Upstream fix size: <rough lines changed>
```

- [ ] **Step 1: Check release signing.** Download the `v1.0.0` release assets. Verify the `SHA256SUMS` file against
  its `.sig` with the GPG key the release workflow names, and check that a binary's checksum matches. Record the key
  fingerprint.
- [ ] **Step 2: Check the registries.** Confirm `rfdrew/pbs` resolves on registry.terraform.io and on the OpenTofu
  registry (`tofu init` with `required_providers { pbs = { source = "rfdrew/pbs", version = "1.0.0" } }` in a
  scratch directory outside the repo). If the OpenTofu registry lacks it, record that as a blocking finding for the
  truenas CI, which uses `tofu`. The fix then belongs in the truenas repo (a provider mirror), not upstream.
- [ ] **Step 3: Check OpenTofu support for write-only attributes.** In the same scratch directory, plan a
  `pbs_notification_endpoint_webhook` with a secret against the throwaway PBS, using `tofu` 1.12. Expected: the plan
  works and the secret is absent from `tofu show -json`. Record the minimum OpenTofu version the truenas image needs.
- [ ] **Step 4: Scan dependencies.** Run `go run golang.org/x/vuln/cmd/govulncheck@latest ./...`. Record any
  vulnerability in code the provider reaches.
- [ ] **Step 5: Commit.**

```bash
git add fork-notes/review/findings.md
git commit -m "Record the supply chain review"
```

### Task 2: Code review of the resources in scope

**Files:**
- Modify: `fork-notes/review/findings.md`
- Read: `internal/pbsclient/{client,session,tls,errors,params}.go`, `internal/provider/provider.go`, and
  `internal/provider/resource_{datastore,prune_job,verify_job,sync_job,notification_endpoint_webhook,notification_matcher,user,acl,api_token}.go`
  with the `internal/pbsclient` files they call

**Interfaces:**
- Consumes: the throwaway PBS from Task 0 and the finding format from Task 1.
- Produces: the complete findings list. Each finding has evidence. A suspicion without evidence goes under a
  closing "Unconfirmed" heading, not in the list.

- [ ] **Step 1: Transport.** Trace every path that opens a connection: the provider client, data sources, the
  version probe, task polling, and `ChangeCertificate`. Confirm that each one runs `VerifyConnection` with the pin, and
  that redirects cannot reach another host without it. Check whether the HTTP transport honors `HTTPS_PROXY`. The
  truenas runners set it for internet access, and PBS must not be reached through the proxy unless `NO_PROXY` covers
  it. Record the behavior even if it is not a defect, because the truenas CI job needs to know it.
- [ ] **Step 2: Secrets.** For `api_token`, the webhook `secret` and header values, and `pbs_api_token`'s output
  secret, check `Sensitive` and `WriteOnly` in the schema, what ends up in state (`terraform show -json` after an
  apply against the throwaway PBS), and what `TF_LOG=DEBUG` prints (request and response bodies and headers). Review
  Focus 3.
- [ ] **Step 3: Datastore.** Against the throwaway PBS: create a datastore on an existing empty directory, change
  `gc_schedule`, destroy without `destroy_data`, and check that the directory and its chunks remain. Then recreate
  on the same path, which the truenas repo needs after a state loss. Also check whether `import` works for an
  existing datastore. Review Focus 1.
- [ ] **Step 4: Field clearing.** For the prune job, verify job, webhook, matcher, and user: apply with an optional
  attribute set, remove it, apply, and read the object from the PBS API. Expected: the field is gone and the next
  plan is empty. Review Focus 2.
- [ ] **Step 5: Stable plans.** Apply a configuration shaped like "What the truenas repo needs" against the throwaway
  PBS (use a local ntfy-like URL, nothing has to receive the call), then plan again. Expected: "No changes". Any diff
  is a finding. Review Focus 5.
- [ ] **Step 6: Scope of deletes.** Destroy `pbs_acl`, `pbs_user`, and the matcher. Check that only that object goes:
  other ACLs on the same path, token ACLs, `root@pam`, and its tokens stay. Check that `pbs_user` destroy removes the
  user's tokens (expected PBS behavior), and that the docs say so.
- [ ] **Step 7: Error handling.** With the throwaway PBS stopped (`make pbs-down`), run a plan. Expected: a clear
  error naming the endpoint, not a crash. With a wrong `tls_cert_fingerprint`, expected: an error that shows the
  presented fingerprint and sends no request.
- [ ] **Step 8: Commit.**

```bash
git add fork-notes/review/findings.md
git commit -m "Record the code review findings"
```

### Task 3: Triage with the user

**Files:**
- Modify: `fork-notes/review/findings.md`

- [ ] **Step 1: Classify each finding.** A finding qualifies for an upstream pull request ("glaring") if it is any of
  these: a secret leak, a request without the pin, data loss on destroy or replace, or wrong behavior in a resource
  the truenas repo uses (a failed apply, a diff on every plan, a field that cannot be cleared). Everything else stays
  in the notes as minor.
- [ ] **Step 2: Present the list to the user.** Show each qualifying finding with its evidence and the size of the
  fix. Stop and wait. The user picks which ones become pull requests and in what order. Record the decisions in
  `findings.md` under "Decisions".
- [ ] **Step 3: Commit.**

```bash
git add fork-notes/review/findings.md
git commit -m "Record the triage decisions"
```

### Task 4: One fix per approved finding (repeat per finding)

Repeat this task once per approved finding, in the order the user chose. Never put two findings in one branch.

**Files (per finding):**
- Create or modify: one test file next to the code at fault (`internal/pbsclient/<area>_test.go` or
  `internal/provider/<resource>_test.go`, or `<resource>_acc_test.go` for an acceptance test)
- Modify: the one source file at fault, `CHANGELOG.md`, and `docs/` only if `make docs` changes it
- Create on `review-plan`: an entry in `fork-notes/review/prs.md`

**Interfaces:**
- Consumes: finding F<n> from `findings.md`.
- Produces: the branch `fix/<slug>` on the fork, and a pull request URL in `prs.md`.

- [ ] **Step 1: Make the worktree.**

```bash
git fetch upstream
git worktree add ../terraform-provider-pbs-fix-<slug> -b fix/<slug> upstream/main
cd ../terraform-provider-pbs-fix-<slug>
```

- [ ] **Step 2: Write the failing test.** Name it after the behavior, in upstream's style (`TestClient...`,
  `Test<Resource>...`, `TestAcc<Resource>...`). It asserts the expected behavior from the finding, for example "the
  update sends `delete=comment`" or "the debug log holds no `Authorization` value".
- [ ] **Step 3: Run it and watch it fail.** `go test ./internal/... -run '<TestName>' -v` (add `TF_ACC=1` and the
  `PBS_*` exports for an acceptance test). Expected: FAIL, showing the defect from the finding. If it passes, the
  finding is wrong. Go back to `findings.md`, mark it, and drop the branch.
- [ ] **Step 4: Make the smallest fix.** Change only the code at fault. No refactors, renames, or cleanups next to
  it, even if they look right.
- [ ] **Step 5: Run it and watch it pass.** The same command. Expected: PASS.
- [ ] **Step 6: Run upstream's checks.** `gofmt -l .` prints nothing, `go vet ./...` passes, `golangci-lint run ./...`
  passes, `make test` passes, and `make docs` leaves `git status` clean apart from intended changes. For an
  acceptance test, rerun the in-scope acceptance tests from Task 0, Step 7.
- [ ] **Step 7: Add the changelog entry.** Under `## [Unreleased]`, `### Fixed`, one sentence a user understands:
  what went wrong and what happens now. Create the `[Unreleased]` heading if `main` has none.
- [ ] **Step 8: Commit and push.**

```bash
git add <test file> <source file> CHANGELOG.md
git commit -m "Fix <what was wrong>"
git push -u origin fix/<slug>
```

- [ ] **Step 9: Ask the user before opening the pull request.** Show the diff stat, the test, and the draft pull
  request text below. Wait for approval. Then give the user the compare URL
  `https://github.com/rfdrew/terraform-provider-pbs/compare/main...<the user's account>:terraform-provider-pbs:fix/<slug>`
  to open it.

```markdown
<one paragraph: the defect, as a user meets it, and how to reproduce it>

The test `<TestName>` reproduces it: it fails on main and passes with this change.

The fix: <one or two sentences>.
```

- [ ] **Step 10: Record it.** Back on `review-plan`, append to `fork-notes/review/prs.md`:
  `- F<n>: <claim> - fix/<slug> - <PR URL> - open`. Commit with "Record the pull request for F<n>". Remove the
  worktree with `git worktree remove ../terraform-provider-pbs-fix-<slug>` only after upstream merges or closes it.

### Task 5: Hand back to the truenas session

**Files:**
- Create: `fork-notes/review/handoff.md`

- [ ] **Step 1: Write the verdict.** One of: "use upstream 1.0.0 as is", "use upstream once these pull requests
  ship", or "do not use". Include: the provider version to pin, the minimum OpenTofu version, whether `HTTPS_PROXY`
  or `NO_PROXY` matters for the CI job, the token and fingerprint attributes to set, open pull requests and what the
  truenas repo must avoid until they ship, and anything Task 2 found that changes the truenas design (for example,
  datastore import, or how the webhook secret round-trips).
- [ ] **Step 2: Commit and push.**

```bash
git add fork-notes/review/handoff.md
git commit -m "Write the handoff for the truenas repo"
git push origin review-plan
```

- [ ] **Step 3: Tell the user** that the truenas session can resume from `fork-notes/review/handoff.md`.
