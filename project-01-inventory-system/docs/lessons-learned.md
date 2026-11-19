# Lessons Learned

[← Back to case study](../README.md)

## What worked

- **Modelling the real process, not a generic one.** The dispute problem was solved by
  recording each handover separately. That came from watching how stock actually moved
  between people, not from a feature checklist.
- **One source of truth with derived caches.** Recalculating totals from batches has saved
  more debugging time than any other decision. When something looked wrong, the answer was
  always in the batch records.
- **Transactions everywhere stock changes.** Concurrency problems never reached production
  as lost updates.
- **Shipping in small increments.** Continuous delivery over seven months (500+ commits)
  let the business adopt the system section by section and gave fast feedback on each part.
- **Scripted data changes.** Every import, migration and production data repair was a
  reviewed, re-runnable script, never a manual database edit.

## What I'd do differently

- **Automated tests earlier.** Unit tests cover the money and stocktake calculations, but the
  multi-step workflows are verified by hand. Integration tests against the Firestore emulator
  would make larger refactors safer.
- **CI/CD from day one.** Deploys are manual CLI commands. A pipeline with type-checking,
  tests and a preview environment would cost little and catch mistakes before production.
- **A staging environment.** There is currently one production environment, so testing
  against realistic data needs extra care.
- **Smaller modules.** The stock-request service and its pages grew to over 10,000 lines
  each. Splitting by workflow step (approve, dispatch, receive, disputes) would make them
  easier to understand and test.
- **A shared collection-name registry and a single movement log.** Constants were spread
  across modules, which let small naming inconsistencies creep in.
- **A dedicated audit-event log.** Provenance (who, when, why) is recorded on each document,
  but a single append-only event stream would be easier to query and harder to alter.

## What's next

1. A CI pipeline with type-checking, unit tests and emulator-based integration tests.
2. Further security hardening: more defence-in-depth layers and two-factor sign-in.
3. A scheduled reconciliation job that rebuilds any stale stock totals automatically.
4. Generic idempotency keys for write endpoints.
5. Point-in-time recovery and scheduled backups.
6. Splitting the largest modules by workflow step.
