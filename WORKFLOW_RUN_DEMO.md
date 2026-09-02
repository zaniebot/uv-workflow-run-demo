# uv CI reporting demo

This disposable repository uses uv at `e34768e9e0de1e4085d0ff574a33d1cb4a6baa59`. It explores the
workflow isolation proposed in [uv-dev#958](https://github.com/astral-sh/uv-dev/pull/958), with a
GitHub App reporting the individual results. The imported snapshot replaces a revoked-token fixture
with a plain placeholder. Free runners and longer timeouts are separate commits, following
[uv#18359](https://github.com/astral-sh/uv/pull/18359).

A PR or a push to `main` starts `Build`, which builds the real development binaries. When it
completes, GitHub starts one `CI` workflow through `workflow_run`. That workflow contains all the
migrated test families:

```text
Build
  build-dev-binaries / ...

CI
  prepare
  test-integration
    cargo-nextest
    nushell
    ...
  test-smoke
    linux
    linux aarch64
    macos
    ...
  test-reporting
    failure and rerun
```

The App reports `CI`, `CI / test-integration / nushell`, `CI / test-smoke / linux`, and the other
individual jobs on the source PR commit. There is no waiting meta job. On `main`, the same app
checks attach to the source push commit, so a failed test makes that commit red even when its
`Build` run succeeded.

A child's Details link opens its actual Actions job, including steps and logs. The aggregate check
and the Full CI tree link open the shared `CI` run. Native nesting remains inside that run. GitHub
groups app checks separately from Actions checks on the PR Checks tab. That app list is flat: the
slash-separated paths preserve names, but do not create collapsible groups. Opening the actual `CI`
run is necessary to see the workflow tree. The App does not insert synthetic nodes into the `Build`
graph. Moving another test family later means adding it to this same `CI` workflow and retaining its
existing job path.

The `test-reporting` fixture deliberately fails on attempt 1 and succeeds on a rerun, so there is a
predictable failure to click. It is separate from uv's real tests. Rerunning its actual Actions job
retains the same `CI` run and updates the app checks when the local reporter reconciles it. Native
rerun controls work; the App's own Re-run control needs the future webhook receiver.

The temporary `Zaniebot CI Demo` app runs through a local reporter during this experiment. Its
credentials stay outside Actions. The app implementation is a private draft in
`zaniebot/github-services`.

The first PR trial used `shared-ci-app-reporting` as the temporary default branch, with a small
trial PR targeting it. The repository now uses `main` as its default branch to demonstrate real push
reporting too. The workflow proposal remains a draft against `meta-job-demo`, which preserves the
earlier `main` baseline. `workflow_run` must be installed on the default branch; merely changing a
PR's workflow file does not activate that listener. None of these experiments change uv or uv-dev's
default branch.

The child selects the build's merge SHA from artifact metadata, so its test scripts match the
binaries. Its own workflow SHA identifies the trusted default-branch definition, which is different
from the source PR head. For a push, both SHAs may coincide if `main` has not advanced before the
child starts; the app still reports on the producing commit. The app verifies the repository,
workflow IDs, event, wrapper blob, and source run/attempt before associating checks. The initial
demo supports same-repository PRs and pushes to `main`.

Each side attempts to save a harmless, uniquely named cache marker. The PR run can write in its
merge-ref cache scope, and the push run can write in `main`'s scope; GitHub denies the
`workflow_run` write. This is the Actions cache service's policy, independent of `GITHUB_TOKEN`
permissions. See
[GitHub's cache change](https://github.blog/changelog/2026-06-26-read-only-actions-cache-for-untrusted-triggers/).
The child explicitly grants only `actions: read` and `contents: read` and receives no app
credential. Registry tests retain their restriction to `astral-sh/uv`.

This demonstrates the test-consumer boundary, not complete build isolation: compilation and build
scripts still run in `Build` with that trigger's cache scope.
