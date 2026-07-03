# OMS Project — X Technology

End-to-end Business Analysis project: a phased Order Management System for a company that runs **two parallel business lines** — B2B power-plant equipment bidding and a new B2C retail channel built to liquidate leftover bid inventory.

This repo documents the full BA process: problem discovery → process design → data modeling → system design → requirements documentation. It is not a finished product — it is a working record of how the requirements were built, including mistakes that were caught and corrected.

---

## Business Context

X Technology supplies equipment for hydro and thermal power plant projects. Bidding requires buying components in bundles, which leaves usable leftover stock that doesn't sell at retail on its own. Over time this created idle inventory tying up capital. Management decided to launch an online retail channel to move that stock — running independently alongside the existing bidding business, without disrupting it.

## Why This Project Has Phases

The first version of this project jumped straight from "manual process" to "fully automated multi-vendor e-commerce platform with vouchers, loyalty tiers, and microservices." A mentor review caught the gap: that jump skips the part every real transformation actually requires — standardizing the process, cleaning the data, training the people — *before* software gets built.

The project was restructured into phases to reflect that:

| Phase | Focus | Status |
|---|---|---|
| **G1–G2** | Process standardization (SOP) + data cleanup, led by Accounting | Documented |
| **Buffer** | Wait for business to hit growth threshold justifying investment | Documented |
| **G3** | OMS MVP — retail ordering, MoMo + bank transfer, inventory ledger | Documented (this repo's main scope) |
| **G4** | Future state — vouchers, loyalty, multi-seller, automated bank reconciliation | Idea only, not designed |

G3 is a **monolith** by design — not microservices. That choice was also a correction: an earlier version mixed monolith data modeling with microservice-style sequence diagrams, which doesn't hold together. Architecture decisions should follow the team's actual stage, not the most impressive-sounding pattern.


---

## What's in This Repo

```
/docs
  BRD_Phase3.docx              Business Requirements — written for the GĐ (Director)
  SRS_Phase3.docx              Software Requirements — written for Dev/QA
  BA_Document_v1.docx          As-Is / Gap / To-Be analysis (original scope)

/diagrams
  /bpmn
    BPMN_Transittion_phase_2.png                               Process standardization workflow
    BPMN_Transittion_phase_3.png                               Build approval workflow (BRD → SRS → Dev → UAT)
    BPMN_Work_flow_phase_2.png                                 Phase 2 operational workflow
    BPMN_Work_flow_phase_3.png                                 Phase 3 operational workflow
    BPMN_AS_IS_Phase_1.png                                     Original manual process
  /sequence
    Sequence-Order_Processing_phase_3.png                      Checkout →  Order creation
    Sequence-IPN_Validation_&_Processing_Phase_3&4.png         IPN signature validation & Duplicate/outdated/terminal-state detection
    Sequence-Payment_Phase_3&4.png                             Payment state update, request + polling
    Sequence-Trigger_Downstream_phase_3.png                    Success/failure downstream effects
    Sequence-Reconciliation.png                                Daily gateway reconciliation batch job
    Sequence-Cart_to_Master_Order_phase_4.png                  Checkout →  Master Order creation
    Sequence-Trigger_Downstream_Phase_4.png                    Success/failure downstream effects phase 4 with (voucher, loyalty, multi-vendor)
  /er            
    erd_phase4.png        Extended model (voucher, loyalty, multi-vendor) — G4 reference
    erd-phase3.png             Scoped-down model actually used for G3
```

---

## Phase 3 — System Design Summary

**Architecture:** Monolith, 5 modules (`ORD`, `PM`, `IVTR`, `CUS`, `ADM`), single shared database, internal function calls — no API Gateway between modules.

**Core design decisions and why they exist:**

- **IPN as source of truth.** MoMo's redirect callback is UI-only; it never updates payment status. Only the server-to-server IPN does, because the customer can close the browser before redirect fires.
- **Idempotency on every IPN.** MoMo can resend the same notification. Each one is checked against a `gateway_txn_ref` before any state change happens, so a retry never double-processes a payment.
- **Terminal state protection.** Once a payment is `confirmed` or `cancelled`, no later IPN — even a legitimate one that arrives out of order — is allowed to overwrite it. Timestamp comparison catches stale, delayed IPNs.
- **Two-phase locking on inventory.** Stock is reserved (not deducted) at checkout using `SELECT FOR UPDATE`, to prevent two customers from buying the last unit at the same moment. It's only deducted for real once payment is confirmed; otherwise it auto-releases after 15 minutes.
- **Inventory Ledger, not a stock counter.** Every stock movement (sale, manual adjustment, incoming bid-leftover stock, incoming retail purchase) is an append-only row. The balance is a `SUM()`, not a field that gets directly updated — so there's always an audit trail of *why* stock changed.
- **Snapshot pattern.** Order line items store the price at time of purchase, not a live reference to the product table, so historical orders stay accurate even after prices change.
- **Reconciliation buffer window.** The daily MoMo reconciliation job skips transactions from the last hour, so a late-arriving IPN doesn't get falsely flagged as a discrepancy before it's had time to land.

**What was deliberately left out of G3, and why:** shopping cart (direct order creation instead), vouchers, loyalty tiers, multi-seller splitting, automated bank reconciliation, refunds. These add real complexity that isn't justified until the retail channel has enough volume to need them — they're sketched in the v2 ERD as a forward reference, not designed in detail.

**On "API" in a monolith context.** API does not mean HTTPS. Internal module calls (`PaymentModule.initiate()`, `InventoryModule.reserve()`) are also APIs — they just cross function boundaries, not network boundaries. The only HTTPS interfaces in G3 are external: Frontend → Backend, MoMo IPN → Backend, Backend → MoMo query API. Everything else is an internal call within the same process.

---

## Related — Data Analytics Project

A separate, earlier project in `/sql` and `/dashboard`: analysis of 700,000+ job postings (Luke Barousse dataset) using CTEs, window functions, and self-joins, visualized in a 5-page Power BI dashboard. Included here because it's part of the same portfolio, not because it's part of the OMS system design.

---

## Status
