# Design Decisions & Trade-offs

[← Back to case study](../README.md)

Each decision is written as a short record: **context → options → decision → consequences.**

---

## D1 · Build a custom system instead of subscribing to an inventory SaaS

- **Context:** the business needed to track treatment usage (not just sales), sealed and
  opened packs, a multi-step approval and dispute process between warehouse and outlets, and
  purchasing from a marketplace that has no integrations.
- **Options:** an off-the-shelf inventory SaaS · a custom build.
- **Decision:** custom build (the business's decision, which I then designed and delivered).
- **Consequences:** ➕ fits the real process, including dispute evidence, pack/loose units
  and screenshot import. ➖ months of build time, ongoing maintenance, no ready-made
  accounting or POS integrations, and reliance on one engineer.

## D2 · Cloud Firestore instead of a relational database

- **Context:** a small team, no dedicated operations staff, and Firebase already used for
  sign-in and file storage.
- **Options:** PostgreSQL (managed) · Firestore.
- **Decision:** Firestore.
- **Consequences:** ➕ fully managed, transactions, no servers to patch, integrated with Auth
  and Storage. ➖ no joins, so display data is deliberately denormalised. Aggregates need
  explicit design (see D4). Collection naming and consistency depend on discipline rather
  than a schema.

## D3 · A stateless API on Cloud Run instead of an always-on server or Cloud Functions

- **Context:** traffic follows business hours, with low concurrency.
- **Options:** a VM or always-on container · Cloud Functions · Cloud Run.
- **Decision:** a containerised Express API on Cloud Run.
- **Consequences:** ➕ scales to zero outside business hours, is a plain Express app (no
  framework lock-in) and is easy to run locally. ➖ occasional cold starts, and in-memory
  caches are per instance (hence the optional Redis layer).

## D4 · Stock totals as a recalculated cache, not a counter

- **Context:** totals must be fast to read and must never silently drift.
- **Options:** increment and decrement a counter · compute on every read · recalculate a
  cache from batches.
- **Decision:** recalculate from batches after each change, with a full-rebuild command.
- **Consequences:** ➕ self-healing and auditable. ➖ more reads per write, and a short window
  where a total can lag if a process crashes after the main transaction commits.

## D5 · Usage logging that doesn't deduct stock immediately

- **Context:** outlets log daily usage, but miscounts and mistakes are common in a busy
  salon.
- **Options:** deduct batch stock on every usage entry · keep usage as a log and reconcile at
  stocktake.
- **Decision:** usage is a log. The stocktake baseline is the snapshot minus logged usage,
  and differences go through discrepancy reports.
- **Consequences:** ➕ batch quantities only change through controlled, explainable
  operations. ➖ live on-hand figures at outlets depend on usage being logged before the next
  count.

## D6 · Layered access control instead of more roles

- **Context:** six fixed roles worked at first. Later, management wanted to adjust which
  roles see which sections (for example, giving accounting access to an existing role)
  without a code release.
- **Options:** add more hard-coded roles · replace the role system · add a configurable layer
  on top.
- **Decision:** keep the fixed roles for route-level safety and add an admin-editable
  **role × module matrix**, plus location scoping for outlet users.
- **Consequences:** ➕ access changes without deployments, and no explosion of roles. ➖ three
  layers to reason about, so each is enforced on the server, and the UI only mirrors them.

## D7 · Custom JWT sessions alongside Firebase Auth, plus device approval

- **Context:** most staff sign in with email and password, some with Google, and shared
  devices are common at outlets.
- **Options:** Firebase Auth only · custom sessions only · both.
- **Decision:** the backend issues its own signed JWT with roles embedded; Firebase ID
  tokens are accepted as a fallback for Google sign-in; every new device must be approved by
  an admin.
- **Consequences:** ➕ no database lookup for roles on each request, and a stolen password
  alone isn't enough. ➖ two token types to maintain, and role changes take effect through a
  short-lived revocation cache rather than instantly.

## D8 · On-device OCR instead of a cloud API or LLM

- See [Engineering challenges §5](engineering-challenges.md#5-validating-ocr-output).
- ➕ zero running cost, private, and deterministic validation. ➖ lower raw accuracy than
  cloud OCR, which the arithmetic cross-checks and human review compensate for.

## D9 · Per-feature idempotency instead of a generic idempotency-key layer

- **Context:** retries and double clicks must not apply the same stock change twice.
- **Options:** a generic idempotency-key store for every write · guards built into each
  critical operation.
- **Decision:** guards per operation: status re-checks inside transactions, and
  existence checks keyed on a business reference.
- **Consequences:** ➕ cheap and fitted to each operation. ➖ new endpoints must remember to
  add their own guard. A generic layer is the natural upgrade as the system grows.
