# Migration Approach: SQL Server to MariaDB

This section describes how the 44 in-scope tables move from the legacy SQL Server database to the new MariaDB design with minimal downtime, how the old and new systems run side by side, and how we prove the data matches.

## Assumptions

- The legacy database stays the system of record until each base is cut over.
- Other legacy modules (orders, technical records, labour, finance) still read inventory data during the transition. They cannot all change on day one.
- Changes in the legacy database can be captured continuously. This needs confirming with the database owner.
- There are 8 bases. Base is already a separation key in the data, so it is a natural unit for cut-over waves.
- Volumes are those in the assignment: about 1.2M parts, 6M stock records and 40M movements a year.
- A short planned freeze at cut-over (minutes, not hours) is acceptable, outside peak hangar activity.

## The central difficulty

A plain table copy will not work, because the new model is not a copy. `sStock` and `sOrderPartReceipt` become one stock item, classification flags move onto the part, life and certificate rows become JSON, and about 53 log tables disappear into a single ledger. So the migration has two steps:

1. Copy the legacy tables as they are into a temporary holding area.
2. Transform that copy into the new tables, repeatedly, until cut-over.

This keeps the risky logic in one place that can be rehearsed and tested, and it can be re-run safely at any time.

## Phases

| Phase | What happens | Exit check |
|---|---|---|
| 0. Profile | Scan the 44 tables for orphans, duplicates, missing values the new model requires, and bad dates. Produce a list of data fixes. | Fix list agreed with domain owners. |
| 1. Build | Create the new schema and the transform. Define the opening-balance and ledger seeding rules (below). | Transform runs end to end on a test copy. |
| 2. Rehearse | Run a full load and transform at least three times against fresh copies. Time each run. | Reconciliation passes with no unexplained differences. |
| 3. Parallel running | Keep the new tables current from legacy changes on a short schedule. Nobody writes to the new system yet. | Lag stays under the agreed limit for a full week. |
| 4. Shadow reads | A pilot group reads from the new API and search while still entering data in legacy. Differences are logged and reviewed. | Screens agree with legacy on sampled business checks. |
| 5. Cut-over by base | One base at a time, pilot base first. See below. | Reconciliation for that base passes and users sign off. |
| 6. Decommission | After all bases are live and a stable period has passed, make the legacy inventory tables read-only, archive and retire them. | Retention requirements met by the archive. |

## Running side by side

For the duration of the transition, any given base has exactly one system accepting writes.

- **Before a base's cut-over:** legacy takes writes. Changes flow into the new tables automatically.
- **After a base's cut-over:** the new system takes writes for that base, and legacy becomes read-only for that base's inventory data.
- **Old modules still reading inventory:** a small sync copies changes from the new system back to the legacy tables they read (quantity, location, status). It uses the same outbox pattern as the search sync, so there is one event stream with two consumers. This is the last piece to retire and is listed in the risks.
- **Routing:** requests are routed by base to the old or new path, so switching a base is a configuration change, not a deployment.

Cut-over steps for one base:

1. Announce a short freeze window.
2. Stop entry in legacy for the base and wait until the new tables have caught up.
3. Run the transform for the base and the full reconciliation checks.
4. If every check passes, switch the base to the new system and start the sync back to legacy.
5. Watch the base closely for the first day and keep the rollback path open for an agreed period.

Rollback: stop writes in the new system, replay the base's new changes into legacy, and switch routing back. This works while few changes have been made, which is why waves are small and the pilot base goes first.

## What happens to dropped history

- The old log, history and deleted tables are not carried over as live data. They are exported once to cheap archive storage and kept for the retention period (assumed 7 years, to be confirmed), so audits can still query them.
- The new movement ledger starts with one opening entry per stock item at cut-over, so quantities add up from day one. If the legacy log is reliable, recent history can also be loaded and marked as migrated. How much to load is an open question.
- Number ranges (`sOrderReceiptNo` and similar) are not copied row by row. Each new sequence starts above the highest number already used, checked at cut-over.
- Finance columns (`aJournal*`, `aTransactionID*`, `Amount*Cost`) are not moved into Inventory. Their values go to the archive and, if Finance wants, are sent once as opening cost events.

## Proving the data matches

Three levels of checks run on every rehearsal and every cut-over. Results are stored with a timestamp as evidence for sign-off.

### 1. Counts

| Check | Example |
|---|---|
| Row counts per legacy table against the copy | `sPart` rows equal copied rows. |
| Row counts per new table against expected | Stock items equal `sStock` rows, per base. |
| Rows excluded by rule | Every row not migrated is listed with a reason, and the numbers add up. |
| Child counts | Certificates in JSON equal rows in `sOrderPartReceiptCert`. |

### 2. Checksums

- For each table and base, compute a hash over the key business columns in a fixed order, after applying the same cleaning rules on both sides.
- Compare by key range so a mismatch narrows quickly to a small block of rows.
- For merged and JSON data, hash the expected result built from legacy against what is stored in the new tables.
- Compare totals of quantity and value by part, condition and ownership.

### 3. Sampled business reconciliations

These can be understood and agreed by someone from the business.

| Reconciliation | What it proves |
|---|---|
| On-hand quantity by part, base, condition and ownership | Stock is the same. |
| Serialised items appear once with quantity 1 | Traceability survived the merge. |
| Ledger total per stock item equals the stock quantity | Opening entries and later movements add up. |
| Items with shelf life expiring in the next 12 months | Dates and time zones were handled correctly. |
| Items with certificates: counts match, 100 spot-checked against records | Certificates were not lost in the JSON move. |
| Serviceable and unserviceable counts per location type | Location rules and condition mapping are right. |
| Open demands and their reserved quantities | Work in progress still reads correctly. |
| 200 random stock items per base, checked field by field by a stores supervisor | Real users agree it looks right. |
| Total stock value against Finance's own figure, within a tolerance | Costs were carried correctly. |

Any difference becomes a ticket with a cause: a source data error, a transform bug, or a rule change. Cut-over needs no open items without a signed explanation.

## Search index

- After the first complete load, build the search collections into a new collection and switch an alias to it, so users never see a half-built index.
- During parallel running, the same event stream keeps search current.
- Run the same 500 saved queries against legacy search and the new search and compare results, especially part numbers typed with dashes, spaces or different case.

## Risks and how we handle them

| Risk | Handling |
|---|---|
| Dirty legacy data breaks rules in the new model (orphans, duplicate serials) | Profile early, fix at source where possible, otherwise set aside and list. |
| Copy lag at peak times | Alert on lag, cut over outside peak, size it in rehearsal. |
| Old modules depend on legacy table shapes | The sync back keeps those tables current. Retire module by module. |
| Cut-over takes longer than the window | Timed in rehearsals. If checks fail, the base stays on legacy. |
| Guessed rules turn out wrong | Low-confidence rows from the disposition table are reviewed with domain owners before the pilot. |

## Open questions

1. Can continuous change capture be turned on in the legacy database?
2. How much movement history is worth loading into the ledger, beyond opening balances?
3. What is the retention requirement for archived logs?
4. Which other modules read inventory tables, and can any switch to the new API early?
5. Does Finance want an opening cost feed, or does it keep its own history?
