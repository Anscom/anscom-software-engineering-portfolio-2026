# Technical Q&A: The Hard Questions

[← Back to case study](../README.md)

These are the questions a senior engineer would ask about this system, with my answers.
Code snippets are **simplified illustrations written for this page**, not the production
source.

**Jump to:** [Data modelling](#data-modelling) · [Consistency & concurrency](#consistency--concurrency) ·
[Failure scenarios](#failure-scenarios) · [Security](#security) · [Performance & scale](#performance--scale) ·
[OCR import](#ocr-import) · [Judgement](#judgement)

---

## Data modelling

### Q1. How do you represent "how much stock is at Outlet A"?
There are three layers, each with one job:

| Layer | Holds | Role |
|---|---|---|
| **Batch** | variant, location, expiry, sealed qty, loose qty, reserved qty | Source of truth |
| **Stock level** (per location × variant) | on hand, reserved, available, near-expiry, expired | Derived cache for fast reads |
| **Reservation** | batch, request line, qty, status | Explains *why* stock is reserved |

`available = max(0, onHand − reserved)`. Nothing writes the stock level as an independent
fact; it's always a summary of batches (see Q5).

### Q2. Why track sealed packs and loose units separately?
Because the business consumes stock in a different unit from the one it buys in. A box of 10
ampoules is bought and transferred as a box but used one ampoule at a time in treatments. A
single integer can't say "3 sealed boxes and 4 loose ampoules", and that difference matters
at stocktake (a sealed box can still be transferred; an opened one can't).

```ts
// Simplified: take `units` from a batch, opening a sealed pack only when needed
function take(batch: { sealed: number; loose: number }, units: number, perPack: number) {
  let { sealed, loose } = batch;
  while (loose < units && sealed > 0) { sealed -= 1; loose += perPack; } // open a pack
  if (loose < units) throw new Error('Insufficient stock');
  loose -= units;
  return { sealed, loose };
}
```

### Q3. Firestore has no joins. How do you avoid N+1 reads and still show readable data?
- **Batched reads** (`getAll(...refs)`) fetch all related documents for a page in one round
  trip.
- **Write-time denormalisation where it's stable.** Batch details are copied onto transfers,
  and the shipment number onto its POs, so outlets can display them without permission to
  read warehouse documents.
- **Cached reference data** (brands, categories, suppliers) is joined in memory instead of
  being re-read.

The rule I follow is to denormalise data that rarely changes and is read often, and to
reference data that changes often.

---

## Consistency & concurrency

### Q4. Two warehouse staff approve requests that need the same last 5 units. What happens?
Both requests run as Firestore transactions, and Firestore transactions are optimistic. The
first transaction to commit wins. The second one's reads are now stale, so Firestore retries
it; on retry it sees `reserved + qty > onHand` and fails with a conflict. Stock can never be
reserved twice.

```ts
await db.runTransaction(async (tx) => {
  const req = await tx.get(requestRef);
  if (req.get('status') !== 'APPROVED_FOR_WAREHOUSE') throw conflict('Already processed');

  const batches = await Promise.all(plan.map((p) => tx.get(batchRef(p.batchId)))); // all reads first
  plan.forEach((p, i) => {
    const b = batches[i].data()!;
    if (b.reservedQty + p.qty > b.onHandQty) throw conflict('Not enough stock');
  });

  plan.forEach((p) => {                                   // then all writes
    tx.update(batchRef(p.batchId), { reservedQty: FieldValue.increment(p.qty) });
    tx.set(reservationRef(), { ...p, requestId: req.id, status: 'ACTIVE' });
  });
  tx.update(requestRef, { status: 'WAREHOUSE_APPROVED' });
});
```

The key design choice is that **reservation helpers take the caller's transaction** instead
of opening their own. That makes "reserve stock" and "approve request" one atomic unit; if
either fails, neither happens.

### Q5. Why recalculate stock totals instead of incrementing a counter?
Increments are cheaper, but if one is ever missed (a bug, a crash, a manual data fix), the
counter is wrong **forever**, and nothing in the system can detect it. Recalculating from
batches means:
- the total heals the next time that item changes;
- a single command can rebuild every total from scratch;
- when someone disputes a number, the explanation is in the batch records.

```ts
async function recompute(locationId: string, variantId: string) {
  const snap = await batches.where('locationId', '==', locationId)
                            .where('variantId', '==', variantId).get();
  const t = snap.docs.reduce((acc, d) => {
    const b = d.data();
    acc.onHand += b.onHandQty;
    acc.reserved += b.reservedQty;
    if (isExpired(b)) acc.expired += b.onHandQty;
    else if (isNearExpiry(b)) acc.nearExpiry += b.onHandQty;
    return acc;
  }, { onHand: 0, reserved: 0, nearExpiry: 0, expired: 0 });
  await stockLevelRef(locationId, variantId)
    .set({ ...t, available: Math.max(0, t.onHand - t.reserved) }, { merge: true });
}
```

The deliberate exception: approving a request adjusts `reserved` directly inside its own
transaction, to avoid re-reading every batch on each approval.

### Q6. An outlet confirms a delivery in three separate steps. How do you prevent crediting the same units twice?
Each confirmation computes what has **already been credited** for that line from earlier
completed movements, and credits only the difference, capped at what was dispatched. Any
units not yet confirmed stay in a remaining in-transit movement.

```ts
const already = creditedSoFar(lineId);                        // from previous confirmations
const toCredit = Math.min(confirmedNow, dispatched - already); // never above what was sent
if (toCredit <= 0) return;                                     // repeated confirmation = no-op
```

### Q7. How do you stop a double click from receiving the same shipment twice?
The status is checked twice: once before the transaction (a cheap early exit) and again
**inside** it. If two requests race, the first commit sets the shipment to `DELIVERED`; the
second transaction re-reads it, sees `DELIVERED` and aborts. No batches are duplicated.

### Q8. Do you have idempotency keys?
Not as a generic layer. Each critical operation has its own guard:
- a status re-check inside the transaction (receiving, approving);
- an existence check keyed on a business reference (a discrepancy-report adjustment is
  applied only if no movement already exists for that report ID).

This was a cost-benefit call for a small team. I'd add generic idempotency keys (a
client-generated key plus a stored response) as more writers and integrations appear.

---

## Failure scenarios

### Q9. The process crashes right after a shipment-receiving transaction commits, before stock totals are refreshed. What state is the system in?
- **Correct:** the POs, the new batches and the shipment status, because they committed
  atomically together.
- **Briefly stale:** the derived stock-level cache for the affected items.
- **Recovery:** the next change to those items recalculates them, and the rebuild command
  fixes everything at once.

Nothing is lost or double-counted; one cache may be out of date for a short time. The
improvement I'd make is a scheduled reconciliation job (or an outbox task written in the same
transaction) so that recovery doesn't depend on the next change.

### Q10. A discrepancy report's stock adjustment commits, but marking the report "resolved" fails. Is it safe to retry?
Yes. Before applying an adjustment, the service checks whether a movement already exists for
that report. A retry skips the stock change and only completes the status update.

### Q11. What happens to stock if an outlet never confirms a delivery?
It stays **in transit**: removed from the warehouse but not credited to the outlet, and
visible as pending on both sides. Today it's resolved manually (confirm or cancel the
transfer). An automatic time-out was rejected on purpose: silently moving stock back would
hide exactly the kind of loss the system exists to expose. The planned improvement is an
alert when stock stays in transit longer than a set time.

---

## Security

### Q12. Walk through what happens on an API request.
`security headers → CORS allow-list → rate limit → verify JWT (roles inside the token) →
device approved? → role allowed for this route? → module enabled for this role? → validate
body (Zod) → service logic + location scope check → transaction`.

Every layer runs on the server. The frontend only mirrors the rules to hide menus.

### Q13. Why embed roles in the JWT? Doesn't that delay revocation?
Embedding roles removed a database read from **every** request. Revocation is handled
separately: each user has a "tokens valid after" timestamp that is checked against a cache
lasting 60 seconds. The trade-off is explicit: a revoked user can keep access for up to
about a minute, in exchange for one fewer database read per call. For an internal tool, I
judged that acceptable.

### Q14. Why device approval on top of passwords?
Outlet devices are shared and staff turnover is real. With device approval, a leaked password
alone isn't enough; the device must also have been approved by an admin. Unapproved devices
get a clear "waiting for approval" screen instead of a confusing error.

### Q15. How do outlet staff only see their own outlet?
Through a location scope resolved per user: the outlets they manage, or the one they are
assigned to. It's checked in the service layer on every location-specific read and write, and
cached for 15 minutes; the cache is cleared immediately when a user's role or assignment
changes. Cost prices are removed from responses for outlet roles.

---

## Performance & scale

### Q16. How did you find the slowdowns?
By tracing slow pages back to the queries behind them instead of guessing. Each cause had a
distinct signature:
- list pages whose read count grew with the number of rows (N+1 reads);
- the same reference collections being read on every request;
- loops issuing one query per outlet.

Each was fixed with a targeted change (batched reads, caches cleared on every write, roles in
the token, `in` queries with composite indexes, virtualised lists) rather than a rewrite.
Measured in database work: a 25-item product page went from **100 reads to 4**, cross-outlet
request lists from **up to 20 queries to 2**, and every API call lost one role lookup.

### Q17. What breaks first at 100× the load?
In order:
1. **Sequence counters.** PO, shipment and batch numbers come from single counter documents,
   and Firestore sustains about one write per second per document. Fix: pre-allocate number
   ranges or use sharded counters.
2. **Hot batches.** Many simultaneous requests for the same popular item contend on the same
   documents and retry. Fix: allocate across more batches, or queue reservations per item.
3. **Recalculating totals.** For items with very many batches, this becomes expensive. Fix:
   archive depleted batches, or apply deltas plus a periodic full reconciliation.
4. **Per-instance caches.** With many Cloud Run instances, the in-memory cache misses more
   often. The shared Redis layer already exists for this.

### Q18. How would you support 100 separate businesses on this system?
Today it serves a single business. Supporting many would mean adding a tenant ID to every
document, enforcing it in every query, scoping caches and counters by tenant, and adding
tenant-aware indexes. It's a deliberate redesign, not a configuration change. I'd do it only
with a real need, and test isolation explicitly.

---

## OCR import

### Q19. Why not use a cloud OCR API or an LLM?
- **Cost:** on-device OCR costs nothing per screenshot.
- **Privacy:** order and payment screenshots never leave the purchaser's device.
- **Verifiability:** an LLM might return a plausible but wrong number. Arithmetic
  cross-checks either pass or don't. I would rather have a deterministic check plus a human
  review than a confident guess.

### Q20. How do you catch OCR mistakes?
Every amount that can be derived from others is recomputed and compared:

```ts
const expected = round2(unitPrice * qty - discount + gst);
if (Math.abs(expected - paid) > 0.01) {
  // If the bill total agrees with `expected`, OCR most likely misread `paid`
  warn(`Order ${orderNo}: OCR read ${paid}, maths says ${expected}`);
}
```

A second check runs at the bill level: the sum of order amounts plus the transaction fee must
equal the bill total. When exactly one side disagrees, the likely misread value is
suggested, and the purchaser confirms it before any PO is created.

---

## Judgement

### Q21. What's the weakest part of the system, honestly?
- **Test coverage.** Money and stocktake calculations are unit-tested, but the multi-step
  workflows are verified manually.
- **Delivery process.** There's no CI and one production environment.
- **Module size.** The largest workflow module has grown too big.

They're all known and prioritised in [lessons learned](lessons-learned.md). I chose to
deliver the core operational loop first for a business that depended on it daily.

### Q22. What are you most proud of?
That the system answers questions people used to argue about. "Who lost the stock?" became
a lookup: packed versus dispatched versus received, with a name and a timestamp on each step.
The technical work (transactions, derived totals, idempotent adjustments) exists to make that
answer trustworthy.
