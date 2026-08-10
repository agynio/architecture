# What `agyn local` Shows While It Works

## Target

- [agyn-cli — Preflight](../architecture/agyn-cli.md#preflight)
- [agyn-cli — Progress](../architecture/agyn-cli.md#progress)
- [agyn-cli — Readiness](../architecture/agyn-cli.md#readiness)
- [agyn-cli — Upgrade](../architecture/agyn-cli.md#upgrade)
- [agyn-cli — Local Platform Commands](../architecture/agyn-cli.md#local-platform-commands-agyn-local)

## Delta

`agyn local start` and `agyn local upgrade` do the right work and report it as the tooling underneath happens to phrase it. Both are the first thing a new user runs, and both take minutes.

### Preflight

- `start` checks the three host tools and stops with a fix hint when one is absent. It never offers to install them, so the first run of the product ends by asking the user to go and read a package manager's documentation.
- A tool that is present but older than the image's `lima.yaml` needs passes the check. The failure lands later, from `limactl`, naming neither the tool version nor the fix.
- Nothing checks free disk space. The image is ~900 MB compressed and ~4.6 GB decompressed, so a full disk fails partway through decompression — after the download.
- `doctor` reports and cannot fix, so a machine being prepared before a first start still needs the commands typed by hand.

Desired state: the preflight is one set of checks with one install offer behind it, reached from both commands. `start` runs it every time; `doctor` is it standalone; `doctor --fix` and `start --install-deps` accept the offer without prompting.

### Progress

- Underlying tool output goes to the terminal. An upgrade shows eight identical `AuthorizationPolicy` warnings, klog lines with a PID in them, and a Helm release listing; a start shows a decompression notice and a raw byte counter.
- Waiting for readiness prints a static line and then nothing for up to three minutes, which is indistinguishable from a hang.
- Nothing reports what a step is waiting on or what it cost.
- The endpoint list prints three URLs, so a finished `start` ends in a menu rather than a next step.

Desired state: steps, animated on a TTY and printed once when not, with tool output in a log that a failing step names. One call to action at the end of `start` — the console URL, once.

### Errors on the success path

- Credential provisioning runs as soon as the endpoints answer, which is before the platform has provisioned its system organization. The step fails, prints its error and a recovery command, and `start` continues to the endpoint list and exits `0`. A first run therefore ends with an error and a success.
- Readiness is defined as "the endpoints answer", which is not the condition the next step needs.

Desired state: readiness includes the system organization having an assigned id; provisioning runs only after that; a failed step fails the command.

### Upgrade

- The chart version being moved to is visible only in Helm's own output, and only after the fact.
- An upgrade of a release already at the newest chart runs anyway and reports a new revision, so "nothing to do" and "everything was replaced" read alike.
- Rollout waiting is silent for as long as it takes.

## Acceptance Signal

`agyn local start` on a machine missing `limactl` offers to install it, shows the command it will run, and continues into the same run once it succeeds. Declining prints the command and stops.

`agyn local start` on a full disk fails before downloading anything, naming the space required and the space available.

A first `agyn local start` prints no error, exits `0`, and ends with exactly one URL: the console.

A `start` interrupted at any step leaves a terminal that says which step was in progress; no step is silent for more than a few seconds without saying what it is waiting on.

`agyn local start` and `agyn local upgrade` print no `helm`, `kubectl` or `limactl` output. A failing step prints the tail of the log and its path; `--debug` streams the tool output as before.

Piping either command to a file yields a readable list of steps with no control characters.

`agyn local upgrade` names the installed and target chart versions before changing anything, and ends in one line naming both. Run twice, the second run reports the release as already current and performs no upgrade.

`agyn local status` remains the way to see every endpoint.

## Notes

- Two errors visible in the reported output are already closed in `agyn-cli` and reach users on the next release: the platform namespace rename (`namespaces "platform" not found`) and the bootstrap-token reinstall step after an upgrade. They are not part of this delta; the ordering fault behind the credential failure is.
- The preflight's install offer is the first thing in the CLI that changes the machine outside `~/.agyn`. It follows `--install-ca`: prompted on a TTY, explicit flags either way, and not implied by `-y`.
- Readiness reading the system organization declaration is the same read `agyn local credentials` already performs. It becomes a wait condition rather than a step that fails when it is early.
- Naming the workloads still rolling out during a wait requires reading workload state from inside the VM, which the CLI already does over the `limactl` channel for the CA and the organization.
