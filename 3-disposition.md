# Legacy Table Disposition

This section records what happens to each important legacy table in the new Inventory design. Every table gets one of four decisions:

| Decision | Meaning |
|---|---|
| **Keep** | Stays as its own table in MariaDB, usually renamed and trimmed. |
| **Merge** | Its data moves into another table, either as columns or as a JSON field. |
| **Drop** | Leaves the operational schema because something else now does its job. Legacy rows are archived, not deleted. |
| **Externalise** | Belongs to another module. Inventory keeps only an ID and talks to the owner through an API or events. |

Confidence shows how sure I am without confirmation from a domain expert. Low-confidence rows are listed as questions at the end.

## How each decision was made

Each table was checked against the same six questions, in this order:

1. Is it a real business thing with its own lifecycle? Keep it as an aggregate root.
2. Is it a short list of choices, such as status, condition or category? Keep a small lookup and copy the code onto the row.
3. Is it a log, history or deleted copy? Drop it. The movement ledger and the audit trail replace it.
4. Does it belong to another module (prefixes `t`, `l`, `a`, `uRAL`, or finance postings)? Externalise it and keep only the ID.
5. Is it a number range? Replace it with a sequence.
6. Is it a one-to-many detail? Keep it as a child table if people search or filter by it, otherwise fold it into JSON.

## Core business tables

| # | Legacy table | Context | Decision | New home | Rationale | Confidence |
|---|---|---|---|---|---|---|
| 1 | `sPart` | Part Catalogue | Keep | `part` (root) | The catalogue entry for every part. It has its own lifecycle (draft, approved, inactive) and every other aggregate refers to it. | High |
| 2 | `sPartClassification` | Part Catalogue | Merge | Flags on `part`, code list kept as lookup | Flags such as shelf-life required and certificate required drive write rules and search, so they sit on the part row. The code list stays for reference. | Medium |
| 3 | `sPartAlternative` | Part Catalogue | Keep | `part_alternative` (child of Part) | One-to-many, and search needs it so a request for one part number can find its approved alternates. `tApprovalID` is externalised to Technical Records. | High |
| 4 | `sPartATA`, `sPartModel` | Part Catalogue | Keep | `part_ata`, `part_model` (children) | Used to filter parts by aircraft chapter and model, so they stay searchable child tables. | Medium |
| 5 | `sPartUnit` | Part Catalogue | Merge | Columns on `part` | Unit of issue is one value per part, so a separate table only adds a join. | Medium |
| 6 | `sStock` | Stock and Receipt | Merge | `stock_item` (root) | The physical item on a shelf. Combined with its receipt line so one row answers "what is this, where is it, and where did it come from". | High |
| 7 | `sOrderPartReceipt` | Stock and Receipt | Merge | `stock_item`, with life and warranty data in JSON | Serial, batch, condition and ownership become columns because screens filter by them. Detail-only fields go to JSON. Cost and journal columns are externalised (see row 25). Text copies such as `PartNo` and `Location` are dropped because they duplicate real references. | High |
| 8 | `sOrderPartReceiptCert` | Stock and Receipt | Merge | `certificates` JSON on `stock_item` | A receipt can carry several certificates. They are shown on the detail screen but rarely searched, so a JSON array with type, number, date and approval reference fits. | Medium |
| 9 | `sOrderPartReceiptLifeCode` | Stock and Receipt | Merge | `life_data` JSON on `stock_item` | Life data varies by part type. Fields that need searching, such as hours since new, are exposed as generated columns. | Medium |
| 10 | `sOrderPartReceiptDiscrepancy` | Stock and Receipt | Keep | `receipt_discrepancy` (child) | Discrepancies are raised, worked and closed, so they have their own small lifecycle. | Medium |
| 11 | `sPartWriteOff` | Stock and Receipt | Merge | `movement_ledger_entry` (type `WRITE_OFF` or `WRITE_ON`) | A write-off is a stock adjustment, not a document. The misspelt `WriitenOffDate` maps to the ledger timestamp. | High |
| 12 | `sBaseWarehouseLocation` | Warehouse and Location | Keep | `location` (root) | The bin or shelf, with rules such as serviceable stock only. It is the target of every move. | High |
| 13 | `sBaseWarehouse`, `sBasePlant`, `sBaseLocation` | Warehouse and Location | Merge | Copied IDs and names on `location` | Short, stable hierarchy. Copying base, plant and warehouse onto each location removes three joins from every stock search. | Medium |
| 14 | `sDemand` | Demand and Issue | Keep | `demand` (root) | A mechanic's request for a job. Has its own status and owns its lines. | High |
| 15 | `sDemandPart` | Demand and Issue | Keep | `demand_line` (child) | Keeps request and reservation data. Issue and return detail moves to the children below, and finance columns are externalised. | Medium |
| 16 | `sPartIssue`, `sCoreReturn`, `sOweBack` | Demand and Issue | Keep | Children of `demand` | Each records a business event against a demand line. The quantity change itself goes to the ledger. | Medium |
| 17 | `sPickingList`, `sPickingListItem` | Demand and Issue | Keep | `picking_list`, `picking_list_item` | Stores staff work from picking lists, so they need to exist as documents with their own status. | Medium |
| 18 | `sDespatch`, `sDespatchPart` | Despatch and Transfer | Keep | `despatch`, `despatch_line` | Shipping documents with addresses, air waybill, flight and export licence. Stock rules don't belong here, and this data doesn't belong in stock. | High |
| 19 | `sPartDespatch`, `sStockBaseTransfer`, `sIBT` | Despatch and Transfer | Merge | `movement_ledger_entry` (types `DESPATCH`, `TRANSFER_OUT`, `TRANSFER_IN`) | These record quantity leaving one place and arriving at another, which is exactly what the ledger holds. Transfer cost columns are externalised. | Medium |
| 20 | `sStockCheck`, `sStockCheckPart` | Stock Control | Keep | `stock_check` (root) and `stock_check_line` | A count has its own lifecycle: planned, counting, reviewed, posted. Posting writes adjustments to the ledger. | High |
| 21 | `sMinMax` | Stock Control | Keep | `min_max_policy` | A small planning rule per part and location. Kept apart so changing a threshold never locks stock rows. | Medium |

## Lookup and workflow status tables

| # | Legacy table | Decision | New home | Rationale | Confidence |
|---|---|---|---|---|---|
| 22 | `sPartStatus`, `sPartCondition`, `sPartClass`, `sPartUnitName` | Keep as lookups | `ref_*` tables, with codes copied onto rows | Short lists that rarely change. Each row stores the code, so reads need no join. Part status is a single code, and any active or inactive flags are derived from it so they can't contradict each other. | High |
| 23 | `sStockStatus`, `sStockOwnership`, `sStockCategory`, `sStockLocationCategory`, `sOrderPartReceiptStatus` | Keep as lookups | `ref_*` tables | Same reasoning. Condition and ownership codes are also copied into the ledger so history reads correctly even if a lookup later changes. | High |
| 24 | `sDemandItemStatus` | Merge | Status column with allowed transitions in code | This is a workflow state, not reference data. Holding the allowed transitions in the Demand aggregate stops invalid jumps, such as issued straight back to requested. | Medium |

## Cross-cutting families

| # | Legacy family | Decision | New home | Rationale | Confidence |
|---|---|---|---|---|---|
| 25 | Finance posting columns (`aJournalID*`, `aTransactionID*`, `Amount*Cost`, `AmountBaseTransferCost`, `AmountRCTransferCost`) | Externalise | Finance module, fed by `StockReceived`, `StockIssued`, `StockTransferred` and `StockWrittenOff` events | Finance owns journals. Inventory publishes the cost in an event and keeps at most the journal reference Finance sends back. | High |
| 26 | The 53 Log, History and Deleted tables (for example `sStockLog`, `sBaseWarehouseLocationHistory`, `*Deleted`) | Drop | `movement_ledger_entry` for quantity and location changes, `audit_event` for edits to records | One append-only ledger replaces per-table copies. Legacy rows move to an archive store for compliance and investigation, and the ledger is seeded with opening balances at cut-over. | High |
| 27 | Number-range tables (`sOrderReceiptNo` range columns, `*Range` tables) | Drop | MariaDB sequences, one per document type and base | Range tables need row locks to hand out numbers and become a hotspot. A sequence gives the same unique numbers without blocking. The receipt header data in `sOrderReceiptNo` is kept as a small child of the receipt. | Medium |
| 28 | Nine audit and locking columns on every table | Merge | Standard system columns: `created_at`, `created_by`, `updated_at`, `updated_by`, `version` | One rule replaces nine columns. `version` is the optimistic concurrency token and `GUID` becomes the public identifier in APIs. Full change history lives in `audit_event`. | Medium |
| 29 | User and other-module references (`uRALUserIDCreated`, `t*`, `l*` IDs) | Externalise | ID columns only, with names copied where a screen needs them | Platform, Technical Records and Labour own these records. Inventory never copies their tables. | High |
| 30 | `sStock.sBaseWarehouseLocationIDLast` | Drop | Derived from the ledger | "Previous location" is history, and the ledger already holds it. Storing it on the row creates a second, possibly wrong, answer. | Medium |
| 31 | `BarCode` and `sStockBarCode` | Merge | One `barcode` column on `stock_item`, unique per base | Two places for the same value is a known source of mismatches. If both are filled and differ, migration flags the row for review. | Low |

## Summary

| Decision | Count |
|---|---|
| Keep | 14 rows (including lookups) |
| Merge | 12 rows |
| Drop | 3 rows (logs and history, number ranges, last-location copy) |
| Externalise | 2 rows (finance postings, other-module references) |

The design ends up with about 25 operational tables in place of the 44 in-scope tables plus 53 log copies.

## Questions for domain owners

1. Is an inter-base transfer one transaction, or does the receiving base confirm arrival separately? This decides whether `sIBT` becomes one ledger entry or two.
2. Does any screen search certificates by number? If so, certificates move from JSON to a child table.
3. If `BarCode` and `sStockBarCode` disagree, which one do people scan?
4. How long must the archived log and history data stay queryable, and by whom?
5. Is `GUID` already exposed to partners? If so, it must stay the public identifier.
