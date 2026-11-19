# Engineering Challenges

[← Back to case study](../README.md)

Each section covers the problem, how I solved it, and the alternatives I considered.

---

## 1. Ending warehouse ↔ outlet disputes

**Problem.** Stock moved from the warehouse to outlets with no reliable record. When an
outlet reported fewer items than the warehouse claimed to have sent, the gap couldn't be
traced: was it short when packed, lost in transit, or miscounted on arrival? Each dispute
became an argument about who should bear the cost.

**Approach: record every handover separately.**

| Step | Recorded | Who owns the stock |
|---|---|---|
| Pack | Quantity packed per line, with a reason for any shortage (low stock, packing count, substitute…). Damage found before dispatch is recorded here. | Warehouse |
| Dispatch | Quantity sent. Stock is removed from the warehouse and placed **in transit**. A printed picklist and confirmation travel with it. | Neither (in transit) |
| Receive | The outlet confirms the quantity that actually arrived, line by line. | Outlet |
| Gap | Dispatched > received → a shortfall with a reason (missing or damaged) and a discrepancy report. The warehouse can re-dispatch the missing quantity as a linked follow-up. | Under investigation |

Two implementation details matter:
- **Partial confirmations never over-credit.** The quantity already credited is tracked per
  line, so an outlet confirming a delivery in several steps only adds the newly confirmed
  units.
- **Re-processing is safe.** Shortfall reports are checked by request reference before they
  are created, so a retry doesn't open a second report.

**Result.** A discrepancy now points to a specific step and a specific person. The system
doesn't decide who pays; it gives management the evidence to decide.

**Alternatives considered.**
- *One "transfer" action that moves stock in a single step* (what most off-the-shelf tools
  do). It's simpler, but it removes the evidence that settles disputes.
- *Credit the outlet at dispatch and correct it later.* This hides losses in transit inside
  the outlet's numbers.

---

## 2. Concurrent stock changes

**Problem.** Warehouse staff receiving goods, a manager approving a request and someone
resolving a discrepancy can all touch the same batch at the same moment. Firestore has no
row locks, so a naive read → modify → write loses updates.

**Approach.**
- **Every multi-document change runs in a transaction** (about 40 call sites, 21 of them in
  the stock-request workflow alone). Firestore transactions are optimistic: if any document
  read inside the transaction changes before commit, the whole transaction retries.
- **Reservation helpers take the caller's transaction** instead of opening their own. The
  reservation record, the batch's reserved quantity, the stock total and the request status
  therefore commit as **one** atomic unit.
- **Guard conditions are re-checked inside the transaction.** For example, receiving a
  shipment re-reads its status inside the transaction and aborts if it is already
  delivered, so two simultaneous clicks can't receive goods twice.
- **A reservation can never exceed the stock on hand**, enforced inside the same transaction.
- **When retrying could change the meaning, the client is told instead.** If staff requests
  change while a manager is compiling them, the API returns a conflict and asks the user to
  retry rather than silently merging.

**Alternatives considered.**
- *Lock flags or lock documents* make contention explicit, but they need expiry and deadlock
  handling and serialise work that rarely conflicts at this scale.
- *A queue per batch* removes contention entirely, but adds infrastructure and latency for a
  problem optimistic transactions already solve.

---

## 3. Stock totals that cannot drift

**Problem.** "How much is available at Outlet A?" has to be one fast read, but the true
answer is spread across many batch records. A counter that is only ever incremented and
decremented eventually drifts after a bug or a crash, and there's no way back.

**Approach.**
- **Batches are the source of truth.** The per-location stock total is a **derived cache**.
- After any batch change, the total for that (location, variant) is **recalculated from all
  its batches** rather than adjusted by a delta. Wrong data therefore corrects itself the next
  time the item changes.
- A **full rebuild** command regenerates every total from scratch whenever needed.
- The one exception is deliberate: reservation changes adjust reserved and available
  quantities inside the reservation's own transaction, which avoids re-reading every batch on
  each approval.

**Pack and loose units.** Many items are bought as packs but used one unit at a time (a box
of ampoules, a pack of masks). Batches hold *sealed packs* and *loose units* separately.
Taking more than the loose quantity opens a pack automatically, and enough loose units roll
back up into a pack. All of that logic lives in one well-tested utility so every module
behaves the same way.

**Alternatives considered.**
- *Pure increments* are cheaper per write, but cannot self-heal.
- *Compute everything when it's read* is simple, but turns every stock lookup into a
  multi-document query on the busiest path.

---

## 4. Performance under growing data

**Problem.** As real data accumulated, key pages slowed down noticeably. Profiling showed
there wasn't one bug; there were several patterns that don't scale.

| Root cause | Fix | Effect |
|---|---|---|
| N+1 reads: list pages fetched related records once per product | Batched `getAll()` reads | Several reads per product → one batched read per page |
| Reference data (brands, categories, suppliers…) fully re-read on every request | In-memory cache with expiry, plus a shared Redis layer, cleared on every write | Repeated full-collection reads removed from hot paths |
| User roles looked up in the database on every request | Roles embedded in the signed JWT; short-lived cache for revocation and device checks | One fewer database read per API call |
| Per-outlet loops issuing up to 20 queries | A single `in` query with a new composite index | ~20 queries → 1 |
| Very long tables rendered in full | Virtualised scrolling | Only visible rows are rendered |
| Rate limiter was a no-op placeholder | Real rate limiting, with stricter limits on login routes | Abuse protection |

**Principle.** Make one targeted fix per root cause rather than a rewrite. Caches are
**invalidated on write**, so the expiry time is only a safety net and never the reason data
is correct.

---

## 5. Validating OCR output

**Problem.** Purchasers buy supplies from a Chinese e-commerce app and used to retype every
order into a purchase order. OCR can read the order screenshots, but it regularly drops or
misreads characters ("×2" read as "*?", "78.59" read as "78.50"). One wrong digit in a price
breaks the accounts.

**Approach.**
- **On-device OCR** (tesseract.js with Simplified Chinese and English models). There's no
  per-call API cost, screenshots never leave the device, and the roughly 15 MB model loads
  only the first time the feature is used, so it doesn't slow down normal page loads.
- **A pure parser, separate from OCR**, extracts shop, item, variant, order number, date,
  quantity, unit price, discount, GST and amount paid. Because it never touches images, it can
  be unit-tested.
- **Arithmetic cross-checks:**
  - `unit price × qty − discount + GST = amount paid` (per order)
  - `Σ amounts paid + transaction fee = bill total` (per payment)
  - A mismatch is flagged, and when one side agrees, the likely misread value is suggested.
- **The human makes the final call.** Drafts are shown for review before any PO is created.
- Orders are grouped by shop, and each shop becomes the supplier on its PO.

**Alternatives considered.**
- *Cloud OCR or vision APIs* are more accurate, but cost money per image and send purchase
  data to a third party.
- *LLM extraction* is flexible, but non-deterministic and harder to validate. Arithmetic
  checks give a guarantee an LLM can't.
- *Manual entry* is what the purchasers were trying to escape.
