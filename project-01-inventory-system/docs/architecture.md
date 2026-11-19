# Architecture

[← Back to case study](../README.md)

## 1. Components

| Component | Technology | Responsibility |
|---|---|---|
| Web app | React 18 SPA on Firebase Hosting | All user interfaces: desktop for warehouse and admin, mobile for outlets |
| API | Node.js + Express on Google Cloud Run | Authentication, authorisation, validation, business rules, transactions |
| Database | Cloud Firestore | System of record for all business data |
| File storage | Cloud Storage | Product images, receipt files |
| Identity | Firebase Authentication | Google sign-in; token verification |
| Cache | In-memory + Redis | Reference data (brands, categories, suppliers…), user scope lookups |

The browser talks to Firestore **only through the API**, so every rule is enforced in one
place on the server. The Firebase SDK in the browser is used only for sign-in and file
uploads.

## 2. Backend structure

```
src/
├── app.ts               middleware chain + route mounting
├── middleware/          auth · roles · module gate · validation · rate limit · request ID · errors
├── modules/<feature>/   one folder per business area
│   ├── *.routes.ts      URL + permissions + validation schema
│   ├── *.controller.ts  HTTP in/out
│   ├── *.service.ts     business rules + database transactions
│   └── *.types.ts/dto   types and Zod schemas
└── common/              shared errors, caching, pagination, location scoping
```

Feature modules include products and catalogue, batches, stock levels, reservations,
movements, purchase orders, stock requests, usage logging, stocktake, discrepancy reports,
notifications, reorder levels, users, roles and access, and device approval.

**Error handling:** one error type (`code`, `message`, `HTTP status`, `details`) and one
central handler that returns a consistent JSON shape with a request ID. Unexpected errors are
logged with full detail but returned to the client as a generic message.

## 3. Request pipeline

```mermaid
sequenceDiagram
  participant U as Browser
  participant M as Middleware
  participant S as Service
  participant DB as Firestore

  U->>M: HTTPS request (Bearer JWT, device ID)
  M->>M: security headers, CORS, rate limit, request ID
  M->>M: verify JWT (roles inside token, no DB lookup)
  M->>M: device approved? (cached)
  M->>M: role allowed for route? module enabled for role?
  M->>M: validate body (Zod)
  M->>S: call service
  S->>S: business rules + location scope check
  S->>DB: transaction: read → validate → write
  DB-->>S: commit
  S->>DB: refresh derived stock totals
  S-->>U: JSON response
```

## 4. Data model (simplified)

```mermaid
erDiagram
  PRODUCT ||--|{ VARIANT : has
  SUPPLIER }o--o{ PRODUCT : supplies
  VARIANT ||--o{ BATCH : "stocked as"
  LOCATION ||--o{ BATCH : holds
  VARIANT ||--o{ STOCK_LEVEL : "summarised in"
  LOCATION ||--o{ STOCK_LEVEL : "summarised at"
  SUPPLIER ||--o{ PURCHASE_ORDER : receives
  PURCHASE_ORDER }o--o| SHIPMENT : "grouped into"
  SHIPMENT ||--o{ BATCH : "creates on receipt"
  STOCK_REQUEST ||--o{ RESERVATION : reserves
  BATCH ||--o{ RESERVATION : "reserved from"
  STOCK_REQUEST ||--o{ MOVEMENT : "dispatched as"
  STOCKTAKE ||--|{ COUNT_LINE : contains
  COUNT_LINE ||--o{ DISCREPANCY_REPORT : "explained by"
  DISCREPANCY_REPORT ||--o{ MOVEMENT : "adjusts via"
  USER }o--o| LOCATION : "assigned to"
```

**Modelling principles**

- **Batch = source of truth.** A batch holds the variant, location, supplier, expiry date,
  sealed quantity, loose quantity and reserved quantity.
- **Stock level = derived cache.** One document per (location, variant) holds on hand,
  reserved, available, near-expiry and expired. It is always rebuildable from batches.
- **Deliberate denormalisation.** Firestore has no joins, so display data that is needed
  together is copied at write time, for example batch details on a transfer, or the shipment
  number on its purchase orders.
- **Counters inside transactions.** PO numbers, shipment numbers and per-day batch numbers
  are allocated atomically.

## 5. Key workflows

### Stock request lifecycle

```mermaid
stateDiagram-v2
  [*] --> PendingManagerApproval: outlet staff request
  PendingManagerApproval --> ApprovedForWarehouse: outlet manager approves
  PendingManagerApproval --> Rejected
  ApprovedForWarehouse --> WarehouseApproved: warehouse approves<br/>reserve batches (same transaction)
  ApprovedForWarehouse --> Rejected
  WarehouseApproved --> Transferring: dispatch<br/>debit warehouse · stock in transit
  WarehouseApproved --> Cancelled: release reservations
  Transferring --> Partial: outlet confirms, some lines short
  Transferring --> Fulfilled: everything confirmed
  Transferring --> Cancelled: cancel transfer
  Partial --> Fulfilled: shortfall resolved / re-dispatched
  Fulfilled --> [*]
```

### Purchase order → stock

```mermaid
flowchart TD
  A[Create PO<br/>manual · Excel · screenshots · catalogue] --> B[PO ordered]
  B --> C[Group POs into a shipment]
  C --> D{Receive shipment}
  D -->|one transaction| E[Update every linked PO]
  D -->|one transaction| F[Create batches<br/>auto batch number + QR label]
  D -->|one transaction| G[Mark shipment delivered]
  E & F & G --> H[Refresh stock totals]
  H --> I[Stock available for outlet requests]
  G --> J[Accounting: payment mode, receipts,<br/>refunds → Excel export]
```

### Stocktake → discrepancy resolution

```mermaid
flowchart LR
  A[Start count<br/>snapshot system qty] --> B[Count sealed + loose]
  B --> C{Variance?}
  C -->|no| E[Line OK]
  C -->|yes| D[Discrepancy report<br/>damaged · missing · extra · expired · return]
  D --> F{All variances<br/>covered?}
  E --> F
  F -->|no| B
  F -->|yes| G[Confirm count]
  D --> H[Resolve report → stock adjustment<br/>applied once, idempotently]
```

## 6. Deployment

```mermaid
flowchart LR
  Dev[Developer machine] -->|vite build + firebase deploy| Hosting[Firebase Hosting]
  Dev -->|gcloud run deploy --source| Build[Cloud Build → container image]
  Build --> Run[Cloud Run service]
  Dev -->|rules + indexes| FS[Firestore]
```

- One production environment plus local development. Configuration comes from environment
  variables, and secrets are never committed.
- Logs are structured JSON written to stdout and collected by Cloud Logging. Every request
  has a request ID that also appears in error responses.
