# MariaDB Design Notes: Part, Stock Item, Location, Movement Ledger

The full DDL is in `inventory-schema.sql`. It creates 15 tables: seven small code lists, Part with three children, Location, Stock Item, the Movement Ledger, and the audit trail. This note explains the choices so you can talk through them.

## The decisions the assignment asks for

| Decision | What I chose | What I rejected, and why |
|---|---|---|
| Primary key | `BIGINT UNSIGNED` auto-increment on every table | A UUID (`BINARY(16)`) as the primary key. Every secondary index and foreign key repeats the primary key, so a 16-byte key makes all of them bigger. Random UUIDs also scatter inserts across the index. A time-ordered UUID reduces that, but not the size. |
| Public identifier | `public_id BINARY(16)`, a time-ordered UUID, used only in URLs and events | Exposing the BIGINT. It is guessable and tells outsiders how many rows exist. The UUID is unique-indexed and never used in joins. |
| Base separation | `base_id NOT NULL` on stock, location and ledger, always the first column of the indexes screens use. Part is global. | A separate table or schema per base. With 8 bases and shared parts, it multiplies migrations and breaks cross-base search and transfers. |
| Concurrency | `row_version` on each root. Updates say `WHERE id = ? AND row_version = ?` and add 1. Zero rows updated means someone else changed it, and the API returns a conflict. | Row locks held while a user edits. Hangar tablets disconnect, so locks would be held for hours. |
| Lookups | Small lookup table, plus the id and a copy of the code on the row | Joining to lookups on every read, or storing only the code with no table. The copy answers screens without a join, and the table keeps the list controlled. |
| JSON versus columns | Columns for anything a screen filters, sorts or enforces. JSON for detail shown only on a screen. | Putting everything in JSON (cannot be indexed well) or everything in columns (hundreds of columns that are mostly empty). |

## Part

- **Global.** One row per part number for the whole company. Bases reach it through stock.
- **Search key.** `part_number_key` is a generated column that strips everything except letters and digits and upper-cases. `AB-12345`, `ab 12345` and `AB12345` all become `AB12345`. It has a unique index, so the same part cannot be entered twice with different punctuation. If the legacy data already has such duplicates, the migration will list them.
- **Status.** Only `status_code` is stored. `is_active` is derived from it, so the two cannot disagree.
- **Copied flags.** Shelf-life, certificate and manufacture-date requirements are copied from the classification onto the part, so the rule checks at receipt need no join. When a classification changes, an event updates the parts that use it.
- **JSON.** `life_rules`, `export_rules` and `accounting_rules` hold per-part configuration that is not searched or shared. If rules turn out to be shared between many parts, they move into their own table.
- **Children.** `part_alternative`, `part_ata` and `part_model` stay as tables because search needs them. The alternate table blocks a part being its own alternative and keeps the reverse lookup indexed.
- **Condition is not on Part.** I found no condition column on the legacy part table. Condition belongs to stock.

## Location

- **Four legacy tables become one.** Plant, warehouse, warehouse location and base location merge into `location`, with a generated `location_path`.
- **Behaviour is columns, not JSON,** because the rules need them: serviceable, unserviceable, put-away, allocation, line store, quarantine, query. A check constraint stops a location being both serviceable and unserviceable.
- **Unique** on base, plant, warehouse and location code.

## Stock Item

- **One row per physical item,** merging `sStock` and `sOrderPartReceipt`.
- **Current values and as-received values are kept apart.** The first draft mixed them. Current location, condition, ownership and quantity come from stock. Receipt date, received quantity and received condition are stored separately for traceability.
- **`previous_location_id`** replaces `sBaseWarehouseLocationIDLast`. The name now says what it is.
- **Cost.** Two costs: one in the order currency and one in the base currency. This is my reading of the legacy columns and needs confirming with Finance.
- **Copies from Part** (number, search key, description) are there so the stock screen and the search index need no join. They go stale when a part changes, and the staleness policy in the Solr section says how they are refreshed.
- **JSON** for certificates, life data and warranty, each as an array so several can exist. Two generated columns, `has_certificate` and `life_tsn_hours`, let screens filter on the most common questions without parsing JSON.
- **Rules in the database:** quantity cannot go negative, and a serialised item has a quantity of 0 or 1. The serial number is unique per part. That uniqueness is an assumption, and it would be wrong if serial numbers are reused after scrapping.
- **Indexes** all start with `base_id` and follow the screens: location view, part availability by condition and ownership, search by part key, batch, barcode, expiring shelf life.

## Movement Ledger

- **One table instead of the 53 log tables.** Every quantity, location, condition or ownership change is a row: receive, issue, return, transfer, despatch, write-off, write-on, stock-check adjustment, reclassify.
- **Quantity is always positive.** The movement type gives the direction. This avoids sign mistakes.
- **Append-only.** Triggers block updates and deletes. The application account only has insert and select rights. A mistake is fixed by a new entry that points at the one it reverses.
- **Context.** `source_type` and `source_id` point at the demand, despatch, stock check or transfer that caused it. `correlation_id` groups the entries of a single action, such as both sides of a transfer.
- **Origin.** `LIVE`, `OPENING` or `MIGRATED`, so migrated history can be told apart.
- **Partitioned by year** so old years can be archived by dropping a partition. MariaDB cannot combine partitioning with foreign keys, so the ledger has none. A nightly check confirms every ledger row points at a real stock item and that the ledger total matches stock on hand.
- **Not a duplicate of audit.** The ledger records stock movement. The audit table records who changed master data.

## Unified audit trail

One `audit_log` table replaces the per-table log, history and deleted copies of master data. Each row holds the entity, action, user and a JSON of old and new values. Deleted-record copies are no longer needed because the last values are in the audit row.

## Standard columns

Each root table has `created_at`, `created_by_user_id`, `updated_at`, `updated_by_user_id` and `row_version`. This replaces the nine legacy audit and locking columns. `Version` became `row_version`, and `GUID` became `public_id`.

## Changes from the earlier draft

- Fixed the problems found in review: current and as-received values mixed, base taken from the plant, a location flag copied onto stock, and one currency for two costs.
- Part status is a single code, with `is_active` derived.
- Dropped the `query` flag from Stock Item. It is a location flag.
- Boolean names follow one pattern (`is_`, `allows_`), instead of a mix.
- Dates and times are UTC and stored to the microsecond.
- Codes for tables owned by other modules (ECCN, dangerous goods, ATA, model) are stored as plain references, not copied code lists.

## Assumptions and open questions

1. Dimensions are in millimetres and weights in kilograms. The workbook gives no units.
2. `ShelfLife` is an expiry date.
3. Serial numbers are unique per part for the whole company.
4. The two legacy cost columns are order-currency and base-currency.
5. Two legacy barcode columns can be merged into one. Which one people scan is unknown.
6. Several legacy parts may normalise to the same search key. The migration profile needs to check.
7. The DDL targets MariaDB 10.6 or later. I have not run it against a server here, so run it on yours and tell me about any errors.
