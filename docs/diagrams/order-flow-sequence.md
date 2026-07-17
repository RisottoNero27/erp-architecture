# Sequence-диаграмма: сквозной поток заказа (Kaspi → отгрузка)

Показывает разделение ответственности: внешние вызовы делает только backend (FastAPI) и Celery-воркеры; PostgreSQL — только хранение и транзакционная целостность.

```mermaid
sequenceDiagram
    autonumber
    participant K as Kaspi API
    participant C as Celery Worker
    participant A as Backend (FastAPI)
    participant DB as PostgreSQL
    participant UI as TWA (склад)
    participant T as Telegram

    Note over K, DB: 1. Ingestion и бронирование (Epic 21)
    C->>K: GET /v2/orders (state=KASPI_DELIVERY)
    K-->>C: JSON Orders & Entries
    C->>DB: Валидация SKU + UPSERT (идемпотентно)
    C->>DB: Бронь (reserved_quantity += N)

    Note over A, T: 2. Передача в доставку (Epic 22)
    UI->>A: POST /ready-to-ship
    A->>DB: Фиксация готовности
    A->>K: POST /v2/orders (status=ASSEMBLE)
    K-->>A: 200 OK (запуск генерации накладной)

    loop Поллинг накладной
        A->>K: GET /v2/orders/{code}
        K-->>A: URL накладной
    end

    A->>K: Скачивание PDF
    A->>T: Отправка PDF в топик склада

    Note over A, UI: 3. Физическая отгрузка (Epic 22)
    UI->>A: POST /ship (склад подтвердил)
    A->>DB: Транзакция: снятие брони + списание quantity (SELECT FOR UPDATE)
    A->>DB: Запись в ledger (inventory_transactions, append-only)
    A->>DB: UPDATE статус = shipped
```
