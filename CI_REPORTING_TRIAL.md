# CI reporting trial

This PR starts the real uv builds, followed by the shared `CI` workflow and the
`Zaniebot CI Demo` app's per-job reports. The `test-reporting / failure and rerun`
fixture deliberately fails once so its Details link and native job tree can be
inspected. The integration and smoke tests run alongside it.

The workflow proposal is [PR #2](https://github.com/zaniebot/uv-workflow-run-demo/pull/2).
