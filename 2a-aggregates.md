# Inventory ERD: Aggregates, Child Entities and Lookups

Four diagrams, same structure as the slides. Paste any block into a Mermaid viewer (GitHub, VS Code preview, mermaid.live).

Key: PK primary key, FK foreign key, UK unique key. Lookups are the small code tables.

## 1. Overview

```mermaid
erDiagram
    part ||--o{ part_alternative : "has alternatives"
    part ||--o{ part_ata : "has ATA refs"
    part ||--o{ part_model : "fits models"
    part_classification ||--o{ part : classifies
    part_status ||--o{ part : "status of"
    part ||--o{ stock_item : "which part"
    location ||--o{ stock_item : "held at"
    location_category ||--o{ location : categorises
    stock_condition ||--o{ stock_item : ""
    stock_ownership ||--o{ stock_item : ""
    stock_status ||--o{ stock_item : ""
    stock_category ||--o{ stock_item : ""
    stock_item }o..o{ movement_ledger_entry : "by id, no FK"
    part }o..o{ audit_log : "by id, no FK"
    location }o..o{ audit_log : "by id, no FK"
    stock_item }o..o{ audit_log : "by id, no FK"
```

## 2. Part aggregate

```mermaid
erDiagram
    part_classification ||--o{ part : classifies
    part_status ||--o{ part : "status of"
    part ||--o{ part_alternative : "part_id"
    part ||--o{ part_alternative : "alternative_part_id"
    part ||--o{ part_ata : has
    part ||--o{ part_model : has
    part ||--o| part : "merged from / to"

    part_classification {
        smallint part_classification_id PK
        varchar code UK
        varchar description
        bool requires_shelf_life
        bool requires_certificate
        bool requires_manufacture_date
    }
    part_status {
        smallint part_status_id PK
        varchar code UK "ACTIVE, WAITING_APPROVAL, INACTIVE"
        varchar description
    }
    part {
        bigint part_id PK
        binary public_id UK "shown in the API"
        varchar part_number
        varchar part_number_key "built from part_number"
        smallint part_classification_id FK
        smallint part_status_id FK
        bigint merged_from_part_id FK
        bigint merged_to_part_id FK
        json life_rules
        json export_rules
        json accounting_rules
        int row_version "concurrency token"
    }
    part_alternative {
        bigint part_alternative_id PK
        bigint part_id FK
        bigint alternative_part_id FK
        varchar approval_ref
        bool is_two_way
    }
    part_ata {
        bigint part_ata_id PK
        bigint part_id FK
        varchar ata_ref
        bool is_default
    }
    part_model {
        bigint part_model_id PK
        bigint part_id FK
        varchar model_ref
        bool applies_to_all_serials
    }
```

Part is global: it has no base_id.

## 3. Location and stock item aggregates

```mermaid
erDiagram
    location_category ||--o{ location : categorises
    location ||--o{ stock_item : "held at"
    location ||--o{ stock_item : "previous location"
    part ||--o{ stock_item : "which part"
    stock_condition ||--o{ stock_item : ""
    stock_ownership ||--o{ stock_item : ""
    stock_status ||--o{ stock_item : ""
    stock_category ||--o{ stock_item : ""

    part {
        bigint part_id PK "see previous diagram"
    }
    location_category {
        smallint location_category_id PK
        varchar code UK
        varchar description
    }
    location {
        bigint location_id PK
        binary public_id UK
        bigint base_id "platform id, UK with the three codes"
        varchar plant_code
        varchar warehouse_code
        varchar location_code
        varchar location_path "built from the three"
        varchar barcode
        smallint location_category_id FK
        bool is_serviceable
        bool is_quarantine
        bool allows_put_away
        bool allows_allocation
        bool is_active
        json settings
        int row_version
    }
    stock_condition {
        smallint stock_condition_id PK
        varchar code UK
        bool is_serviceable
    }
    stock_ownership {
        smallint stock_ownership_id PK
        varchar code UK
    }
    stock_status {
        smallint stock_status_id PK
        varchar code UK
    }
    stock_category {
        smallint stock_category_id PK
        varchar code UK
    }
    stock_item {
        bigint stock_item_id PK
        binary public_id UK "shown in the API"
        bigint base_id "platform id"
        bigint part_id FK
        varchar part_number "copied from part"
        bigint location_id FK
        bigint previous_location_id FK
        smallint stock_condition_id FK
        smallint stock_ownership_id FK
        smallint stock_status_id FK
        smallint stock_category_id FK
        decimal quantity_on_hand
        varchar unit_code
        varchar serial_number "UK with part_id"
        varchar batch_number
        varchar barcode
        datetime received_at
        date shelf_life_expires_on
        decimal unit_cost_base
        json certificates
        json life_data
        json warranty_data
        int row_version
    }
```

Copied part fields are refreshed by events and can be up to 60 seconds old. base_id is an id from the platform; there is no base table in Inventory.

## 4. Ledger and audit log

```mermaid
erDiagram
    movement_ledger_entry {
        bigint movement_id PK
        datetime movement_at PK "table split by year"
        bigint base_id
        bigint stock_item_id "by id, no FK"
        bigint part_id "by id, no FK"
        bigint from_location_id "by id, no FK"
        bigint to_location_id "by id, no FK"
        decimal quantity "always above zero"
        decimal unit_cost_base
        varchar condition_code "code, not FK"
        varchar ownership_code "code, not FK"
        varchar source_type
        bigint source_id
        bigint reverses_movement_id "fixes a mistake"
        bigint actor_user_id
        varchar correlation_id
        varchar origin
        json details
    }
    audit_log {
        bigint audit_id PK
        varchar entity_type "by id, no FK"
        bigint entity_id
        varchar action
        bigint actor_user_id
        datetime changed_at
        json changes "before and after"
    }
```

Why no foreign keys: history must survive changes to stock items, inserts stay fast on a very large table, old years can be archived, triggers block updates and deletes, and a wrong entry is fixed by adding a reversing entry.
