sequenceDiagram
    autonumber
    participant K as Kaspi API
    participant C as Celery Worker
    participant DB as PostgreSQL
    participant UI as TWA / ERP Frontend
    participant T as Telegram

    rect rgb(240, 248, 255)
    Note right of K: 1. Ingestion & Бронирование (Epic 21)
    C->>K: GET /v2/orders (state=KASPI_DELIVERY, 6h window)
    K-->>C: JSON Orders & Entries
    C->>DB: Валидация SKU (Strict Mapping)
    C->>DB: BEGIN SAVEPOINT (на каждый заказ)
    C->>DB: UPSERT Customer
    C->>DB: UPDATE inventory_balances (reserved_quantity += N)
    C->>DB: COMMIT (Если нет ошибок маппинга)
    end

    rect rgb(255, 245, 238)
    Note right of K: 2. Передача в доставку (Epic 22)
    UI->>DB: POST /ready-to-ship (Менеджер/Склад)
    DB->>K: POST /v2/orders (status=ASSEMBLE)
    K-->>DB: 200 OK (Генерация накладной запущена)
    
    loop Polling накладной (Max 4 retries)
        DB->>K: GET /v2/orders/{code}
        K-->>DB: URL накладной (kaspiDelivery.waybill)
    end
    
    DB->>K: Download PDF(waybill_URL)
    DB->>DB: Кэширование PDF
    DB->>T: Отправка объединенного PDF в топик склада
    end

    rect rgb(240, 255, 240)
    Note right of K: 3. Физическая отгрузка и Ledger (Epic 22)
    UI->>DB: POST /ship (Склад подтвердил отгрузку)
    DB->>DB: SELECT FOR UPDATE inventory_balances
    DB->>DB: Снятие брони: reserved_quantity -= N
    DB->>DB: Списание: quantity -= N
    DB->>DB: INSERT inventory_transactions (Ledger по location_id)
    DB->>DB: UPDATE SalesOrder (status=shipped)
    end
