# uv workflow_run demo

This disposable repository uses the uv source at `e34768e9e0de1e4085d0ff574a33d1cb4a6baa59` and the
workflow change proposed in [uv-dev#958](https://github.com/astral-sh/uv-dev/pull/958). The imported
snapshot replaces a revoked-token test fixture with a plain placeholder.

The commits keep the production workflow change, free-runner and timeout adjustments, and this
demo's trigger configuration separate. The free-runner settings follow
[uv#18359](https://github.com/astral-sh/uv/pull/18359).

Opening a PR runs the actual development binary builds on standard GitHub-hosted runners. A separate
`workflow_run` starts alongside `CI` and waits for its `integration tests` meta job, which starts
after the binary builds succeed. The meta job waits for the isolated tests and reports their result
in the original CI run, with links to each job in its summary. Both sides use only `actions: read`
for the handoff and match the CI run ID and attempt number.

The demo selects the build's merge SHA from artifact metadata so test scripts match the binaries.
The upstream draft instead accepts only pushes to `main` and can use `head_sha` directly. To rerun
the tests and update the CI result, rerun the `integration tests` job in `CI`.

Each workflow attempts to save a harmless, uniquely named cache marker. The PR run should save
within its merge-ref scope; the `workflow_run` save should be denied by GitHub.

Use the pull request's `integration tests` check to reach the meta job and its summary of individual
integration results. The demo runs the build and integration suites; the remaining uv workflows are
omitted. Registry tests retain their existing restriction to `astral-sh/uv`.
