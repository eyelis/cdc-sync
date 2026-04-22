sequenceDiagram
    participant L_DB as Legacy DB (SQL 2012)
    participant S_W as Sync Worker
    participant N_API as New Service API
    participant N_DB as New DB

    Note over L_DB, N_DB: SENS 1 : LEGACY -> NEW (CDC)
    L_DB->>L_DB: Maj manuelle ou vieux Batch
    L_DB->>L_DB: Capture CDC (LSN)
    S_W->>L_DB: Poll CDC (fn_cdc_get_all_changes)
    L_DB-->>S_W: Changements (Table Subscription)
    S_W->>S_W: Filtrage Context_Info != 0x1234
    S_W->>S_W: Mapping vers JSON PIVOT
    S_W->>N_API: Appel POST /catalog (JSON Pivot)
    N_API->>N_DB: Validation + Audit + Insert

    Note over L_DB, N_DB: SENS 2 : NEW -> LEGACY (EVENTS / OUTBOX)
    N_API->>N_DB: Transaction: Save Catalog + Save Outbox
    N_DB-->>S_W: Event Trigger (ou Poll Outbox)
    S_W->>S_W: Mapping JSON PIVOT -> SQL Legacy
    S_W->>L_DB: SET CONTEXT_INFO 0x1234
    S_W->>L_DB: BATCH INSERT SUBSCRIPTION_VERSION
    Note right of L_DB: Création de N lignes (Versioning)
