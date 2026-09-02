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

A viable next design keeps the build producer on `main` and dispatches test consumers on a
dedicated, non-default worker branch. The worker branch contains reviewed workflow definitions; the
selected uv commit and producing run ID are inputs, rather than permission to replace the worker's
workflow definition. Tests can continue to download binaries from the exact producing run and check
out the matching commit. The worker token stays at `contents: read` and `actions: read`, with no app
credential or inherited secrets.

This protects the trusted branch's cache while still allowing writes in the worker branch's own
scope. A separate worker repository would isolate that scope further, at the cost of a cross-repo
artifact handoff and reporting. The live evidence above covers the branch design; no cross-repo
experiment was needed to establish that boundary.

The reporting tradeoff remains: a dispatch on the worker branch has a native, nested `CI` run, but
app copies on the tested source commit are flat. If GitHub enables and enforces `cache-mode` for uv,
the existing jobs could instead declare their cache authority directly and retain the current native
reporting. Neither design changes the separate question of build scripts executing in the trusted
producer.

References:
[GitHub's proposed explicit cache mode](https://github.com/orgs/community/discussions/194493),
[runner support](https://github.com/actions/runner/pull/4538), and
[existing branch restrictions](https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching#restrictions-for-accessing-a-cache).
