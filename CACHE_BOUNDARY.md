# Actions cache boundary experiment

The earlier `workflow_run` trial denied a cache write after a pull request, but allowed one after a
push to `main`. The event name alone therefore did not establish the intended boundary.

These small, manually dispatched workflows test the explicit `cache-mode` setting and existing
branch scoping. They print only the non-secret cache access claims, look up a supplied marker key,
and optionally attempt to save a uniquely named file containing `cache-boundary`. The pinned cache
client predates `cache-mode`, so a denied save must be enforced by the service instead of merely
being skipped by the client.

The checks are:

1. A `write` run on `main` can create a marker.
2. A `read` run on `main` can find that marker but cannot create another.
3. A `write` run on a non-default branch can find the `main` marker and writes in its own scope.
4. `main` and a sibling branch cannot find the non-default branch's marker.

The cache API's recorded `ref` and the save/restore logs are the evidence. A green save step alone
is not evidence of a successful write: `actions/cache` reports permission denials as warnings.

## Results on September 2, 2026

| Case                                                        | Result                                                                                                                                                                      | Evidence                                                                                         |
| ----------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `main`, workflow `cache-mode: write`                        | The token had `main` permission `3`; the marker was saved under `refs/heads/main`.                                                                                          | [Write baseline](https://github.com/zaniebot/uv-workflow-run-demo/actions/runs/33676216901)      |
| `main`, workflow `cache-mode: read`                         | The token still had `main` permission `3`. It found the baseline marker and saved another under `refs/heads/main`.                                                          | [Workflow-level read](https://github.com/zaniebot/uv-workflow-run-demo/actions/runs/33676251033) |
| `cache-boundary`, workflow `cache-mode: write`              | The token had permission `3` for `refs/heads/cache-boundary` and permission `1` for `refs/heads/main`. It found the baseline marker and saved only in its own branch scope. | [Non-default branch](https://github.com/zaniebot/uv-workflow-run-demo/actions/runs/33676251051)  |
| `main` looking up the branch marker                         | Cache miss.                                                                                                                                                                 | [Main lookup](https://github.com/zaniebot/uv-workflow-run-demo/actions/runs/33676306084)         |
| `cache-boundary-sibling` looking up the branch marker       | Cache miss.                                                                                                                                                                 | [Sibling lookup](https://github.com/zaniebot/uv-workflow-run-demo/actions/runs/33676305809)      |
| `main`, job-level `cache-mode: read` and `cache-mode: none` | Both tokens still had `main` permission `3`; both jobs found and saved markers.                                                                                             | [Job-level modes](https://github.com/zaniebot/uv-workflow-run-demo/actions/runs/33676368579)     |

All these runs reported `ACTIONS_CACHE_MODE` as unset. GitHub accepted the declarations, but the
explicit modes were not enforced in this repository. Runner and toolkit support has merged, and the
runner change says emission is gated by the Actions service. That is consistent with a feature that
is not enabled here; the experiment does not identify the service's rollout configuration. Do not
rely on the YAML setting until the issued claims and actual cache operations are restricted.

The existing branch boundary did enforce the property we need: a worker dispatched on a non-default
branch could read trusted `main` caches but could not create entries in that scope. Its own cache
entries were invisible to `main` and sibling branches.

## Consequences for uv

`astral-sh/uv-dev` already provides the preferable worker location. Its protected `main` is a mirror
of `astral-sh/uv/main`, updated by the existing
[sync workflow](https://github.com/astral-sh/uv/blob/2b2545c882841676fe0fecf0f3f0f5a08565d123/.github/workflows/sync-uv-dev.yml).
At the inspected revision, both refs pointed to `2b2545c882841676fe0fecf0f3f0f5a08565d123`. The
[existing uv-dev CI run](https://github.com/astral-sh/uv-dev/actions/runs/33674808578) completed
successfully and produced its own development-binary artifacts. Matching `main` cache keys had
different cache-entry IDs in the two repositories. No additional mirror branch is needed.

For main-branch commits, use the existing mirror CI: `uv/main` advances, the sync updates
`uv-dev/main`, CI builds and tests that exact commit in `uv-dev`, and the app publishes the selected
job results on the same SHA in `uv`. The cache boundary is the executing repository. Merely calling
a reusable workflow in `uv-dev` from an `uv` workflow would retain the caller's execution context
and would not provide that separation.

For upstream PRs, a later consumer can be dispatched on protected `uv-dev/main` with an explicit
source repository, producing run ID, attempt, and tested SHA. It must use reviewed workflow
definitions and an authenticated, read-only artifact handoff. The tested revision is input data; it
does not replace the workflow definition. The app must validate both repository IDs and write checks
only in the source repository. Test jobs receive no upstream write credential or app key.

`uv-dev` also has trusted automation, so its cache must not become an indirect path back into
upstream writes. For example, the existing
[rebase workflow](https://github.com/astral-sh/uv/blob/2b2545c882841676fe0fecf0f3f0f5a08565d123/.github/workflows/rebase-conflicted-pull-request.yml)
restores a Rust cache in its preparation job and passes a bundle to a separately credentialed push
job. Review these consumers and handoffs before broadening which untrusted code runs in
`uv-dev/main`. This is a remaining deployment requirement, not a confirmed exploit in that workflow.

The native `uv-dev` CI run retains its existing nested job tree; app copies on an upstream commit
remain flat. The dedicated non-default branch remains a fallback if a more isolated worker scope is
needed. `cache-mode` is not available for this design. Neither repository nor branch separation
changes the separate question of build scripts executing in the trusted producer.

References:
[GitHub's proposed explicit cache mode](https://github.com/orgs/community/discussions/194493),
[runner support](https://github.com/actions/runner/pull/4538), and
[existing branch restrictions](https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching#restrictions-for-accessing-a-cache).
