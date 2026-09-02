# Actions cache boundary experiment

The earlier `workflow_run` trial denied a cache write after a pull request, but allowed one after a
push to `main`. The event name alone therefore did not establish the intended boundary.

These small, manually dispatched workflows test the explicit `cache-mode` setting and existing
branch scoping. They print only the non-secret cache access claims, look up a supplied marker key,
and optionally attempt to save a uniquely named file containing `cache-boundary`. The pinned cache
client predates `cache-mode`, so a denied save must be enforced by the service instead of merely
being skipped by the client.

The intended checks are:

1. A `write` run on `main` can create a marker.
2. A `read` run on `main` can find that marker but cannot create another.
3. A `write` run on a non-default branch can find the `main` marker and writes in its own scope.
4. `main` and a sibling branch cannot find the non-default branch's marker.

The cache API's recorded `ref` and the save/restore logs are the evidence. A green save step alone
is not evidence of a successful write: `actions/cache` reports permission denials as warnings.

References:
[GitHub's proposed explicit cache mode](https://github.com/orgs/community/discussions/194493),
[runner support](https://github.com/actions/runner/pull/4538), and
[existing branch restrictions](https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching#restrictions-for-accessing-a-cache).
