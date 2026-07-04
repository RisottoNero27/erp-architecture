```mermaid
sequenceDiagram
    autonumber
    participant K as Kaspi API
    participant C as Celery Worker
    participant DB as PostgreSQL
    participant UI as TWA
    participant T as Telegram

    Note over K, DB: 1. Ingestion & Бронирование (Epic 21)
    C->>K: GET /v2/orders (state=KASPI_DELIVERY)
    K-->>C: JSON Orders & Entries
    C->>DB: Валидация SKU + UPSERT
    C->>DB: Бронь (reserved_quantity += N)

    Note over DB, T: 2. Передача в доставку (Epic 22)
    UI->>DB: POST /ready-to-ship
    DB->>K: POST /v2/orders (status=ASSEMBLE)
    K-->>DB: 200 OK (Запуск генерации)
    
    loop Поллинг накладной
        DB->>K: GET /v2/orders/{code}
        K-->>DB: URL накладной
    end
    
    DB->>K: Скачивание PDF
    DB->>T: Отправка PDF в топик склада

    Note over DB, UI: 3. Физическая отгрузка (Epic 22)
    UI->>DB: POST /ship (Склад подтвердил)
    DB->>DB: Снятие брони и списание quantity
    DB->>DB: Запись в Ledger (inventory_transactions)
    DB->>DB: UPDATE статус = shipped
