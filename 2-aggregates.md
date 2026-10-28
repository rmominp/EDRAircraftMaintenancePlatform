# Task 1 – Aggregates, Roots, Children and Invariants

This section describes the aggregates inside the six Inventory contexts. For each one it gives the root, the children, what it references but does not own, and the rules it must always protect.

## Overview

| Aggregate | Context | Root | Children | Legacy source |
|---|---|---|---|---|
| Part | Part Catalogue | Part | Alternates, ATA links, model links | `sPart` and related |
| Stock Item | Stock and Receipt | Stock Item | Receipt, certificates, life data, discrepancies | `sStock`, `sOrderPartReceipt` and children |
| Movement Ledger | Stock and Receipt | Ledger Entry | Finance and document references | `sStockLog`, `sPartWriteOff`, `sPartDespatch` |
| Location | Warehouse and Location | Location | None (warehouse and plant are lookups) | `sBaseWarehouseLocation`, `sBaseLocation` |
| Demand | Demand and Issue | Demand Item | Issues, core returns, picking items | `sDemandPart`, `sPartIssue`, `sCoreReturn` |
| Stock Check | Stock Control | Stock Check | Stock Check Lines | `sStockCheck`, `sStockCheckPart` |

Smaller aggregates: Demand Request (header, `sDemand`), Picking List (`sPickingList`), Despatch (`sDespatch`, `sDespatchPart`), Min/Max Policy (`sMinMax`).

---

## 1. Part

**What it is.** The catalogue entry for a part number: what it is called, how it is classified, what units it uses, and which rules apply to stock of this part.

**Root.** Part.

**Children.**
- Part Alternative: another part that may be used instead, with the approving authority and QA number.
- Part ATA link: the ATA chapter(s) the part belongs to, with one marked as default.
- Part Model link: the aircraft models the part applies to.

**References (lookups, not owned).** Classification, status, class, condition, unit name. The row holds the lookup ID and also a copy of the code and the few flags that screens filter on.

**Copied values.** Classification code and its flags (shelf-life required, certificate required, manufacture date required), status code, unit code. They are read-only on the part. Only the lookup owner changes them, and a refresh updates the copies.

**Rules the aggregate protects.**
1. The part number is unique after normalisation (dashes, spaces and case ignored).
2. A part has exactly one status at a time. The status code is the source of truth, and any active/inactive flags are derived from it.
3. A part cannot be an alternate of itself. An alternate must have an approval status, an authority and a QA number before it can be used.
4. A two-way alternate is stored consistently in both directions or flagged once, never half-stored.
5. At most one ATA link is the default.
6. If the classification requires shelf life, the shelf-life months on the part must be greater than zero.
7. A merged part points to exactly one surviving part. The merge chain has no loops, and a merged part cannot receive new stock.
8. The unit of measure must exist and be valid.

**Events.** `PartCreated`, `PartStatusChanged`, `PartRulesChanged` (so Stock can use the latest rules).

---

## 2. Stock Item

**What it is.** One physical item or batch of stock at a place, received on a receipt, with its serial or batch number, certificate and life data.

**Root.** Stock Item.

**Children.**
- Receipt: the receipt line that brought it in (supplier batch, received quantity, inspection, cost).
- Certificates: one or more release certificates (for example EASA Form 1 or FAA 8130-3), each with type, number, date and approval reference.
- Life data: time since new, overhaul and installation per life code, with "known/unknown" flags.
- Receipt discrepancies: problems noted at receipt, with type, priority and status.

**References.** Part (by ID), Location (by ID), ownership, condition, status and category lookups.

**Copied values.** Part number and description, location path, base, condition code and whether it is serviceable. These support search and the display of the item. Part and Location changes refresh them.

**Rules the aggregate protects.**
1. The part must exist and be active before new stock is received against it.
2. An item with a serial number has quantity exactly 1. An item without one can hold any quantity.
3. Quantity on hand is never negative.
4. Quantity changes only through a movement. Nothing edits quantity directly.
5. A serial number is unique within a part *(assumed)*.
6. An item sits in exactly one location, and its base comes from that location. The base can change only through a transfer.
7. Unserviceable stock cannot sit in a location marked serviceable.
8. Stock can be put away only into an active location that allows put-away.
9. Stock marked do-not-allocate cannot be reserved or issued.
10. Customer-owned stock cannot be used to satisfy company demand unless the ownership allows it.
11. Every item traces to a receipt. If the part requires a certificate, the receipt must hold at least one. If life is unknown, the item must say so and give a reason.
12. Expired shelf-life stock cannot be issued *(assumed)*.
13. Two users changing the same item at once is detected by a row version. The second change is refused.

**Events.** `StockReceived`, `StockMoved`, `StockConditionChanged`, `StockStatusChanged`.

---

## 3. Movement Ledger

**What it is.** The single, permanent record of everything that changes stock. It replaces the per-table log, history and deleted copies.

**Root.** Ledger Entry.

**Children.** Finance references (journal, account, posting type) and document references (despatch, transfer, receipt number). Both are optional.

**References.** Stock Item, Part, From and To location, Demand Item, Stock Check Line, Despatch.

**Movement types.** Receive, Issue, Return, Transfer, Write-off, Write-on, Despatch, Adjustment, Stock-check posting.

**Rules the aggregate protects.**
1. Entries are append-only. No update and no delete. A mistake is fixed with a reversing entry.
2. Every entry has a source and a destination. When the real source or destination is outside the warehouse (supplier, scrap, customer), a named endpoint stands in for it.
3. Quantity is positive. The movement type decides the direction.
4. Every entry names its stock item and traces back to the receipt, the certificate, the serial or batch, and the life data.
5. Applying an entry must not make the stock item's quantity negative.
6. Quantity on hand equals the sum of the item's movements. A daily check proves this.
7. Each entry records who did it, when, and why (reason code).
8. Only the Stock and Receipt context writes entries. Other contexts ask it to, using commands.

---

## 4. Location

**What it is.** A place stock can be kept, such as a bin or shelf, and the rules for that place.

**Root.** Location.

**Children.** None. Changes to a location's structure are recorded in the audit trail. Stock moving between locations is recorded in the ledger.

**References.** Base (platform, by ID), warehouse and plant (lookups), location category (lookup).

**Copied values.** Warehouse name, plant name, location path, category code. The quarantine flag is inherited from the warehouse.

**Rules the aggregate protects.**
1. A location belongs to exactly one base, warehouse and plant.
2. The location code is unique within its warehouse *(assumed)*.
3. Behaviour flags cannot contradict each other. A location cannot be both serviceable and unserviceable, and cannot be both allocatable and do-not-allocate.
4. A location with stock on hand cannot be deactivated.
5. Stock cannot be put away into an inactive location or one that does not allow put-away.
6. A quarantine warehouse's locations accept only stock the quarantine rules allow.
7. A location that is part of an open stock check cannot issue stock *(assumed, based on the stock check link on the legacy location)*.

**Events.** `LocationCreated`, `LocationRulesChanged`, `LocationDeactivated`.

---

## 5. Demand

**What it is.** One request for a quantity of a part, from request through reservation, picking, issue and any return.

**Root.** Demand Item. The header (Demand Request) holds the demand number, type, priority, base and work scope, and is a separate small aggregate.

**Children.**
- Issue: each issue of stock against this item, with quantity, cost and who picked, issued and accepted it.
- Core return: a returned core or removed part linked to this item.
- Picking item: a line on a picking list that fulfils this item.

**References.** Demand Request, Part, original part, Stock Item (chosen stock), Location, order or task (external), Picking List.

**Rules the aggregate protects.**
1. Quantity requested is greater than zero.
2. Status moves only along the allowed path: requested, authorised, back-ordered or on order, reserved, picked, issued, accepted, completed. Cancelled is allowed any time before issue.
3. The legacy set of "status" booleans collapses into one state. Dates and actors for each step (authorised, reserved, picked, issued, accepted) are kept as a history list, and their order must be consistent.
4. Reserved stock must be serviceable, allocatable, in an allowed location, and owned in a way that fits the demand.
5. One stock item cannot be reserved for more than its available quantity, across all demands.
6. Quantity issued never exceeds quantity requested *(assumed)*, and quantity returned never exceeds quantity issued.
7. Issuing creates a ledger movement through the Stock context. The demand does not change stock itself.
8. A core return links to its parent demand item and to the receipt it came from.
9. Financial postings (WIP, cost of sales, transfers) are not part of this aggregate. Inventory sends cost events and Finance posts.

**Events.** `DemandRequested`, `StockReserved`, `PartPicked`, `PartIssued`, `PartReturned`, `DemandCancelled`.

---

## 6. Stock Check

**What it is.** A counting exercise at a base: what the system said was there, what was counted, and the difference.

**Root.** Stock Check (number, base, type, status, planned, opened and closed dates, who created and who posted it).

**Children.** Stock Check Line: part, location, condition, ownership, original quantity, counted quantity, difference, who counted and when, who posted and when.

**References.** Base (external), Stock Item, Part, Location.

**Rules the aggregate protects.**
1. A check moves through open, in progress, posted and closed, in that order.
2. Lines can be added or changed only while the check is open or in progress.
3. Counted quantity is zero or more. Difference is always counted minus original and is never typed in by hand.
4. Each line is unique for its check, stock item, location, condition and ownership.
5. A line's location belongs to the same base as the check.
6. Posting a line asks the Stock context to adjust quantity through a ledger movement. The adjustment and the line's posted flag succeed or fail together *(decision: per line, not per check)*.
7. A posted or closed check cannot be edited. A correction creates a new, audited adjustment.
8. Reopening a closed check is allowed only for users with the right permission, and is recorded.

**Events.** `StockCheckOpened`, `StockCountRecorded`, `StockCheckPosted`, `StockCheckClosed`.

---

## 7. Smaller aggregates

| Aggregate | Root | Key rules |
|---|---|---|
| Demand Request | Demand Request | Demand number is unique and comes from a number generator, not a range table. It holds the header only; items belong to Demand. |
| Picking List | Picking List with its items | Items refer to real stock and demand items. A picked list cannot be edited. A cancelled list records who and why. |
| Despatch | Despatch with its parts | Owns the shipping document only. Despatching asks Stock to move quantity. Export licence and dangerous goods fields are required where the part says so. |
| Min/Max Policy | Min/Max Policy | One policy per part, ownership and base. Minimum is at most the reorder level, which is at most the maximum. A locked policy cannot be changed automatically. |

## 8. Rules between aggregates

1. One transaction changes one aggregate. Anything wider is handled by commands and events.
2. Aggregates refer to each other by ID only.
3. Copied values are read-only on the copying aggregate. The owning aggregate publishes an event, and a handler refreshes the copies.
4. Any movement of stock goes through the Stock Item aggregate and ends as a ledger entry.
5. Anything owned by another module (finance, orders, technical records, labour, platform) is held as an external ID.

## 9. Open questions

- Is a serial number unique per part, or per part and base?
- Can stock be issued beyond the requested quantity (over-issue)?
- Does an open stock check lock the locations it covers?
- Should expired stock be blocked from issue, or only flagged?
