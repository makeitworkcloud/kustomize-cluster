# Bootstrap guidance

This directory establishes App-of-Apps wiring and cluster bootstrap configuration. Changes here can affect Argo CD access, OIDC/RBAC, CI identity, secret availability, and application ordering.

The active Applications reconcile `bootstrap/secrets`, `operators`, and `workloads/apps`; other bootstrap resources require a separately reviewed bootstrap/provisioning path. Preserve sync waves, hook semantics, and wait-job dependencies. Do not change Argo CD configuration or bootstrap identity resources without explicit confirmation.
