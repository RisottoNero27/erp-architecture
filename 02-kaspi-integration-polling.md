# Спецификация: 21 — Kaspi: Polling заказов
---

## Архитектурный контекст

```
  Kaspi API
  ┌─────────────────────────────────────────────────────────────┐
  │  GET /v2/orders                                             │
  │  filter: status=ACCEPTED_BY_MERCHANT  (новые заказы)            │
  │  filter: updatedDate ≥ now-30min  (отмены и изменения)      │
  └───────────────┬─────────────────────────────────────────────┘
                  │ каждые 3 мин (новые) / 5 мин (sync)
  ┌───────────────▼─────────────────────────────────────────────┐
  │  Celery Worker                                              │
  │                                                             │
  │  poll_new_orders()           sync_order_updates()           │
  │  ├─ upsert customers         ├─ найти наши заказы           │
  │  ├─ upsert sales_orders      ├─ сравнить статус Kaspi       │
  │  ├─ upsert kaspi_order_meta  └─ если CANCELLED → отменить   │
  │  ├─ upsert sales_order_lines     + освободить резерв        │
  │  ├─ резервировать остатки                                   │
  │  └─ trigger: upload_kaspi_pricelist.delay()                 │
  └───────────────┬─────────────────────────────────────────────┘
                  │
  ┌───────────────▼─────────────────────────────────────────────┐
  │  PostgreSQL                                                 │
  │  customers · sales_orders · kaspi_order_meta                │
  │  sales_order_lines · inventory_transactions                 │
  └─────────────────────────────────────────────────────────────┘
```

**Две задачи, не одна:** `poll_new_orders` работает часто и заточен на скорость — только `ACCEPTED_BY_MERCHANT`. `sync_order_updates` работает реже и ловит отмены через скользящее окно по `updatedDate`. Разделение позволяет настраивать частоту независимо.

> **Модель остатков (сверено с БД v1.14):** `inventory_balances` привязан к
> ЛОКАЦИИ (`location_id`), НЕ к складу напрямую. Локация знает склад
> (`locations.warehouse_id`). Остаток разрезан по состоянию (`condition_state_id`):
> `good` (норма, id=1) / `defective` (брак). UNIQUE на `(product_id, location_id,
> condition_state_id)`. Резерв бронирует ТОЛЬКО `good` и ищет баланс через
> JOIN на `locations` по складу заказа. Допущение: один товар = одна локация
> на складе (резерв не распределяет по нескольким локациям). `inventory_transactions`
> (ledger) тоже на уровне `location_id` — пишется при отгрузке, не при брони.

> **WAF Kaspi (проверено боем 22.05.2026):** запросы с дефолтным `User-Agent`
> инструмента (`httpx`, `curl`, `python-requests`) Kaspi молча дропает в таймаут —
> 0 байт ответа, без 4xx, соединение и TLS при этом проходят. Лечится честным
> `User-Agent` интеграции + `Accept: application/vnd.api+json` (JSON:API).
> Диагностический признак на будущее: если заказы перестали подтягиваться и в
> логах висят таймауты на исходящих к kaspi.kz — первым делом проверять заголовки,
> а не сеть/токен. Поэтому таймаут/пустой ответ в клиенте логируется отдельным
> классом ошибки (см. раздел 4 «Kaspi API клиент»).

> **Модель заказа Kaspi — state / status / флаги (сверено боем + докой 23.05.2026):**
> У Kaspi ДВА измерения: `status` (бизнес-исход: ACCEPTED_BY_MERCHANT/COMPLETED/CANCELLED)
> и `state` (стадия: NEW/SIGN_REQUIRED/PICKUP/DELIVERY/KASPI_DELIVERY/ARCHIVE).
> ВАЖНО: фильтрация по `status` У НАС НЕ РАБОТАЕТ (Kaspi игнорирует, отдаёт всё);
> фильтровать НАДО по `state`. У нас вся доставка Kaspi Delivery + автоприём →
> заказы минуют NEW, сразу `state=KASPI_DELIVERY` + `status=ACCEPTED_BY_MERCHANT`.
> Все рабочие заказы (упаковка, передача, предзаказы) висят в KASPI_DELIVERY.
> Внутренние стадии различаются ФЛАГАМИ, не state:
>   - `preOrder=true` → наш статус 'preorder' (без резерва, BR-21-05);
>   - `preOrder=false` → 'packing' (+ резерв);
>   - `assembled` (true=собран, накладная есть) → сохраняем в meta для Epic 22,
>     на статус в Epic 21 НЕ влияет. Заказ может прийти сразу assembled=true,
>     если собран между тиками поллинга — заводим как packing всё равно.
> Поллинг фильтрует state=KASPI_DELIVERY + узкое окно creationDate (~6 ч).

> **Окно дат обязательно (проверено боем):** `/v2/orders` отдаёт `400 Bad Request`
> на запрос БЕЗ диапазона по датам. И `get_new_orders` (creationDate), и
> `get_updated_orders` (updatedDate) ОБЯЗАНЫ слать `[$ge]`/`[$le]`. Для новых
> заказов окно ~6 ч (KASPI_DELIVERY за дни = тысячи; нужны только свежие).

---

## 1. Изменения в БД

### 1.1 Расширение `sales_order_status` enum

> ⚠️ `ALTER TYPE ... ADD VALUE` нельзя выполнять внутри транзакции, а Alembic
> по умолчанию оборачивает `upgrade()` в транзакцию. Поэтому добавление значения
> enum выносится в **ОТДЕЛЬНУЮ миграцию** (см. раздел 9), которая идёт перед
> миграцией со схемой. Внутри одной миграции новое значение enum использовать
> в том же `upgrade()` тоже нельзя (Postgres не видит закоммиченное значение).

```sql
-- Отдельная миграция, выполняется первой
ALTER TYPE sales_order_status ADD VALUE 'preorder' BEFORE 'packing';

-- Итоговый enum:
-- 'preorder'      → заказ ждёт прихода товара (preOrder=true на Kaspi)
-- 'packing'       → в сборке (обычный или экспресс)
-- 'ready_to_ship' → готов к отправке
-- 'shipped'       → отправлен
-- 'delivered'     → доставлен
-- 'cancelled'     → отменён
```

### 1.2 Новые поля в `sales_orders` и `warehouses`

```sql
ALTER TABLE sales_orders
    ADD COLUMN is_express   BOOLEAN NOT NULL DEFAULT FALSE,
    ADD COLUMN warehouse_id UUID    REFERENCES warehouses(id) ON DELETE SET NULL;
    -- warehouse_id: склад, с которого собирается заказ.
    -- Определяется при ingestion по pickupPointId заказа (НЕ по остаткам).

CREATE INDEX idx_so_is_express ON sales_orders (is_express)
    WHERE is_express = TRUE;
CREATE INDEX idx_so_warehouse_id ON sales_orders (warehouse_id);

-- Привязка нашего склада к Kaspi-точке отгрузки уже существует
-- (создана вместе с UI управления складами, хранит полную строку '16483752_PP1'):
--   warehouses.kaspi_store_id   (+ есть kaspi_city_id)
-- В миграции Epic 21 НЕ создаётся повторно. Резолв склада идёт по этому полю.
```

**Как отличить тип заказа:**

| Kaspi флаги | Наш статус | is_express |
|-------------|------------|------------|
| `preOrder=true` | `preorder` | false |
| `express=true` | `packing` | **true** |
| обычный | `packing` | false |

Экспресс не требует отдельного статуса — он определяется флагом `is_express`. Это позволяет фильтровать экспресс-заказы в любом статусе (не только `packing`).

### 1.3 Новая таблица `kaspi_order_meta`

Хранит всё специфичное для Kaspi — не засоряет `sales_orders`.

```sql
CREATE TABLE kaspi_order_meta (
    id              UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    sales_order_id  UUID         NOT NULL UNIQUE
                                 REFERENCES sales_orders(id) ON DELETE CASCADE,

    -- Идентификаторы Kaspi
    kaspi_order_id  VARCHAR(100) NOT NULL UNIQUE,  -- base64 ID (/v2/orders/{id})
    kaspi_code      VARCHAR(50)  NOT NULL,          -- числовой код '20013004'

    -- Флаги типа заказа
    pre_order           BOOLEAN  NOT NULL DEFAULT FALSE,
    express             BOOLEAN  NOT NULL DEFAULT FALSE,
    signature_required  BOOLEAN  NOT NULL DEFAULT FALSE,
    is_kaspi_delivery   BOOLEAN  NOT NULL DEFAULT FALSE,

    -- Тип доставки
    delivery_mode   VARCHAR(30),
    -- 'DELIVERY_PICKUP' | 'DELIVERY' | 'KASPI_DELIVERY'

    -- Оплата
    payment_mode    VARCHAR(50),
    -- 'PAY_WITH_CREDIT' | 'INSTANT_PAYMENTS' | ...

    -- Стоимость доставки
    delivery_cost   NUMERIC(12, 2),

    -- Адрес доставки (только для DELIVERY / KASPI_DELIVERY)
    delivery_address        TEXT,   -- formattedAddress
    delivery_lat            NUMERIC(10, 7),
    delivery_lon            NUMERIC(10, 7),

    -- Данные клиента от Kaspi (телефон может не совпадать с customers.phone)
    kaspi_customer_id   VARCHAR(100),
    kaspi_first_name    VARCHAR(100),
    kaspi_last_name     VARCHAR(100),
    kaspi_phone         VARCHAR(20),

    -- Временные метки Kaspi (epoch ms → конвертируем в timestamptz)
    kaspi_created_at        TIMESTAMPTZ,
    kaspi_approved_at       TIMESTAMPTZ,

    -- Статус на стороне Kaspi (для sync)
    kaspi_status        VARCHAR(50) NOT NULL DEFAULT 'ACCEPTED_BY_MERCHANT',
    -- 'ACCEPTED_BY_MERCHANT' | 'COMPLETED' | 'CANCELLED' | 'ARRIVED'

    kaspi_state         VARCHAR(50),
    -- 'NEW' | 'PICKUP' | 'DELIVERY' | 'KASPI_DELIVERY' | 'ARCHIVE'

    kaspi_pickup_point_id VARCHAR(100),
    -- pickupPointId из заказа — для трассировки определения склада

    synced_at           TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_kom_sales_order_id ON kaspi_order_meta (sales_order_id);
CREATE INDEX idx_kom_kaspi_status   ON kaspi_order_meta (kaspi_status);
CREATE INDEX idx_kom_synced_at      ON kaspi_order_meta (synced_at DESC);
```

### 1.4 Seed: маппинг статусов Kaspi

```sql
-- Получить id канала Kaspi
-- INSERT INTO order_status_mappings (sales_channel_id, marketplace_status, internal_status_id)
-- sales_channel_id = (SELECT id FROM sales_channels WHERE name = 'Kaspi')

INSERT INTO order_status_mappings (sales_channel_id, marketplace_status, internal_status_id)
SELECT
    sc.id,
    m.marketplace_status,
    os.id
FROM sales_channels sc
CROSS JOIN (VALUES
    ('ACCEPTED_BY_MERCHANT', 'packing'),
    ('ARRIVED',              'packing'),
    ('COMPLETED',            'delivered'),
    ('CANCELLED',            'cancelled')
) AS m(marketplace_status, internal_code)
JOIN order_statuses os ON os.code = m.internal_code
WHERE sc.name = 'Kaspi';
```

---

## 2. SQLAlchemy модели

```python
# app/models/kaspi_order_meta.py
class KaspiOrderMeta(Base):
    __tablename__ = "kaspi_order_meta"

    id              = Column(UUID(as_uuid=True), primary_key=True, default=uuid4)
    sales_order_id  = Column(UUID(as_uuid=True),
                             ForeignKey("sales_orders.id", ondelete="CASCADE"),
                             nullable=False, unique=True)

    kaspi_order_id  = Column(String(100), nullable=False, unique=True)
    kaspi_code      = Column(String(50), nullable=False)

    pre_order           = Column(Boolean, nullable=False, default=False)
    express             = Column(Boolean, nullable=False, default=False)
    signature_required  = Column(Boolean, nullable=False, default=False)
    is_kaspi_delivery   = Column(Boolean, nullable=False, default=False)

    delivery_mode       = Column(String(30))
    payment_mode        = Column(String(50))
    delivery_cost       = Column(Numeric(12, 2))
    delivery_address    = Column(Text)
    delivery_lat        = Column(Numeric(10, 7))
    delivery_lon        = Column(Numeric(10, 7))

    kaspi_customer_id   = Column(String(100))
    kaspi_first_name    = Column(String(100))
    kaspi_last_name     = Column(String(100))
    kaspi_phone         = Column(String(20))

    kaspi_created_at    = Column(DateTime(timezone=True))
    kaspi_approved_at   = Column(DateTime(timezone=True))

    kaspi_status        = Column(String(50), nullable=False,
                                 default="ACCEPTED_BY_MERCHANT")
    kaspi_state         = Column(String(50))
    kaspi_pickup_point_id = Column(String(100))
    assembled           = Column(Boolean, nullable=False, default=False)
    synced_at           = Column(DateTime(timezone=True),
                                 default=lambda: datetime.now(timezone.utc))

    sales_order         = relationship("SalesOrder", back_populates="kaspi_meta")


# Добавить в модель SalesOrder (в проекте — app/models/sales.py, НЕ sales_order.py):
is_express   = Column(Boolean, nullable=False, default=False)
warehouse_id = Column(UUID(as_uuid=True),
                      ForeignKey("warehouses.id", ondelete="SET NULL"),
                      nullable=True)

kaspi_meta   = relationship("KaspiOrderMeta",
                             back_populates="sales_order",
                             uselist=False)

# Добавить в Python-enum SalesOrderStatus (app/models/sales.py или enums.py):
#   PREORDER = "preorder"
# Миграция добавляет 'preorder' в PostgreSQL-enum, но Python-side enum об этом
# значении не знает — без члена PREORDER чтение заказа со статусом 'preorder'
# упадёт на десериализации (LookupError). Значение должно совпадать байт-в-байт.
```

---

## 3. Data flow: от Kaspi API до БД

### Шаг 1 — Что возвращает Kaspi API

```json
GET /v2/orders?filter[orders][status]=ACCEPTED_BY_MERCHANT

{
  "data": [{
    "type": "orders",
    "id": "MjAwMTMwMDQ=",
    "attributes": {
      "code": "20013004",
      "status": "ACCEPTED_BY_MERCHANT",
      "state":  "NEW",
      "totalPrice": 15900,
      "preOrder": false,
      "express":  false,
      "signatureRequired": false,
      "isKaspiDelivery": false,
      "deliveryMode": "DELIVERY_PICKUP",
      "paymentMode":  "PAY_WITH_CREDIT",
      "deliveryCost": 0,
      "creationDate": 1479470446241,
      "approvedByBankDate": 1479470451108,
      "customer": {
        "id": "Nzc3MDAwMDAwMA==",
        "firstName": "Иван",
        "lastName":  "Иванов",
        "cellPhone": "77001234567"
      },
      "deliveryAddress": {
        "formattedAddress": "г. Алматы, ул. Абая 1",
        "latitude":  43.238949,
        "longitude": 76.889709
      }
    }
  }]
}
```

```json
GET /v2/orders/{kaspi_order_id}/entries   (РЕАЛЬНЫЙ ответ, проверено 22.05.2026)

{
  "data": [{
    "id": "OTEyNjk1MDQ2IyMw",
    "type": "orderentries",
    "attributes": {
      "quantity":  1,
      "basePrice": 4771.0,
      "offer": {
        "code": "958781639",                                  // ← внутренний код Kaspi, НЕ наш SKU
        "name": "Повседневные шорты LIMIKO SHORT-72050-AK коричневый S"
      }
    },
    "relationships": {
      "product": { "data": { "id": "MTM3OTU3MDk3", "type": "masterproducts" } },
      "deliveryPointOfService": { "data": { "id": "16483752_PP1", "type": "pointofservices" } }
    }
  }]
}
```

> ⚠️ **РАСХОЖДЕНИЯ спеки с боевым API (учесть в P3/P4):**
> 1. **SKU.** Внутри `offer` НЕТ ключа `sku` — есть `code` (внутренний код Kaspi)
>    и `name`. Идентификатор для маппинга в `sku_mapping` = `offer.code`
>    (см. ниже, как кладётся в XML-прайс Epic 20). НЕ читать `offer.sku`.
> 2. **express** живёт в `attrs["kaspiDelivery"]["express"]`, а НЕ `attrs["express"]`.
>    Для не-Kaspi-доставки `kaspiDelivery` может отсутствовать → express=False.
> 3. **pickupPointId** приходит на уровне заказа (`attrs["pickupPointId"]`,
>    напр. "16483752_PP1") — резолв склада без доп. запросов. Подтверждено.
> 4. **Телефон** на заказах в статусе COMPLETED/ARCHIVE замаскирован Kaspi
>    ('+0(000)-...'); на активных (ACCEPTED_BY_MERCHANT) — реальный. Мы обрабатываем
>    только активные, поэтому _upsert_customer по телефону корректен.

### Шаг 2 — Последовательность записи в БД

```
Для каждой страницы (до 100 заказов):

asyncio.gather → параллельный fetch entries для всех заказов страницы

BEGIN TRANSACTION
│
├── для каждого заказа:
│   SAVEPOINT sp_{kaspi_code}
│   │
│   ├── 1. Дубликат-чек:
│   │      SELECT id FROM sales_orders
│   │      WHERE sales_channel_id = :kaspi AND marketplace_order_id = :code
│   │      → есть: UPDATE kaspi_status + synced_at, RELEASE, continue
│   │
│   ├── 2. Валидация entries:
│   │      для каждого entry → offer.code → SELECT product_id FROM sku_mapping
│   │      если хоть один не найден → ROLLBACK SAVEPOINT, Sentry error
│   │
│   ├── 3. Upsert customer:
│   │      SELECT id FROM customers WHERE phone = :phone
│   │      → нет: INSERT customers (full_name, phone)
│   │
│   ├── 4. Resolve warehouse:
│   │      по attrs.pickupPointId → warehouses.kaspi_store_id (полная строка)
│   │
│   ├── 5. INSERT sales_orders:
│   │      number='KASPI-20013004', marketplace_order_id='20013004'
│   │      status='preorder'|'packing', is_express=T|F
│   │      warehouse_id, total_amount, customer_id
│   │
│   ├── 6. INSERT kaspi_order_meta (все поля из attributes)
│   │
│   ├── 7. INSERT sales_order_lines (по каждому entry):
│   │      product_id из sku_mapping, quantity, unit_price=basePrice
│   │
│   ├── 8. Бронь остатков (только если status='packing'):
│   │      SELECT FOR UPDATE → reserved_quantity += N
│   │      (БЕЗ записи в ledger — товар физически не уехал;
│   │       preorder не бронируется — товара ещё нет)
│   │
│   RELEASE SAVEPOINT sp_{kaspi_code}
│
COMMIT
│
└── если created > 0 → upload_kaspi_pricelist.delay()
```

---

## 4. Сервисный слой — ingestion

```python
# app/services/kaspi_ingestion_service.py

async def ingest_kaspi_orders(
    db: AsyncSession,
    kaspi_orders: list[dict],   # уже обогащены полем _entries
) -> dict:
    """
    Идемпотентно обрабатывает список заказов.
    Каждый заказ — в отдельном savepoint.
    """
    stats = {"created": 0, "updated": 0, "skipped": 0, "failed": 0}
    kaspi_channel_id: int = await _get_kaspi_channel_id(db)

    for raw in kaspi_orders:
        try:
            async with db.begin_nested():
                result = await _process_single_order(db, raw, kaspi_channel_id)
                stats[result] += 1
        except Exception as e:
            stats["failed"] += 1
            sentry_sdk.capture_exception(e)

    return stats


async def _process_single_order(
    db: AsyncSession,
    raw: dict,
    kaspi_channel_id: int,
) -> str:
    attrs        = raw["attributes"]
    kaspi_id     = raw["id"]           # base64
    kaspi_code   = attrs["code"]       # числовой код
    kaspi_status = attrs["status"]
    entries      = raw["_entries"]     # уже prefetched через gather

    # 1. Дубликат-чек
    existing = await db.scalar(
        select(SalesOrder)
        .options(joinedload(SalesOrder.kaspi_meta))
        .where(SalesOrder.sales_channel_id     == kaspi_channel_id)
        .where(SalesOrder.marketplace_order_id == kaspi_code)
    )
    if existing:
        return await _update_order_status(db, existing, attrs)

    # 1b. Защита от создания пустого заказа.
    # sync_order_updates всегда шлёт _entries=[] — он предназначен ТОЛЬКО для
    # обновления уже существующих заказов. Если заказа у нас нет и позиций нет,
    # создавать нечего: без entries не пройдёт ни валидация SKU, ни резерв, и в
    # БД попадёт заказ с 0 строк и warehouse_id=NULL. Поэтому — skip.
    # (Если такой заказ реально новый, его подхватит poll_new_orders с entries.)
    if not entries:
        return "skipped"

    # 2. Валидация: все Kaspi SKU должны быть в sku_mapping.
    # kaspi_sku = offer.code (это и есть marketplace_sku в sku_mapping).
    # В прайс попадает только позиция с присвоенным kaspi_sku, поэтому
    # unmapped здесь — это рассинхрон (маппинг удалён после выгрузки), не норма.
    product_ids = []
    for entry in entries:
        kaspi_sku  = entry["attributes"]["offer"]["code"]
        product_id = await _resolve_product_by_kaspi_sku(db, kaspi_sku)
        if not product_id:
            sentry_sdk.capture_message(
                f"CRITICAL: Unmapped Kaspi SKU '{kaspi_sku}' in order {kaspi_code}",
                level="error",
            )
            raise ValueError(f"Unmapped SKU: {kaspi_sku}")  # → ROLLBACK SAVEPOINT
        product_ids.append((product_id, entry))

    # 3. Upsert customer
    customer = await _upsert_customer(db, attrs["customer"])

    # 4. Resolve warehouse — по pickupPointId из заказа
    warehouse_id = await _resolve_warehouse(db, attrs)

    # 5. INSERT sales_order
    is_preorder = attrs.get("preOrder", False)
    # express живёт в kaspiDelivery, а не на верхнем уровне (проверено боем).
    # Для не-Kaspi-доставки блока kaspiDelivery может не быть → express=False.
    is_express  = bool(attrs.get("kaspiDelivery", {}).get("express", False))
    order = SalesOrder(
        number               = f"KASPI-{kaspi_code}",
        sales_channel_id     = kaspi_channel_id,
        customer_id          = customer.id if customer else None,
        marketplace_order_id = kaspi_code,
        status               = "preorder" if is_preorder else "packing",
        is_express           = is_express,
        warehouse_id         = warehouse_id,
        total_amount         = attrs["totalPrice"],
    )
    db.add(order)
    await db.flush()

    # 6. INSERT kaspi_order_meta
    db.add(KaspiOrderMeta(
        sales_order_id     = order.id,
        kaspi_order_id     = kaspi_id,
        kaspi_code         = kaspi_code,
        pre_order          = is_preorder,
        express            = is_express,
        signature_required = attrs.get("signatureRequired", False),
        is_kaspi_delivery  = attrs.get("isKaspiDelivery", False),
        delivery_mode      = attrs.get("deliveryMode"),
        payment_mode       = attrs.get("paymentMode"),
        delivery_cost      = attrs.get("deliveryCost"),
        delivery_address   = attrs.get("deliveryAddress", {}).get("formattedAddress"),
        delivery_lat       = attrs.get("deliveryAddress", {}).get("latitude"),
        delivery_lon       = attrs.get("deliveryAddress", {}).get("longitude"),
        kaspi_customer_id  = attrs["customer"].get("id"),
        kaspi_first_name   = attrs["customer"].get("firstName"),
        kaspi_last_name    = attrs["customer"].get("lastName"),
        kaspi_phone        = attrs["customer"].get("cellPhone"),
        kaspi_created_at   = _epoch_ms_to_dt(attrs.get("creationDate")),
        kaspi_approved_at  = _epoch_ms_to_dt(attrs.get("approvedByBankDate")),
        kaspi_status       = kaspi_status,
        kaspi_state        = attrs.get("state"),
        kaspi_pickup_point_id = attrs.get("pickupPointId"),
        assembled          = attrs.get("assembled", False),
    ))

    # 7. INSERT sales_order_lines
    for product_id, entry in product_ids:
        db.add(SalesOrderLine(
            sales_order_id = order.id,
            product_id     = product_id,
            quantity       = entry["attributes"]["quantity"],
            unit_price     = entry["attributes"]["basePrice"],
        ))
    await db.flush()

    # 8. Резервирование остатков — ТОЛЬКО для 'packing'.
    # preorder создаётся без брони: товара физически нет, бронить нечего.
    # Бронь под preorder возникнет позже, при ручном переводе preorder→packing
    # в UI управления заказами («снятие предзаказа», вне скоупа Epic 21 — см. BR-21-05).
    if order.status == "packing":
        await _reserve_inventory(db, order)

    return "created"


async def _upsert_customer(db: AsyncSession, kaspi_customer: dict):
    """
    Ищет клиента по телефону, при отсутствии — создаёт.
    customers.phone — UNIQUE, поэтому учитываем два края:
      - заказ без телефона (или пустая строка) → возвращаем None (customer_id
        у заказа nullable), клиент не создаётся;
      - один телефон в нескольких заказах одной пачки → второй INSERT поймал бы
        нарушение UNIQUE и убил бы savepoint заказа. Поэтому при IntegrityError
        откатываемся к существующему (его создал предыдущий заказ пачки).
    """
    phone = (kaspi_customer.get("cellPhone") or "").strip()
    if not phone:
        return None

    customer = await db.scalar(select(Customer).where(Customer.phone == phone))
    if customer:
        return customer

    full_name = (
        f"{kaspi_customer.get('firstName', '')} "
        f"{kaspi_customer.get('lastName', '')}"
    ).strip() or None

    # Вложенный savepoint вокруг INSERT — чтобы гонка по UNIQUE(phone) не уронила
    # внешний savepoint заказа. Поймали конфликт → перечитали существующего.
    try:
        async with db.begin_nested():
            customer = Customer(full_name=full_name, phone=phone)
            db.add(customer)
            await db.flush()
        return customer
    except IntegrityError:
        return await db.scalar(select(Customer).where(Customer.phone == phone))


async def _resolve_product_by_kaspi_sku(db: AsyncSession, kaspi_sku: str) -> UUID | None:
    return await db.scalar(
        select(SkuMapping.product_id)
        .join(Marketplace, Marketplace.id == SkuMapping.marketplace_id)
        .where(Marketplace.code == "kaspi")
        .where(SkuMapping.marketplace_sku == kaspi_sku)
    )


async def _resolve_warehouse(db: AsyncSession, attrs: dict) -> UUID | None:
    """
    Определяет склад заказа по pickupPointId, который Kaspi отдаёт на уровне
    самого заказа (attrs['pickupPointId'], напр. '16483752_PP1').

    Kaspi гарантирует: одна накладная = одна точка отгрузки (pickupPointId
    задаётся на заказ, а не на позицию), поэтому товаров с разных складов
    в одном заказе быть не может (BR-21-12).

    Маппинг 'Kaspi point → наш склад' хранится на warehouses.kaspi_store_id
    (РЕАЛЬНОЕ имя поля в модели; назначается в UI складов). Совпадение прямое,
    по полной строке — kaspi_store_id содержит '16483752_PP1' целиком.
    """
    pickup_point_id = attrs.get("pickupPointId")
    if not pickup_point_id:
        logger.warning(
            f"Order {attrs.get('code')} has no pickupPointId — warehouse unresolved"
        )
        return None

    warehouse = await db.scalar(
        select(Warehouse).where(Warehouse.kaspi_store_id == pickup_point_id)
    )
    if not warehouse:
        # Точка есть, но не привязана ни к одному нашему складу — нужно завести
        # привязку в UI. Заказ создастся с warehouse_id=NULL, резерв не пройдёт.
        sentry_sdk.capture_message(
            f"Unmapped Kaspi pickupPointId '{pickup_point_id}' "
            f"in order {attrs.get('code')}",
            level="warning",
        )
        return None
    return warehouse.id


async def _reserve_inventory(db: AsyncSession, order: SalesOrder) -> None:
    """
    Бронирует остатки под заказ. SELECT FOR UPDATE — защита от race condition.

    Модель остатков: inventory_balances привязан к ЛОКАЦИИ (location_id), а
    локация — к складу (locations.warehouse_id). Остаток ещё разрезан по
    состоянию (condition_state_id). Поэтому баланс ищется через JOIN на locations
    с фильтром по складу заказа И по состоянию 'good' (норма) — Kaspi-заказ всегда
    продажа нового товара, бронировать брак нельзя.
    Допущение «один товар = одна локация на складе» (подтверждено): на складе
    заказа товар в состоянии good лежит ровно в одной локации.

    ВАЖНО — резерв НЕ списывает товар физически и НЕ пишет в ledger:
      - quantity (физически на локации) НЕ меняется — товар лежит на месте;
      - reserved_quantity += N — помечаем как забронированное;
      - available = quantity - reserved_quantity — это и уходит в stockCount Kaspi.
    Запись в inventory_transactions (ledger) — ТОЛЬКО при отгрузке (Epic 22).

    Вызывается только для заказов в статусе 'packing'. Для 'preorder' резерва нет
    (товара физически нет, BR-21-05). condition_state резолвится по code='good'
    (id не хардкодить).
    """
    good_state_id = await db.scalar(
        select(ConditionState.id).where(ConditionState.code == "good")
    )

    lines = (await db.scalars(
        select(SalesOrderLine).where(SalesOrderLine.sales_order_id == order.id)
    )).all()

    for line in lines:
        balance = await db.scalar(
            select(InventoryBalance)
            .join(Location, Location.id == InventoryBalance.location_id)
            .where(InventoryBalance.product_id == line.product_id)
            .where(Location.warehouse_id == order.warehouse_id)
            .where(InventoryBalance.condition_state_id == good_state_id)
            .with_for_update()
        )
        available = (balance.quantity - balance.reserved_quantity) if balance else 0
        if balance and available >= line.quantity:
            balance.reserved_quantity += line.quantity   # только бронь, без ledger
        else:
            # Резерв не создаём, но заказ всё равно остаётся (BR-21-05).
            logger.warning(
                f"Insufficient available stock for product {line.product_id} "
                f"in warehouse {order.warehouse_id} (order {order.number}): "
                f"need {line.quantity}, available {available}"
            )


async def _update_order_status(
    db: AsyncSession,
    order: SalesOrder,
    attrs: dict,
) -> str:
    """
    Синхронизация существующего заказа (sync_order_updates).

    ГРАНИЦА Эпик 21 / Эпик 22 (осознанное разделение):
    - 21 обрабатывает ТОЛЬКО отмену (CANCELLED → release резерва), т.к. она требует
      немедленного снятия брони, иначе товар «зависнет» забронированным.
    - Прочие переходы (доставлен/ARCHIVE-COMPLETED, возврат returnedToWarehouse,
      стадии передачи) — это Эпик 22 (управление статусами, в т.ч. запись в Kaspi).
      Здесь они НЕ меняют наш order.status (игнорируются, не падают).
    - Но meta-поля (kaspi_status, kaspi_state, assembled) АКТУАЛИЗИРУЕМ всегда —
      чтобы Эпик 22 имел свежие данные. Это не «обработка перехода», а поддержание
      зеркала Kaspi-состояния.
    """
    new_kaspi_status = attrs["status"]
    new_state        = attrs.get("state")
    new_assembled    = attrs.get("assembled", False)
    meta = order.kaspi_meta

    # Актуализация зеркала Kaspi-полей (всегда). Если ничего не изменилось — skip.
    changed = (
        meta.kaspi_status != new_kaspi_status
        or meta.kaspi_state != new_state
        or meta.assembled != new_assembled
    )
    if not changed:
        return "skipped"

    meta.kaspi_status = new_kaspi_status
    meta.kaspi_state  = new_state
    meta.assembled    = new_assembled
    meta.synced_at    = datetime.now(timezone.utc)

    # Смена НАШЕГО статуса — только на отмену (граница 21). Остальное — Эпик 22.
    if new_kaspi_status == "CANCELLED" and order.status != "cancelled":
        had_reservation = order.status == "packing"
        order.status = "cancelled"
        if had_reservation:
            await _release_inventory(db, order)

    return "updated"


async def _release_inventory(db: AsyncSession, order: SalesOrder) -> None:
    """
    Снимает бронь (зеркально к _reserve_inventory). Тоже без записи в ledger —
    бронь не была физическим движением, поэтому и снятие им не является.
    reserved_quantity -= N, quantity не трогаем.

    Вызывается только если у заказа реально была бронь, т.е. он был в статусе
    'packing'. Для отменённого 'preorder' резерва не было — снимать нечего,
    вызывать эту функцию не нужно (см. _update_order_status).

    Защита от ухода в минус: не снимаем больше, чем фактически забронировано.
    Баланс ищется так же, как в _reserve_inventory: JOIN на locations по складу
    заказа + состояние 'good'.
    """
    good_state_id = await db.scalar(
        select(ConditionState.id).where(ConditionState.code == "good")
    )

    lines = (await db.scalars(
        select(SalesOrderLine).where(SalesOrderLine.sales_order_id == order.id)
    )).all()

    for line in lines:
        balance = await db.scalar(
            select(InventoryBalance)
            .join(Location, Location.id == InventoryBalance.location_id)
            .where(InventoryBalance.product_id == line.product_id)
            .where(Location.warehouse_id == order.warehouse_id)
            .where(InventoryBalance.condition_state_id == good_state_id)
            .with_for_update()
        )
        if balance:
            balance.reserved_quantity = max(
                0, balance.reserved_quantity - line.quantity
            )


def _epoch_ms_to_dt(epoch_ms: int | None) -> datetime | None:
    if not epoch_ms:
        return None
    return datetime.fromtimestamp(epoch_ms / 1000, tz=timezone.utc)
```

---

## 4. Kaspi API клиент

```python
# app/integrations/kaspi_client.py
import httpx
from app.core.config import settings

KASPI_BASE = "https://kaspi.kz/shop/api"

class KaspiClient:
    def __init__(self):
        # ВАЖНО — заголовки проверены против боевого API (22.05.2026):
        #  - Accept ДОЛЖЕН быть application/vnd.api+json (Kaspi = JSON:API),
        #    с application/json возможны капризы.
        #  - User-Agent ОБЯЗАТЕЛЕН и НЕ должен быть дефолтным (httpx/curl/python-*):
        #    WAF Kaspi молча дропает запросы с сигнатурой инструмента в таймаут
        #    (0 байт ответа, без 4xx). Честная строка интеграции фильтр проходит.
        #    НЕ маскируемся под браузер (Mozilla/...) — это нечестно и хрупко,
        #    при ужесточении WAF браузерный UA без браузерных заголовков подозрительнее.
        self.headers = {
            "X-Auth-Token": settings.KASPI_AUTH_TOKEN,
            "Content-Type": "application/vnd.api+json",
            "Accept":       "application/vnd.api+json",
            "User-Agent":   "LIMIKO-ERP/1.0 (+kaspi-integration)",
        }

    async def get_new_orders(self, page: int = 0, window_hours: int = 6) -> dict:
        """
        Забирает заказы в работе. ФИЛЬТР — по state, НЕ по status (проверено боем):
        - Kaspi ИГНОРИРУЕТ filter[orders][status] (возвращает всё подряд, в т.ч. ARCHIVE);
        - filter[orders][state] РАБОТАЕТ и сужает к стадии.
        У нас вся доставка — Kaspi Delivery с автоприёмом, заказы минуют NEW и
        сразу попадают в state=KASPI_DELIVERY со status=ACCEPTED_BY_MERCHANT.
        Поэтому фильтруем state=KASPI_DELIVERY.

        Окно по creationDate ОБЯЗАТЕЛЬНО (без него 400) и узкое (~6 ч): KASPI_DELIVERY
        за 14 дней = тысячи заказов (вся доставка), нам нужны только свежие. При
        ~55 заказах/час окно 6 ч ≈ 330 заказов = 3-4 страницы. Дубликат-чек отсечёт
        уже заведённые. Окно с запасом на простой воркера (не 1 ч — хрупко к сбоям).
        """
        now_ms  = int(datetime.now(timezone.utc).timestamp() * 1000)
        from_ms = now_ms - window_hours * 60 * 60 * 1000
        async with httpx.AsyncClient(timeout=30) as client:
            r = await client.get(
                f"{KASPI_BASE}/v2/orders",
                headers=self.headers,
                params={
                    "page[number]": page,
                    "page[size]":   100,
                    "filter[orders][state]": "KASPI_DELIVERY",
                    "filter[orders][creationDate][$ge]": from_ms,
                    "filter[orders][creationDate][$le]": now_ms,
                },
            )
            r.raise_for_status()
            return r.json()

    async def get_updated_orders(
        self,
        updated_after_ms: int,  # epoch milliseconds
        page: int = 0,
    ) -> dict:
        """Sliding window — все заказы обновлённые после метки."""
        async with httpx.AsyncClient(timeout=30) as client:
            r = await client.get(
                f"{KASPI_BASE}/v2/orders",
                headers=self.headers,
                params={
                    "page[number]": page,
                    "page[size]":   100,
                    "filter[orders][updatedGe]": updated_after_ms,
                },
            )
            r.raise_for_status()
            return r.json()

    async def get_order_entries(self, kaspi_order_id: str) -> list[dict]:
        """Позиции заказа — отдельный запрос."""
        async with httpx.AsyncClient(timeout=30) as client:
            r = await client.get(
                f"{KASPI_BASE}/v2/orders/{kaspi_order_id}/entries",
                headers=self.headers,
            )
            r.raise_for_status()
            return r.json().get("data", [])
```

---

## 5. Celery задачи

> **Архитектура async-в-Celery (осознанное решение, не ошибка):**
> В Эпике 19 принят паттерн «Celery → sync (psycopg2)». Но Эпик 20 (`upload_kaspi_pricelist`)
> сделал исключение: задача синхронная, а внутри `asyncio.run()` крутит async-сервис на
> `async_session_factory`. Epic 21 следует ТОМУ ЖЕ образцу (задачи лягут в тот же
> `tasks/kaspi.py`), потому что: (1) ingestion и Kaspi-клиент уже async, (2) HTTP к Kaspi —
> главная стоимость задачи, async+gather по entries экономит время, (3) единообразие
> внутри файла tasks/kaspi.py важнее правила из Эпика 19.
> ОБЯЗАТЕЛЬНО `await async_engine.dispose()` в finally (как в upload_kaspi_pricelist) —
> иначе пул привязан к закрытому event loop и 2-й тик падает 'event loop is closed'.
> И ОБЯЗАТЕЛЬНО `await db.commit()` после ingest_kaspi_orders — сервис работает через
> savepoints и сам НЕ коммитит; без commit заказы откатятся при закрытии сессии.

```python
# app/tasks/kaspi.py  (добавить к существующему файлу — там уже есть upload_kaspi_pricelist)
# Импорты как в существующем файле: async_engine, async_session_factory из app.core.database

@celery_app.task(
    bind=True,
    base=BaseTaskWithLogging,
    name="kaspi.poll_new_orders",
    max_retries=3,
    default_retry_delay=60,
)
def poll_kaspi_new_orders(self, log_id=None, **kwargs):
    """
    Забирает новые заказы ACCEPTED_BY_MERCHANT, каждые 3 минуты.
    После ingestion → реактивная регенерация прайс-листа.
    """
    return asyncio.run(_async_poll_new_orders())


async def _async_poll_new_orders() -> dict:
    client = KaspiClient()
    total  = {"created": 0, "updated": 0, "failed": 0}
    page   = 0

    # ВАЖНО (паттерн из Эпика 20, tasks/kaspi.py): Celery-задача синхронная,
    # async-ingestion крутится внутри asyncio.run(). В finally — async_engine.dispose(),
    # иначе соединения пула привязаны к закрытому event loop и 2-й тик падает
    # с 'event loop is closed'. При поллинге каждые 3 мин это критично.
    try:
        async with async_session_factory() as db:
            while True:
                data   = await client.get_new_orders(page=page)
                orders = data.get("data", [])
                if not orders:
                    break

                # Параллельный fetch entries для всех заказов страницы
                entries_results = await asyncio.gather(*[
                    client.get_order_entries(o["id"]) for o in orders
                ], return_exceptions=True)

                # Обогащаем заказы; пропускаем те, у которых entries не получили
                enriched = []
                for order, entries in zip(orders, entries_results):
                    if isinstance(entries, Exception):
                        total["failed"] += 1
                        sentry_sdk.capture_exception(entries)
                        continue
                    order["_entries"] = entries
                    enriched.append(order)

                stats = await ingest_kaspi_orders(db, enriched)
                await db.commit()
                total["created"] += stats["created"]
                total["updated"] += stats["updated"]
                total["failed"]  += stats["failed"]
                page += 1

                if len(orders) < 100:
                    break
    finally:
        await async_engine.dispose()

    if total["created"] > 0:
        upload_kaspi_pricelist.delay(triggered_by="system")

    return {
        "processed": page * 100,
        **total,
    }


@celery_app.task(
    bind=True,
    base=BaseTaskWithLogging,
    name="kaspi.sync_order_updates",
    max_retries=3,
    default_retry_delay=60,
)
def sync_kaspi_order_updates(self, log_id=None, **kwargs):
    """
    Скользящее окно 30 минут — ловит отмены и статусные изменения.
    Запускается каждые 5 минут.
    """
    return asyncio.run(_async_sync_order_updates())


async def _async_sync_order_updates() -> dict:
    window_ms = int(
        (datetime.now(timezone.utc) - timedelta(minutes=30)).timestamp() * 1000
    )
    client = KaspiClient()
    total  = {"created": 0, "updated": 0, "failed": 0}
    page   = 0

    try:
        async with async_session_factory() as db:
            while True:
                data   = await client.get_updated_orders(
                    updated_after_ms=window_ms, page=page
                )
                orders = data.get("data", [])
                if not orders:
                    break

                # Для sync нам entries не нужны — только статусы.
                # _entries=[] → _process_single_order вернёт skip для новых заказов
                # (создание идёт только через poll, BR-21-13), а существующие обновит.
                for o in orders:
                    o["_entries"] = []

                stats = await ingest_kaspi_orders(db, orders)
                await db.commit()
                total["updated"] += stats["updated"]
                total["failed"]  += stats["failed"]
                page += 1

                if len(orders) < 100:
                    break
    finally:
        await async_engine.dispose()

    return {
        "processed": page * 100,
        **total,
    }
```

### Beat расписание (добавить в `celery_app.py`)

```python
"kaspi-poll-new-orders": {
    "task":     "kaspi.poll_new_orders",
    "schedule": crontab(minute="*/3"),   # каждые 3 минуты
    "kwargs":   {"triggered_by": "beat"},
},
"kaspi-sync-order-updates": {
    "task":     "kaspi.sync_order_updates",
    "schedule": crontab(minute="*/5"),   # каждые 5 минут
    "kwargs":   {"triggered_by": "beat"},
},
```

---

## 6. Pydantic-схемы

```python
# app/schemas/kaspi_order.py

class KaspiOrderMetaResponse(BaseModel):
    kaspi_order_id:     str
    kaspi_code:         str
    pre_order:          bool
    express:            bool
    signature_required: bool
    delivery_mode:      Optional[str]
    payment_mode:       Optional[str]
    delivery_cost:      Optional[Decimal]
    delivery_address:   Optional[str]
    kaspi_status:       str
    kaspi_state:        Optional[str]
    kaspi_pickup_point_id: Optional[str]
    kaspi_created_at:   Optional[datetime]
    kaspi_approved_at:  Optional[datetime]
    synced_at:          datetime
    model_config = {"from_attributes": True}


class SalesOrderLineResponse(BaseModel):
    id:         UUID
    product_id: UUID
    quantity:   int
    unit_price: Decimal
    model_config = {"from_attributes": True}


class SalesOrderResponse(BaseModel):
    id:                  UUID
    number:              str
    status:              str
    is_express:          bool
    warehouse_id:        Optional[UUID]
    total_amount:        Decimal
    customer_id:         Optional[UUID]
    created_at:          datetime
    kaspi_meta:          Optional[KaspiOrderMetaResponse]
    lines:               list[SalesOrderLineResponse]
    model_config = {"from_attributes": True}


class SalesOrderListResponse(BaseModel):
    items: list[SalesOrderResponse]
    total: int
    page:  int
    size:  int
```

---

## 7. API — эндпоинты

### Матрица прав доступа

| Действие | Endpoint | Permission |
|----------|----------|------------|
| Список заказов | `GET /api/v1/orders` | `orders:read` |
| Детали заказа | `GET /api/v1/orders/{id}` | `orders:read` |
| Ручной polling | `POST /api/v1/admin/job-logs/trigger` | `admin:trigger` |

```python
# app/routers/orders.py

@router.get("", response_model=SalesOrderListResponse)
async def list_orders(
    status:     Optional[str]  = Query(None),
    is_express: Optional[bool] = Query(None),
    channel:    Optional[str]  = Query(None),   # 'kaspi' | 'wholesale' | 'direct'
    page:       int            = Query(0, ge=0),
    size:       int            = Query(50, ge=1, le=200),
    db:         AsyncSession   = Depends(get_async_session),
):
    q = (
        select(SalesOrder)
        .options(
            joinedload(SalesOrder.kaspi_meta),
            selectinload(SalesOrder.lines),
        )
        .order_by(SalesOrder.created_at.desc())
    )
    if status:
        q = q.where(SalesOrder.status == status)
    if is_express is not None:
        q = q.where(SalesOrder.is_express == is_express)
    if channel:
        q = q.join(SalesChannel).where(SalesChannel.name.ilike(channel))

    total = await db.scalar(select(func.count()).select_from(q.subquery()))
    items = (await db.scalars(q.offset(page * size).limit(size))).all()

    return SalesOrderListResponse(
        items=[SalesOrderResponse.model_validate(o) for o in items],
        total=total, page=page, size=size,
    )
```

---

## 8. Бизнес-правила

| # | Правило |
|---|---------|
| BR-21-01 | Идемпотентность: `UNIQUE (sales_channel_id, marketplace_order_id)`. Повторный ingestion одного заказа → UPDATE статуса, не дублирование |
| BR-21-02 | Каждый заказ обрабатывается в `begin_nested()` (savepoint). Ошибка одного заказа не прерывает batch |
| BR-21-03 | Если Kaspi SKU (`offer.code`) из заказа не найден в `sku_mapping` — заказ целиком НЕ создаётся (ROLLBACK SAVEPOINT). Sentry alert уровня `error` (с кодом заказа + SKU). **Стратегия отказа выбрана осознанно (не частичное заведение):** в прайс Kaspi попадает только смаппленная позиция, значит заказать неизвестный SKU нельзя → unmapped = аномалия/рассинхрон, а не норма. Откат + громкий alert корректнее «грязного» заведения. Самовосстановление: добавил маппинг → следующий тик заведёт (BR-21-14). Проверено боем: на dev-БД (1 маппинг) реальные 300 заказов дали 278 failed по unmapped — код отработал верно, это нехватка данных, не баг |
| BR-21-04 | `preOrder=true` → статус `preorder`, `is_express=false`. `express=true` → статус `packing`, `is_express=true`. Обычный → статус `packing`, `is_express=false` |
| BR-21-05 | **Бронь — только при статусе `packing`, БЕЗ записи в ledger, на ЛОКАЦИИ.** Баланс ищется через JOIN `inventory_balances`→`locations` по складу заказа + `condition_state.code='good'` (бронируем только норму, не брак). `reserved_quantity += N`, `quantity` не меняется. Допущение: один товар = одна локация на складе. `preorder` не бронируется. Не хватает доступного (`quantity − reserved_quantity`) → warning, заказ всё равно создаётся. Физическое списание (`quantity −= N` + ledger с `location_id`) — только при отгрузке (Epic 22). Перевод `preorder→packing` и бронь в этот момент — UI «снятие предзаказа», вне Epic 21 |
| BR-21-06 | `sync_order_updates` АКТУАЛИЗИРУЕТ зеркало Kaspi-полей (kaspi_status, kaspi_state, assembled, synced_at) при любом изменении. Но НАШ `order.status` в Эпике 21 меняется ТОЛЬКО на отмену: `CANCELLED` → `cancelled` + release резерва (если был packing). Прочие переходы (доставлен, возврат, стадии) meta фиксирует, но наш статус НЕ трогает — это Эпик 22. Отменённый preorder брони не имел → только статус |
| BR-21-07 | Sliding window sync = 30 минут. Перекрытие с Beat-расписанием (5 мин) намеренное — гарантирует что ни одна отмена не потеряется |
| BR-21-09 | Клиент ищется по телефону (`customers.phone`, UNIQUE). Если не найден — создаётся. Телефон из `attrs.customer.cellPhone`, нормализуется (strip). Пустой телефон → `customer_id=NULL` (поле nullable). Один телефон в нескольких заказах одной пачки → второй заказ переиспользует созданного первым (IntegrityError → перечитать), savepoint заказа не рушится |
| BR-21-10 | После создания N новых заказов → `upload_kaspi_pricelist.delay()` для реактивного обновления `stockCount` (= `quantity − reserved_quantity`) |
| BR-21-11 | Поле `number` = `f"KASPI-{kaspi_code}"`. Kaspi code хранится и в `number`, и в `marketplace_order_id`, и в `kaspi_order_meta.kaspi_code` |
| BR-21-12 | **Склад заказа определяется по `pickupPointId`** (Kaspi отдаёт его на уровне заказа). Маппинг `pickupPointId → warehouse` хранится в существующем `warehouses.kaspi_store_id` (полная строка `16483752_PP1`, назначается в UI складов). Kaspi не формирует мультискладские накладные: одна накладная = одна точка отгрузки, поэтому товаров с разных складов в заказе быть не может. Если `pickupPointId` не привязан ни к одному складу → `warehouse_id=NULL`, бронь не проходит, Sentry warning |
| BR-21-13 | `sync_order_updates` работает только с существующими заказами. Заказ из окна `updatedDate`, которого нет в БД, пропускается (skip), а не создаётся пустым — создание идёт только через `poll_new_orders`, где есть entries |
| BR-21-14 | **Самовосстановление unmapped-SKU.** Заказ с ненайденным SKU откатывается (BR-21-03), но остаётся на Kaspi в статусе `ACCEPTED_BY_MERCHANT`. После добавления маппинга в UI следующий тик `poll_new_orders` (через 3 мин) подхватит его автоматически — ручной ретриггер не нужен. Риск: если продавец вручную сдвинет статус заказа в кабинете Kaspi, заказ выпадет из фильтра и не придёт |

---

## 9. Alembic-миграция

> ⚠️ **Известный баг исходной миграции (исправлено v1.13):** в первой версии у
> `kaspi_order_meta.id` отсутствовал `server_default=gen_random_uuid()` — INSERT
> падал с NotNullViolationError на `id`. Ниже уже исправлено. Если миграция БЕЗ
> дефолта уже применена к БД — НЕ пересоздавать таблицу, а догнать отдельной
> мини-миграцией: `ALTER TABLE kaspi_order_meta ALTER COLUMN id SET DEFAULT gen_random_uuid()`.

```python
# === МИГРАЦИЯ 1 (выполняется первой) =========================================
# migrations/versions/XXXX_epic21_enum_preorder.py
#
# ALTER TYPE ... ADD VALUE нельзя выполнять в транзакции, а Alembic оборачивает
# upgrade() в транзакцию. Поэтому отдельная миграция + отключаем транзакцию.

# В шапке файла:
revision = "epic21_enum_preorder"
down_revision = "<предыдущая_ревизия>"

def upgrade() -> None:
    # autocommit_block выводит выполнение из транзакции Alembic
    with op.get_context().autocommit_block():
        op.execute("ALTER TYPE sales_order_status ADD VALUE IF NOT EXISTS 'preorder' BEFORE 'packing'")

def downgrade() -> None:
    # enum value нельзя удалить в PostgreSQL без пересоздания типа — no-op
    pass


# === МИГРАЦИЯ 2 (зависит от миграции 1) ======================================
# migrations/versions/XXXX_epic21_kaspi_orders.py

revision = "epic21_kaspi_orders"
down_revision = "epic21_enum_preorder"   # ← строго после enum-миграции

def upgrade() -> None:
    # NB: привязка склада к Kaspi-точке УЖЕ существует (warehouses.kaspi_store_id,
    # создана с UI складов). В этой миграции НЕ создаём — иначе "column already exists".

    # 1. Новые поля в sales_orders ('preorder' уже существует в enum)
    op.add_column("sales_orders",
        sa.Column("is_express", sa.Boolean(), nullable=False, server_default="false"))
    op.add_column("sales_orders",
        sa.Column("warehouse_id", postgresql.UUID(as_uuid=True),
                  sa.ForeignKey("warehouses.id", ondelete="SET NULL")))
    op.create_index("idx_so_is_express", "sales_orders", ["is_express"],
                    postgresql_where=sa.text("is_express = TRUE"))
    op.create_index("idx_so_warehouse_id", "sales_orders", ["warehouse_id"])

    # 2. kaspi_order_meta
    op.create_table(
        "kaspi_order_meta",
        sa.Column("id",             postgresql.UUID(as_uuid=True), primary_key=True,
                  server_default=sa.text("gen_random_uuid()")),  # ОБЯЗАТЕЛЬНО — иначе INSERT даёт null в id
        sa.Column("sales_order_id", postgresql.UUID(as_uuid=True),
                  sa.ForeignKey("sales_orders.id", ondelete="CASCADE"),
                  nullable=False, unique=True),
        sa.Column("kaspi_order_id", sa.String(100), nullable=False, unique=True),
        sa.Column("kaspi_code",     sa.String(50),  nullable=False),
        sa.Column("pre_order",           sa.Boolean(), nullable=False, server_default="false"),
        sa.Column("express",             sa.Boolean(), nullable=False, server_default="false"),
        sa.Column("signature_required",  sa.Boolean(), nullable=False, server_default="false"),
        sa.Column("is_kaspi_delivery",   sa.Boolean(), nullable=False, server_default="false"),
        sa.Column("delivery_mode",       sa.String(30)),
        sa.Column("payment_mode",        sa.String(50)),
        sa.Column("delivery_cost",       sa.Numeric(12, 2)),
        sa.Column("delivery_address",    sa.Text()),
        sa.Column("delivery_lat",        sa.Numeric(10, 7)),
        sa.Column("delivery_lon",        sa.Numeric(10, 7)),
        sa.Column("kaspi_customer_id",   sa.String(100)),
        sa.Column("kaspi_first_name",    sa.String(100)),
        sa.Column("kaspi_last_name",     sa.String(100)),
        sa.Column("kaspi_phone",         sa.String(20)),
        sa.Column("kaspi_created_at",    sa.DateTime(timezone=True)),
        sa.Column("kaspi_approved_at",   sa.DateTime(timezone=True)),
        sa.Column("kaspi_status",        sa.String(50), nullable=False,
                  server_default="ACCEPTED_BY_MERCHANT"),
        sa.Column("kaspi_state",         sa.String(50)),
        sa.Column("kaspi_pickup_point_id", sa.String(100)),
        sa.Column("assembled",           sa.Boolean(), nullable=False, server_default="false"),
        sa.Column("synced_at",           sa.DateTime(timezone=True),
                  server_default=sa.text("NOW()")),
    )
    op.create_index("idx_kom_sales_order_id", "kaspi_order_meta", ["sales_order_id"])
    op.create_index("idx_kom_kaspi_status",   "kaspi_order_meta", ["kaspi_status"])
    # DESC-сортировка задаётся через sa.text в имени колонки, НЕ через postgresql_ops
    # (postgresql_ops — это operator class, а не порядок сортировки)
    op.create_index("idx_kom_synced_at",      "kaspi_order_meta",
                    [sa.text("synced_at DESC")])

    # 3. Seed статусных маппингов
    op.execute("""
        INSERT INTO order_status_mappings (sales_channel_id, marketplace_status, internal_status_id)
        SELECT sc.id, m.ms, os.id
        FROM sales_channels sc
        CROSS JOIN (VALUES
            ('ACCEPTED_BY_MERCHANT', 'packing'),
            ('ARRIVED',              'packing'),
            ('COMPLETED',            'delivered'),
            ('CANCELLED',            'cancelled')
        ) AS m(ms, code)
        JOIN order_statuses os ON os.code = m.code
        WHERE sc.name = 'Kaspi'
        ON CONFLICT (sales_channel_id, marketplace_status) DO NOTHING
    """)


def downgrade() -> None:
    op.drop_table("kaspi_order_meta")
    op.drop_column("sales_orders", "warehouse_id")
    op.drop_column("sales_orders", "is_express")
    # warehouses.kaspi_store_id НЕ трогаем — её создавала не эта миграция
    # enum value 'preorder' откатывается отдельной миграцией (no-op) —
    # удалять значение enum в PostgreSQL без пересоздания типа нельзя
```

---

## 10. Структура файлов

```
app/
├── integrations/
│   └── kaspi_client.py              # KaspiClient (get_new_orders, get_updated_orders, get_entries)
├── models/
│   ├── kaspi_order_meta.py          # новая модель
│   └── sales.py                     # SalesOrder + is_express, warehouse_id, kaspi_meta; enum PREORDER
├── schemas/
│   └── kaspi_order.py               # Response-схемы
├── services/
│   └── kaspi_ingestion_service.py   # ingest_kaspi_orders(), _process_single_order(),
│                                    # _resolve_warehouse(), _reserve_inventory(),
│                                    # _release_inventory(), _upsert_customer()
├── tasks/
│   └── kaspi.py                     # + poll_kaspi_new_orders, sync_kaspi_order_updates
│                                    # + _async_poll_new_orders, _async_sync_order_updates
└── routers/
    └── orders.py                    # GET /orders (список + фильтры)

migrations/
└── versions/XXXX_epic21_kaspi_orders.py
```

---

## 11. Статус реализации (промпты)

```
P1  Миграции (enum preorder + схема)            ✓ применено, upgrade/downgrade проверены
P2  Модели (KaspiOrderMeta, SalesOrder, enum)   ✓ маппер чист, relationship проверены
P3  Kaspi-клиент                                ✓ ПРОВЕРЕН БОЕМ: entries=1, offer.code ок
P4  Ingestion-сервис:
    4а создание заказа    ✓ ПРОВЕРЕНО: result=created (заказ+meta+lines+customer)
    4б инвентарь+статусы  ✓ ПРОВЕРЕНО БОЕМ: бронь 10/0→10/1 (quantity не тронут),
                            отмена 10/1→10/0, идемпотентность (дубль→update, count=1),
                            _resolve_warehouse фикс (kaspi_store_id), резерв через локацию+good
P5  Celery-задачи + Beat    ✓ ЛОГИКА ПРОВЕРЕНА БОЕМ: задачи зарегистрированы,
                            poll скачал 300 реальных заказов, фильтр state=KASPI_DELIVERY+окно 6ч,
                            пагинация ок, unmapped→failed корректно (278 на dev из-за 1 маппинга).
                            Полный created — на проде с полным sku_mapping.
P6  Роутер GET /orders                           ✓ ГОТОВ, проверен в Swagger (минимальный список)
Фронт (операционное управление заказами)         ⏳ ОТЛОЖЕН → Эпик 22 ч.2 (единый фронт
                                                    на полном API 21+22, чтобы не переделывать)

═══ ЭПИК 21 ЗАКРЫТ (backend) ═══
Граница 21/22: 21 = заказы входят в систему (поллинг) + отмена (release резерва) +
актуализация зеркала Kaspi-полей. 22 = полная матрица статус-переходов + смена
статусов с нашей стороны (запись в Kaspi: ASSEMBLE/накладная) + единый фронт.

ПЛАН (решение 23.05.2026): добить backend 21 → backend 22 (синхронизация + смена
статусов с нашей стороны) → ЕДИНЫЙ фронт операционного интерфейса. Фронт только
к 21 бесполезен (таблица «посмотреть» без действий → переделка под 22).
Фронт (список заказов)                           ⏳ отдельной сессией после backend
```

Проверенные боем факты (22.05.2026): канал к Orders API открыт, токен имеет скоуп
на заказы, заголовки WAF-совместимы (User-Agent LIMIKO-ERP, Accept vnd.api+json),
структура: SKU = offer.code, склад = pickupPointId, express = kaspiDelivery.express.
