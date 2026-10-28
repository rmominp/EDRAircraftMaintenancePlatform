# Field Naming Standard (draft for approval)

One rule set for every layer, so a field reads the same in MariaDB, Solr, the API and the Angular code.

## 1. Case by layer, and the conversion rule

| Layer | Style | Example |
|---|---|---|
| MariaDB tables and columns | lower snake_case, singular table names | `stock_item.part_number` |
| Solr fields | identical to the database column | `part_number` |
| JSON (API) | lowerCamelCase | `partNumber` |
| TypeScript properties | lowerCamelCase, same as JSON | `partNumber` |
| TypeScript types | PascalCase | `StockItem` |

Conversion rule: split on `_`, lower-case the first word, capitalise the first letter of the rest, join. `stock_item_id` becomes `stockItemId`. The reverse is the opposite. It is mechanical, so a generator handles it and nobody types it by hand.
Abbreviations count as one word (`eccnCode`, `unNumber`, `ataRef`), so the rule never produces `eCCNCode`.

## 2. Tables, keys and references

- **Table:** singular noun for the business thing: `part`, `stock_item`, `location`, `movement_ledger_entry`. Child tables start with the parent: `part_alternative`.
- **Primary key:** `<table>_id`, BIGINT: `part_id`.
- **Public identifier:** `public_id`, a UUID. It is the only id exposed outside the system.
- **Foreign key inside Inventory:** the parent's key name, with a role prefix when needed: `part_id`, `previous_location_id`, `alternative_part_id`.
- **Reference to another module:** `<thing>_ref`, a text or number reference, with no foreign key: `approval_ref`, `ata_ref`, `model_ref`. The suffix tells a developer "this lives in another module". `base_id` is the one exception because every table uses it, and it is documented as the platform's base id.
- **Code from a lookup:** `<thing>_code` (`condition_code`). If the lookup table is also referenced, the id is `<thing>_id`.

## 3. Types and units in names

| Concept | Rule | Example |
|---|---|---|
| Boolean | Start with `is_`, `has_` or `allows_`. Always positive. | `is_serviceable`, `has_certificate`, `allows_allocation` |
| Date (no time) | End with `_on` | `shelf_life_expires_on`, `manufacture_date` becomes `manufactured_on` |
| Timestamp | End with `_at`, stored in UTC, DATETIME(6) | `received_at`, `movement_at` |
| Quantity | `quantity_<meaning>` with a unit code beside it | `quantity_on_hand` with `unit_code` |
| Money | `<meaning>_amount` or `unit_cost`, with a currency code beside it. State which currency in the name when two exist. | `unit_cost_base`, `unit_cost_order` with `order_currency_code` |
| Measurements | Unit suffix | `length_mm`, `weight_kg` |
| Free text | `description`, `remarks`. No other synonyms. | |
| Counts | `<thing>_count` | |

## 4. One term per concept

| Concept | Use | Do not use |
|---|---|---|
| Barcode | `barcode` | `BarCode`, `sStockBarCode` |
| Location | `location` | `bin`, `shelf`, `warehouse location` |
| Remarks | `remarks` | `comments`, `notes`, `StockRemarks` |
| Quantity | `quantity` | `qty` (as a full word) |
| Number | `number` | `no`, `num` |
| Previous | `previous` | `last` |
| Created and updated | `created_*`, `updated_*` | `modified`, `changed` |
| Receipt | `receipt` | `receive` as a noun |

Allowed abbreviations (industry or database terms): `id`, `ref`, `uuid`, `ata`, `eccn`, `oem`, `pma`, `nsn`, `un`, `mm`, `kg`, `url`. Anything else is written in full.

## 5. Standard system columns

These replace the nine columns repeated on every legacy table. They are mapped by rule, not row by row.

| Legacy column | New column | Rule |
|---|---|---|
| `ID` | `<table>_id` | Primary key. |
| `GUID` | `public_id` | The public identifier. The decision is to use `GUID` here because it is a stable unique value. |
| `Version` | `row_version` | The concurrency token, an integer that goes up by 1 on each update. |
| `RecordTimeStampCreated` | `created_at` | UTC. |
| `RecordTimeStamp` | `updated_at` | UTC. |
| `uRALUser_IDCreated` | `created_by_user_id` | Reference to the platform user. |
| `uRALUser_ID` | `updated_by_user_id` | Reference to the platform user. |
| `Closed`, `ReadOnly`, `RecordLocked` | Dropped. | Closing is a status on the business record. Locks are replaced by `row_version`. Read-only becomes permissions. |

## 6. Solr names

Same as the database column. Types are chosen by the schema, not by name suffixes, so the Solr field `part_number` is the same text as the column. Derived search fields add a suffix: `_search` for analysed fields, `_key` for normalised exact keys: `part_number_key`, `part_number_search`.

## 7. Renames applied to the DDL (approved)

| Current name | Problem | Proposed name |
|---|---|---|
| `do_not_allocate` (stock and location) | Negative boolean, no prefix | `is_allocation_blocked` |
| `shelf_life_required`, `certificate_required`, `manufacture_date_required` | Boolean without a prefix | `requires_shelf_life`, `requires_certificate`, `requires_manufacture_date` |
| `export_controlled` | Same | `is_export_controlled` |
| `trigger_shelf_life_from_first_use` | Same | `starts_shelf_life_on_first_use` |
| `query_missing_stock` | Same | `queries_missing_stock` |
| `life_not_known` | Same | `is_life_unknown` |
| `serviceable` (stock_condition) | Same | `is_serviceable` |
| `is_serviceable_location`, `is_unserviceable_location` | Long and a pair | `is_serviceable`, `is_unserviceable` |
| `is_query` (location) | Unclear | `is_under_query` |
| `is_default` (part_ata) | Fine, kept | none |
| `all_serial_numbers` | Boolean without a prefix | `applies_to_all_serials` |
| `manufacture_date` | Date should end in `_on` | `manufactured_on` |
| `unit_cost`, `order_unit_cost` | Currency not in name | `unit_cost_base`, `unit_cost_order` |
| `unit_name` (part) | Ambiguous | `unit_long_name` |
| `plant_code`, `warehouse_code` | Fine, kept | none |
| `category_code`, `condition_code`, `status_code` on stock | Fine, kept | none |
| `legacy_receipt_id` | Fine, a migration reference | none |
| `quantity` (ledger) | Matches the rule | none |

These renames have been applied to the DDL, the design notes and the Solr file.

## 8. The nine legacy examples from the assignment

| Legacy | Decision |
|---|---|
| `uRALUser_IDCreated` | `created_by_user_id`. The platform prefix goes, and the name now says it points at a user. |
| `sStock.sBaseWarehouseLocation_IDLast` | `stock_item.previous_location_id`. The type prefix goes, and "Last" becomes "previous". |
| `sPartAlternative.tApprovalID` | `part_alternative.approval_ref`. Other tables write `tApproval_ID`, and both spellings are the same reference to Technical Records, now named `*_ref`. |
| `BarCode` and `sStockBarCode` | One column: `barcode`. Which of the two is scanned in practice is an open question. |
| `sPartWriteOff.WriitenOffDate` | Typo fixed. Timestamp or date is unknown, so it goes to the ledger as `movement_at`, the business time. Check with the business whether time of day is held. |
| `sStockBaseTransfer.Date` | Becomes the transfer's `movement_at` on the ledger entry, so the word "date" no longer stands alone. |
| `AmountBaseTransferCost`, `AmountRCTransferCost` | Split by currency into `unit_cost_base` and a reporting-currency amount in the ledger `details`, with `order_currency_code` or `reporting_currency_code` beside it. "RC" is read as reporting currency, which needs confirming. |
| `sOrderPartReceipt.PartNo` and `Location` | Dropped. They are text copies of `part_id` and `location_id`. Search copies in Solr (`part_number`, `location_path`) are derived, not stored again. |
| `Version` and `GUID` | `Version` becomes `row_version`, the concurrency token. `GUID` becomes `public_id`. |
