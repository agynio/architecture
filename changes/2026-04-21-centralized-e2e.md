# Centralized E2E Tests

## Target

- [E2E Testing](../architecture/operations/e2e-testing.md)
- [E2E Testing — CI Integration](../architecture/operations/e2e-testing.md#ci-integration)
- [CI/CD — E2E Job](../architecture/operations/ci-cd.md#e2e-job)
- [New Service Development — E2E Tests](../architecture/operations/new-service.md#e2e-tests)

## Delta

### E2E repository

- A single repository [`agynio/e2e`](https://github.com/agynio/e2e) does not yet exist. All E2E tests still live in per-service `test/e2e/` directories.
- Existing tests must be relocated into `agynio/e2e`, grouped into suites under `suites/<name>/`. A suite is a directory with a `suite.yaml` declaring its container image, `select` command (prints matching test identifiers for a tag set), and `run` command (executes the filtered tests).
- Initial suites to define: `go-core` (Go toolchain, cross-service gRPC tests), `go-terraform` (Go + Terraform CLI, provider acceptance tests), `playwright` (Chromium, UI tests). More suites are added over time by creating new directories.
- `devspace.yaml` at the root of `agynio/e2e` contains only the `test-e2e` pipeline. The pipeline auto-discovers suites by scanning `suites/*/suite.yaml` — it does not list them statically.
- Shared fixtures move to `agynio/e2e/testdata/`.

### Service tagging

- Every test declares the services it exercises via native tags: Go build tags (`//go:build e2e && svc_<name> && …`), Playwright `test.describe` tags (`@svc_<name>`).
- `--tag <name>` is the only filter on the pipeline. It is repeatable / comma-separated. No special `--service` flag — service selection is just `--tag svc_<name>`.

### Skip suites with no matching tests

- Before deploying any test pod, the pipeline runs each suite's `select` command in an ephemeral container (using that suite's image) against the requested tags. If `select` output is empty, the suite is skipped — no pod, no sync, no run.

### Service repositories

- Each service repo removes its `test/e2e/` directory.
- Each service `devspace.yaml` removes `deployments.e2e-runner`, `dev.e2e-runner`, `pipelines.test-e2e`, and `commands.test-e2e`. Only the `dev` pipeline (and its `-w` watch flag) remains.

### CI

- `agynio/bootstrap` publishes a composite action at `.github/actions/provision/` that encapsulates cluster bring-up: disk reclaim, self-checkout, pinned tool installs (kubectl, k3d, Terraform), `apply.sh`, `verify.sh`, and `KUBECONFIG` exported into the job environment. Cluster-level versions and scripts live only here.
- `agynio/e2e` publishes a composite action at `.github/actions/run-tests/` that checks itself out, installs DevSpace, runs `devspace run test-e2e` with `--tag` derived from its `service:` / `tag:` inputs, and uploads test artifacts. Suite discovery, pre-scanning, and pod lifecycle live inside the DevSpace pipeline invoked by this action.
- Each service's `ci.yml` e2e job is a short three-step sequence: `provision` (from `agynio/bootstrap`), a service-owned deploy step (usually `devspace dev`, but flexible — a service with custom bring-up owns that logic here), and `run-tests` (from `agynio/e2e`).
- `agynio/e2e` also grows a workflow that composes the same two actions (with no deploy step in between) on every PR and push to its own `main`, running every suite against a cluster with all services at pinned bootstrap images.

### Terraform provider tests

- Acceptance tests currently in [`agynio/terraform-provider-agyn`](https://github.com/agynio/terraform-provider-agyn) relocate to `agynio/e2e/suites/go-terraform/tests/`. The suite's declared image must include the `terraform` CLI.

### Breaking-change workflow

- Teams adopt expand-contract for any change that would otherwise require synchronizing a service PR with an E2E PR. New behavior lands alongside old; tests for new behavior are added to `agynio/e2e` without removing tests for old behavior; a cleanup PR removes both when no consumer depends on the old path.

## Acceptance Signal

- `agynio/e2e` exists and contains every test previously under `agynio/*/test/e2e/`, grouped into suites under `suites/`.
- No service repository contains a `test/e2e/` directory or a `test-e2e` DevSpace command.
- `agynio/bootstrap/.github/actions/provision` exists and is the sole way CI provisions a bootstrap cluster.
- `agynio/e2e/.github/actions/run-tests` exists and is the sole way CI runs E2E suites.
- Every service's `ci.yml` e2e job composes those two actions with a service-owned deploy step in between. No service repo references `apply.sh`, `verify.sh`, kubectl/k3d/Terraform versions, or `devspace run test-e2e` directly.
- No workflow outside `agynio/bootstrap` references cluster-provisioning internals; no workflow outside `agynio/e2e` references the `test-e2e` DevSpace pipeline directly.
- A PR that does not change UI does not deploy the Playwright suite's pod (verified by CI logs).
- `agynio/e2e@main` has a passing CI run that executes every suite.
- The Terraform provider repo no longer contains acceptance tests under `test/e2e/`.

## Notes

- TestLLM (`agynio/testllm`, `agynio/testllm-suites`, `agynio/terraform-provider-testllm`) is unaffected — it was already independently versioned. The only change is that E2E tests consuming TestLLM suites now live in `agynio/e2e` instead of each service repo.
- No service image is built or pushed for E2E. The service under test runs from synced source inside a dev container via `devspace dev`, exactly as it did before.
