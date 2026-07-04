# Спецификация: 22 — Kaspi: Управление статусами (backend)

---

## Архитектурный контекст

```
  Менеджер (UI, Epic 22b)
  │
  ├─ [Передать все] — все packing (НЕ экспресс) разом
  │   → packing → ready_to_ship
  │   → Kaspi API: POST /v2/orders {ASSEMBLE, numberOfSpace} на каждый заказ
  │       (это и генерирует накладную на стороне Kaspi)
  │   → скачать kaspiDelivery.waybill → объединить PDF по складам
  │   → Telegram: отправить файлы в топик склада
  │
  ├─ [Отправить] — подтвердить физическую отгрузку
  │   → ready_to_ship → shipped
  │   → inventory: снять бронь + физическое списание + ledger (location_id)
  │
  └─ [Отменить заказ] — ручная отмена нашей стороной
      → ТОЛЬКО preorder / packing (до ASSEMBLE) → cancelled
      → inventory: снять резерв (если был, т.е. packing)
      → Kaspi API: POST /v2/orders {code, CANCELLED, cancellationReason}

  Экспресс (TWA, склад) — отдельный поток
  └─ [Передать] на карточке — поштучно, против дедлайна
      → packing(express) → ready_to_ship  (тот же ASSEMBLE-пуш)

  Kaspi (sync_order_updates, Epic 21)
  ├─ COMPLETED → shipped → delivered  (Kaspi сам закрывает после доставки)
  └─ CANCELLED → cancelled + release  (клиент отменил / не забрал)
```

**Что мы пушим в Kaspi:** `ASSEMBLE` при передаче в доставку (генерирует накладную) и
`CANCELLED` при ручной отмене (только до ASSEMBLE).  
**Всё остальное:** Kaspi ставит сам, мы ловим через sliding window (Epic 21).

**Ограничение отмены (Kaspi API):** `CANCELLED` принимается, только если текущий статус
Kaspi — `ACCEPTED_BY_MERCHANT` или `APPROVED_BY_BANK` (= наши `preorder`/`packing`).
После `ASSEMBLE` отмена через API невозможна.

---

## 1. Полная матрица статусных переходов

```
Статус LIMIKO     Триггер                          Следующий    Kaspi push
────────────────────────────────────────────────────────────────────────────
preorder          [Товар прибыл] — ручной (Epic 22b)  packing     —
packing           [Передать все] / [Передать] (TWA)  ready_to_ship ASSEMBLE
ready_to_ship     [Отправить] — ручной              shipped       —
shipped           COMPLETED от Kaspi (sync)         delivered     —
любой             CANCELLED от Kaspi (sync)         cancelled     —
preorder/packing  [Отменить] — ручной               cancelled     CANCELLED
```

**`ready_to_ship`** — заказ передан в Kaspi Доставку (запушен `ASSEMBLE`), накладная
сгенерирована. С этого момента ручная отмена через API недоступна.

**`shipped`** — внутренний статус: товар физически ушёл со склада. На этом шаге:
- бронь снимается и конвертируется в физическое списание (`inventory_transactions`)
- заказ уходит из зоны ответственности склада

---

## 2. Изменения в БД

### 2.1 Новые поля в `sales_orders`

```sql
ALTER TABLE sales_orders
    ADD COLUMN cancellation_reason   VARCHAR(60),
    -- Kaspi-код причины: 'OUT_OF_STOCK', 'WRONG_PRICE' и т.д.
    ADD COLUMN cancellation_comment  TEXT,
    -- Свободный комментарий (необязательный, макс 1000 символов)
    ADD COLUMN cancelled_by          VARCHAR(20);
    -- 'merchant' | 'customer'
```

### 2.2 Справочник причин отмены `kaspi_cancellation_reasons`

```sql
CREATE TABLE kaspi_cancellation_reasons (
    code        VARCHAR(60)  PRIMARY KEY,
    name_ru     VARCHAR(200) NOT NULL,
    is_active   BOOLEAN      NOT NULL DEFAULT TRUE
);

INSERT INTO kaspi_cancellation_reasons (code, name_ru) VALUES
    ('OUT_OF_STOCK',                   'Товара нет в наличии'),
    ('MERCHANT_OUT_OF_STOCK',          'Нет у продавца'),
    ('WRONG_PRICE',                    'Неверная цена'),
    ('BUYER_CANCELLATION_BY_MERCHANT', 'Покупатель отменил через продавца'),
    ('PRODUCT_NOT_FOUND',              'Товар не найден'),
    ('INCORRECT_ORDER',                'Некорректный заказ'),
    ('DELIVERY_NOT_AVAILABLE',         'Доставка недоступна');

-- Примечание: полный список уточнить в документации Kaspi.
-- При необходимости добавляются через INSERT без миграции кода.
```

---

## 3. Kaspi Push клиент

> **WAF (урок Эпика 21):** Kaspi молча дропает запросы с дефолтным `User-Agent`
> httpx. Все методы ниже должны переиспользовать настройки `KaspiClient`
> (кастомный UA, `X-Auth-Token`, `Content-Type: application/vnd.api+json`),
> а не создавать «голый» `httpx.AsyncClient`. `KASPI_BASE` = `https://kaspi.kz/shop/api`.
> Все статусные пуши идут с тройкой `id` (raw["id"] = `kaspi_order_id`) + `code` + `status`.

```python
# app/integrations/kaspi_client.py  (добавить методы)

async def cancel_order(
    self,
    kaspi_order_id: str,   # = kaspi_order_meta.kaspi_order_id (raw["id"])
    kaspi_code:     str,   # = kaspi_order_meta.kaspi_code (attrs["code"])
    reason:         str,
    comment:        str | None = None,
) -> dict:
    """
    Отмена заказа на стороне Kaspi.
    POST /v2/orders со статусом CANCELLED.
    ⚠ Kaspi примет CANCELLED ТОЛЬКО если текущий статус заказа
      ACCEPTED_BY_MERCHANT или APPROVED_BY_BANK (= наши preorder/packing).
      После ASSEMBLE — вернёт ошибку (отмена невозможна).
    """
    attributes = {
        "code":               kaspi_code,
        "status":             "CANCELLED",
        "cancellationReason": reason,
    }
    if comment:
        attributes["cancellationComment"] = comment[:1000]

    payload = {"data": {"type": "orders", "id": kaspi_order_id, "attributes": attributes}}
    r = await self._client.post(f"{KASPI_BASE}/v2/orders", json=payload)
    r.raise_for_status()
    return r.json()


async def transmit_to_assembly(
    self,
    kaspi_order_id:  str,
    kaspi_code:      str,
    number_of_space: int = 1,   # кол-во мест/накладных; для одежды всегда 1
) -> dict:
    """
    Передача заказа в Kaspi Доставку (статус ASSEMBLE).
    Именно эта смена статуса ГЕНЕРИРУЕТ накладную: после успешного ответа
    в заказе появляется/обновляется kaspiDelivery.waybill.
    POST /v2/orders {status: "ASSEMBLE", numberOfSpace}.
    """
    payload = {
        "data": {
            "type": "orders",
            "id":   kaspi_order_id,
            "attributes": {
                "code":          kaspi_code,
                "status":        "ASSEMBLE",
                "numberOfSpace": str(number_of_space),
            },
        }
    }
    r = await self._client.post(f"{KASPI_BASE}/v2/orders", json=payload)
    r.raise_for_status()
    return r.json()
```

---

## 4. Сервисный слой

### 4.1 Передача (packing → ready_to_ship)

> Передача делает три вещи: меняет статус, пушит `ASSEMBLE` в Kaspi (это и
> генерирует накладную) и рассылает объединённый PDF в Telegram по складам.
>
> Каноничная реализация и единый эндпоинт — **`batch_ready_to_ship_all`**
> (раздел 5.5 + Эпик 22b, раздел 1.1): берёт все `packing` заказы (кроме
> экспресса), без передачи `order_ids` с фронта. Старый вариант с `order_ids`
> и мультиселектом убран (см. changelog, п.8).
>
> Поштучная передача (экспресс с TWA) — `ready_to_ship_one` (раздел 4.1a).
>
> Сортировка накладных — `_sort_orders_for_invoice` (раздел 5.4).

### 4.1a Поштучная передача (для экспресса с TWA)

```python
async def ready_to_ship_one(
    db:       AsyncSession,
    order_id: UUID,
    actor_id: UUID,
) -> dict:
    """
    Передаёт ОДИН заказ в Kaspi Доставку. Используется TWA-карточкой экспресса
    (тап [Передать] против дедлайна). Логика идентична batch_ready_to_ship_all,
    но для одного заказа.
    """
    order = await db.scalar(
        select(SalesOrder)
        .options(
            selectinload(SalesOrder.lines)
                .joinedload(SalesOrderLine.product)
                .joinedload(Product.model),
            joinedload(SalesOrder.kaspi_meta),
            joinedload(SalesOrder.warehouse),
        )
        .where(SalesOrder.id == order_id)
        .with_for_update()
    )
    if not order or order.status != "packing":
        raise InvalidStatusError("Передать можно только заказ в статусе packing")

    order.status     = "ready_to_ship"
    order.updated_at = datetime.now(timezone.utc)
    await db.flush()

    client     = KaspiClient()
    merged_pdf = await build_merged_invoices([order], client)  # внутри — пуш ASSEMBLE
    filename   = f"invoice_{order.number}_{date.today()}.pdf"
    warehouse  = await db.get(Warehouse, order.warehouse_id)
    await send_to_telegram_group(
        chat_id   = warehouse.telegram_chat_id,
        topic_id  = warehouse.telegram_topic_id,
        filename  = filename,
        pdf_bytes = merged_pdf,
        caption   = f"⚡ Экспресс: {order.number}",
    )
    return {"updated": 1, "pdf_urls": [filename]}
```


### 4.2 Отгрузка (ready_to_ship → shipped)

> **Модель остатков (сверено с БД, Эпик 21):** `inventory_balances` висит на
> `location_id`, НЕ на `warehouse_id`. Локация знает склад через `locations.warehouse_id`.
> Остаток разрезан по `condition_state_id`; продажа Kaspi — всегда `good`.
> Бронь (Эпик 21) увеличивала `reserved_quantity`, физический `quantity` не трогала,
> в ledger ничего не писала. Отгрузка — зеркало: снимаем бронь, списываем физически,
> пишем ledger. `condition_state` резолвится по `code='good'` (id не хардкодить).

```python
async def ship_order(
    db:       AsyncSession,
    order_id: UUID,
    actor_id: UUID,
) -> SalesOrder:
    """
    Подтверждает физическую отгрузку (ready_to_ship → shipped):
    - reserved_quantity -= N (снимаем бронь)
    - quantity          -= N (физически ушло со склада)
    - InventoryTransaction (ledger) с location_id, reference_type='sales_order_shipped'
    """
    order = await db.scalar(
        select(SalesOrder)
        .options(selectinload(SalesOrder.lines))
        .where(SalesOrder.id == order_id)
        .with_for_update()
    )
    if not order:
        raise NotFoundError(f"Order {order_id} not found")
    if order.status != "ready_to_ship":
        raise InvalidStatusError(
            f"Ожидается ready_to_ship, получен {order.status}"
        )

    good_state_id = await db.scalar(
        select(ConditionState.id).where(ConditionState.code == "good")
    )

    for line in order.lines:
        # Баланс ищем через JOIN на локации склада заказа, состояние good
        balance = await db.scalar(
            select(InventoryBalance)
            .join(Location, Location.id == InventoryBalance.location_id)
            .where(InventoryBalance.product_id        == line.product_id)
            .where(Location.warehouse_id              == order.warehouse_id)
            .where(InventoryBalance.condition_state_id == good_state_id)
            .with_for_update()
        )
        if not balance:
            # Брони не было (например, заказ создан без остатка, BR-21-05) —
            # списывать нечего. Логируем и не блокируем отгрузку.
            logger.warning(
                f"Нет баланса для product {line.product_id} на складе "
                f"{order.warehouse_id} при отгрузке заказа {order.number}"
            )
            continue

        # Снимаем бронь и физически списываем
        balance.reserved_quantity = max(0, balance.reserved_quantity - line.quantity)
        balance.quantity         -= line.quantity

        # Ledger — на уровне локации (не склада!)
        db.add(InventoryTransaction(
            product_id     = line.product_id,
            location_id    = balance.location_id,
            quantity       = -line.quantity,        # финальное списание
            reference_id   = order.id,
            reference_type = "sales_order_shipped",
            note           = f"Отгрузка заказа {order.number}",
        ))

    order.status     = "shipped"
    order.updated_at = datetime.now(timezone.utc)
    return order
```

### 4.3 Ручная отмена (→ cancelled + Kaspi push)

```python
async def cancel_order(
    db:        AsyncSession,
    order_id:  UUID,
    reason:    str,
    comment:   str | None,
    actor_id:  UUID,
) -> SalesOrder:
    """
    Ручная отмена заказа:
    - Переводит в cancelled
    - Освобождает резерв (если был)
    - Пушит CANCELLED в Kaspi API
    """
    order = await db.scalar(
        select(SalesOrder)
        .options(
            selectinload(SalesOrder.lines),
            joinedload(SalesOrder.kaspi_meta),
        )
        .where(SalesOrder.id == order_id)
        .with_for_update()
    )
    if not order:
        raise NotFoundError(f"Order {order_id} not found")
    # Отмена через API возможна ТОЛЬКО до передачи в доставку.
    # Kaspi примет CANCELLED лишь при ACCEPTED_BY_MERCHANT/APPROVED_BY_BANK
    # (= наши preorder/packing). После ASSEMBLE (ready_to_ship+) — нельзя.
    if order.status not in ("preorder", "packing"):
        raise InvalidStatusError(
            f"Отменить можно только preorder/packing, получен {order.status}"
        )

    # Захватываем исходный статус ДО перезаписи — иначе проверка брони
    # читала бы уже 'cancelled' и снимала бронь всегда (в т.ч. у preorder).
    had_reservation = order.status == "packing"

    # Валидация причины
    valid_reason = await db.scalar(
        select(KaspiCancellationReason.code)
        .where(KaspiCancellationReason.code     == reason)
        .where(KaspiCancellationReason.is_active == True)
    )
    if not valid_reason:
        raise ValueError(f"Неизвестная причина отмены: {reason}")

    # 1. Обновляем статус
    order.status               = "cancelled"
    order.cancellation_reason  = reason
    order.cancellation_comment = comment
    order.cancelled_by         = "merchant"
    order.updated_at           = datetime.now(timezone.utc)

    # 2. Снимаем бронь только если она была (packing). Preorder брони не имел.
    if had_reservation:
        await _release_inventory(db, order)

    # 3. Обновляем kaspi_meta
    if order.kaspi_meta:
        order.kaspi_meta.kaspi_status = "CANCELLED"
        order.kaspi_meta.synced_at    = datetime.now(timezone.utc)

    await db.flush()

    # 4. Push в Kaspi делается ПОСЛЕ коммита (в роутере) — см. BR-22-04
    return order


async def push_cancellation_to_kaspi(
    kaspi_order_id: str,
    kaspi_code:     str,
    reason:         str,
    comment:        str | None,
) -> None:
    """
    Вызывается ПОСЛЕ коммита транзакции.
    Отдельная функция — чтобы Kaspi-ошибка не откатила БД.
    """
    client = KaspiClient()
    try:
        await client.cancel_order(kaspi_order_id, kaspi_code, reason, comment)
    except Exception as e:
        # БД уже обновлена — логируем, но не падаем
        sentry_sdk.capture_exception(e)
        logger.error(
            f"Failed to push CANCELLED to Kaspi "
            f"for order {kaspi_order_id}: {e}"
        )
```

### 4.4 Получение COMPLETED от Kaspi (в Epic 21 sync)

```python
# Добавить в _update_order_status() в kaspi_ingestion_service.py:

if new_kaspi_status == "COMPLETED":
    if order.status == "shipped":
        order.status = "delivered"
    elif order.status in ("packing", "ready_to_ship"):
        # Kaspi завершил раньше чем мы отгрузили — edge case
        # Принудительно финализируем инвентарь и ставим delivered
        await _finalize_inventory_on_complete(db, order)
        order.status = "delivered"
```

---

## 5. Накладные: скачать с Kaspi → отсортировать → объединить PDF

### 5.1 Откуда берутся накладные

Накладная существует **не сразу**. Она генерируется на стороне Kaspi в момент
передачи заказа в доставку — то есть когда мы пушим `ASSEMBLE` (см. `transmit_to_assembly`,
раздел 3). Только после успешного `ASSEMBLE` в заказе появляется/обновляется
`kaspiDelivery.waybill`.

По документации Kaspi `waybill` — это **ссылка на печатную накладную** для заказов
Kaspi Доставки (а не имя файла, по которому надо угадывать эндпоинт). Природа ссылки
(полный URL или относительный путь от `KASPI_BASE`) уточняется спайком при первом
живом вызове.

Поток передачи: `ASSEMBLE` (на каждый заказ) → перечитать заказ → взять `waybill` →
скачать PDF → объединить → Telegram. Заказы `DELIVERY_PICKUP` (`isKaspiDelivery: false`)
накладных не имеют — в PDF не включаются (BR-22-07).

`kaspi_order_meta.waybill_filename` кэширует последнее известное значение ссылки
(может быть `NULL` до первого `ASSEMBLE`).

### 5.2 Изменение в `kaspi_order_meta`

```sql
ALTER TABLE kaspi_order_meta
    ADD COLUMN waybill_filename VARCHAR(255);
    -- ссылка на накладную из kaspiDelivery.waybill (кэш)
    -- NULL до первого ASSEMBLE и для DELIVERY_PICKUP заказов
```

Добавить в `KaspiOrderMeta` модель:
```python
waybill_filename = Column(String(255))
```

При ingestion (Epic 21) можно сидить значение, но у новых заказов оно ещё NULL —
ссылка реально появляется только после `ASSEMBLE`:
```python
waybill_filename = attrs.get("kaspiDelivery", {}).get("waybill"),
```

### 5.3 Скачивание накладных с Kaspi

```python
# app/integrations/kaspi_client.py  (добавить метод)

async def download_waybill(self, waybill_link: str) -> bytes:
    """
    Скачивает PDF накладной по ссылке из kaspiDelivery.waybill.
    waybill_link — это ССЫЛКА (по докам Kaspi), не имя файла.
    ⚠ Спайк при первом вызове: определить, абсолютный это URL или
      относительный путь. Если относительный — склеить с KASPI_BASE.
    Переиспользуем self._client (WAF-UA + X-Auth-Token), не создаём голый клиент.
    """
    url = waybill_link if waybill_link.startswith("http") else f"{KASPI_BASE}{waybill_link}"
    r = await self._client.get(url)
    r.raise_for_status()
    return r.content   # raw PDF bytes
```

### 5.4 Объединение PDF в сервисном слое

> **Порядок важен (СВЕРЕНО НА ЖИВОМ API):** накладная генерируется не мгновенно.
> POST `ASSEMBLE` возвращает в ответе `waybill = None` — это НЕ значит «накладной нет».
> Реальная ссылка появляется через секунды и видна только при GET-перечитывании
> заказа. Поэтому:
>   1. POST `transmit_to_assembly` — запускает сборку (ответу в части waybill НЕ верим);
>   2. GET `get_order_by_code` с коротким retry — пока в `kaspiDelivery.waybill` не появится ссылка;
>   3. скачать PDF по полученной ссылке.
> Особенность модели Kaspi: после ASSEMBLE `status` остаётся `ACCEPTED_BY_MERCHANT`,
> но `assembled=true` — это и есть признак успешной передачи (не сам `status`).
> `waybill` — абсолютный URL (`https://kaspi.kz/shop/api/waybill/...`), проверено.
> Заказы самовывоза (pickup) ссылки не дают — после N ретраев waybill всё ещё пустой → пропускаем (BR-22-07).

```python
# app/services/invoice_pdf_service.py
from pypdf import PdfWriter, PdfReader
import asyncio, io

WAYBILL_RETRIES = 4          # сколько раз перечитать заказ, ожидая появления ссылки
WAYBILL_RETRY_DELAY = 1.5    # пауза между перечитываниями, сек


async def _fetch_waybill_link(order: SalesOrder, client: KaspiClient) -> str | None:
    """
    POST ASSEMBLE (запуск генерации) → GET по code с retry, пока waybill не появится.
    Возвращает ссылку или None (pickup / накладная так и не сгенерировалась).
    """
    code = order.kaspi_meta.kaspi_code
    kid  = order.kaspi_meta.kaspi_order_id
    # 1. Запускаем сборку. waybill в ответе игнорируем — он почти всегда None.
    await client.transmit_to_assembly(kid, code)
    # 2. Перечитываем заказ, ждём появления ссылки.
    for attempt in range(WAYBILL_RETRIES):
        await asyncio.sleep(WAYBILL_RETRY_DELAY)
        fresh = await client.get_order_by_code(code)
        if fresh:
            waybill = fresh.get("attributes", {}).get("kaspiDelivery", {}).get("waybill")
            if waybill:
                return waybill
    return None


async def build_merged_invoices(
    orders:  list[SalesOrder],
    client:  KaspiClient,
) -> bytes:
    """
    1. Сортирует заказы по 3-уровневой логике
    2. Для каждого: ASSEMBLE → GET-retry за waybill-ссылкой
    3. Скачивает PDF параллельно и объединяет в правильном порядке
    Возвращает bytes объединённого PDF.
    """
    sorted_orders = _sort_orders_for_invoice(orders)

    # 1. Последовательно (статусные мутации Kaspi не любят гонок) запускаем ASSEMBLE
    #    и добываем ссылку на накладную через перечитывание.
    orders_with_waybill: list[tuple[SalesOrder, str]] = []
    for o in sorted_orders:
        if not (o.kaspi_meta and o.kaspi_meta.kaspi_order_id):
            continue
        try:
            waybill = await _fetch_waybill_link(o, client)
            if waybill:
                o.kaspi_meta.waybill_filename = waybill   # кэшируем последнюю ссылку
                orders_with_waybill.append((o, waybill))
            else:
                # pickup или накладная не сгенерировалась за отведённые ретраи
                logger.info(f"{o.number}: waybill не получен (вероятно pickup/задержка)")
        except Exception as e:
            logger.error(f"ASSEMBLE/waybill failed for {o.number}: {e}")
            sentry_sdk.capture_exception(e)

    # 2. Параллельное скачивание PDF
    pdf_bytes_list = await asyncio.gather(*[
        client.download_waybill(link) for _, link in orders_with_waybill
    ], return_exceptions=True)

    # 3. Объединяем через pypdf
    writer = PdfWriter()
    for (order, _), pdf_bytes in zip(orders_with_waybill, pdf_bytes_list):
        if isinstance(pdf_bytes, Exception):
            logger.error(f"Failed to download waybill for {order.number}: {pdf_bytes}")
            sentry_sdk.capture_exception(pdf_bytes)
            continue
        reader = PdfReader(io.BytesIO(pdf_bytes))
        for page in reader.pages:
            writer.add_page(page)

    output = io.BytesIO()
    writer.write(output)
    return output.getvalue()


def _sort_orders_for_invoice(orders: list[SalesOrder]) -> list[SalesOrder]:
    """
    3-уровневая сортировка:
    1. Заказы с несколькими позициями → по убыванию кол-ва позиций
    2. 1 позиция, qty > 1             → по убыванию qty
    3. 1 позиция, qty = 1             → по SKU (будущее: по локации)
    """
    def sort_key(order: SalesOrder):
        lines     = order.lines
        n_lines   = len(lines)
        total_qty = sum(l.quantity for l in lines)

        if n_lines > 1:
            return (0, -n_lines, 0, "")
        elif total_qty > 1:
            return (1, 0, -total_qty, "")
        else:
            sku = lines[0].product.sku if lines else ""
            return (2, 0, 0, sku)

    return sorted(orders, key=sort_key)
```

### 5.5 `batch_ready_to_ship_all` — каноничная реализация

```python
async def batch_ready_to_ship_all(
    db:       AsyncSession,
    actor_id: UUID,
) -> dict:
    """
    Переводит ВСЕ packing заказы (КРОМЕ экспресса) в ready_to_ship.
    Экспресс идёт своим потоком — поштучно с TWA (ready_to_ship_one),
    т.к. у него отдельный дедлайн (express_deadline_at).
    Фронт ID не передаёт — бэкенд сам собирает список.
    """
    orders = (await db.scalars(
        select(SalesOrder)
        .options(
            selectinload(SalesOrder.lines)
                .joinedload(SalesOrderLine.product)
                .joinedload(Product.model),
            joinedload(SalesOrder.kaspi_meta),
            joinedload(SalesOrder.warehouse),
        )
        .where(SalesOrder.status     == "packing")
        .where(SalesOrder.is_express == False)   # noqa: E712 — экспресс исключён
        .with_for_update()
    )).all()

    if not orders:
        raise HTTPException(400, "Нет обычных заказов в статусе packing")

    # 1. Меняем статус
    for order in orders:
        order.status     = "ready_to_ship"
        order.updated_at = datetime.now(timezone.utc)
    await db.flush()

    # 2. Группируем по складам, на каждый склад — ASSEMBLE → waybill → объединённый PDF
    by_warehouse: dict[UUID, list[SalesOrder]] = defaultdict(list)
    for order in orders:
        by_warehouse[order.warehouse_id].append(order)

    client   = KaspiClient()
    pdf_urls = []
    for warehouse_id, wh_orders in by_warehouse.items():
        merged_pdf = await build_merged_invoices(wh_orders, client)  # внутри пуш ASSEMBLE
        filename   = f"invoices_{warehouse_id}_{date.today()}.pdf"
        warehouse  = await db.get(Warehouse, warehouse_id)
        await send_to_telegram_group(
            chat_id   = warehouse.telegram_chat_id,
            topic_id  = warehouse.telegram_topic_id,
            filename  = filename,
            pdf_bytes = merged_pdf,
            caption   = (
                f"📦 Накладные: {len(wh_orders)} заказов\n"
                f"Дата: {date.today().strftime('%d.%m.%Y')}"
            ),
        )
        pdf_urls.append(filename)

    return {"updated": len(orders), "pdf_urls": pdf_urls}
```

> **Дисциплина сайд-эффектов:** `ASSEMBLE` — это уже мутация на стороне Kaspi
> (накладная сгенерирована и не «откатывается» вместе с нашей транзакцией).
> Поэтому коммит нашей БД делается ДО разослки Telegram-файлов (статусы +
> кэш `waybill_filename` фиксируем сразу после `build_merged_invoices`), а сама
> отправка в Telegram — best-effort после коммита: её сбой не должен откатывать
> уже выполненную передачу. Точную точку коммита определяет роутер
> (`get_async_session` не коммитит сам — см. Эпик 21).

---

## 6. Telegram уведомления

```python
# app/services/telegram_service.py

import httpx
from app.core.config import settings


async def send_to_telegram_group(
    chat_id:   str,
    topic_id:  str | None,
    filename:  str,
    pdf_bytes: bytes,
    caption:   str,
) -> None:
    """
    Отправляет PDF в топик супергруппы Telegram.
    
    chat_id  — ID супергруппы (например, -1001234567890)
    topic_id — ID топика (message_thread_id в Telegram API).
               None → отправка в основной чат без топика.
    """
    if not chat_id:
        logger.warning("Telegram chat_id не настроен для склада")
        return

    payload = {
        "chat_id": chat_id,
        "caption": caption,
    }
    if topic_id:
        payload["message_thread_id"] = topic_id

    async with httpx.AsyncClient(timeout=60) as client:
        r = await client.post(
            f"https://api.telegram.org/bot{settings.TELEGRAM_BOT_TOKEN}/sendDocument",
            data=payload,
            files={"document": (filename, pdf_bytes, "application/pdf")},
        )
        r.raise_for_status()
```

### Поля `telegram_chat_id` и `telegram_topic_id` в `warehouses`

```sql
ALTER TABLE warehouses
    ADD COLUMN telegram_chat_id  VARCHAR(50),
    -- ID супергруппы: '-1001234567890'
    -- Одна супергруппа на всю компанию
    ADD COLUMN telegram_topic_id VARCHAR(20);
    -- ID топика конкретного склада внутри супергруппы
    -- Как найти: переслать любое сообщение из топика боту
    -- и взять message_thread_id из webhook payload
```

---

## 7. Pydantic-схемы

```python
# app/schemas/kaspi_order.py  (добавить)

# Передача больше не принимает order_ids — фронт жмёт «Передать все»,
# бэкенд сам собирает packing-заказы (см. batch_ready_to_ship_all).

class BatchReadyToShipResponse(BaseModel):
    updated:   int
    pdf_urls:  list[str]


class ShipOrderResponse(BaseModel):
    id:     UUID
    status: str
    model_config = {"from_attributes": True}


class CancelOrderRequest(BaseModel):
    reason:  str
    comment: str | None = None

    @field_validator("comment")
    @classmethod
    def validate_comment_length(cls, v):
        if v and len(v) > 1000:
            raise ValueError("Комментарий не более 1000 символов")
        return v


class CancellationReasonResponse(BaseModel):
    code:    str
    name_ru: str
```

---

## 8. API — эндпоинты

### Матрица прав доступа

| Действие | Endpoint | Permission |
|----------|----------|------------|
| Передать все | `POST /api/v1/orders/batch/ready-to-ship-all` | `orders:write` |
| Передать один (TWA, экспресс) | `POST /api/v1/orders/{id}/ready-to-ship` | `orders:write` |
| Отгрузить | `POST /api/v1/orders/{id}/ship` | `orders:write` |
| Отменить | `POST /api/v1/orders/{id}/cancel` | `orders:cancel` |
| Список причин отмены | `GET /api/v1/orders/cancellation-reasons` | `orders:read` |

> **Порядок маршрутов:** static-сегменты выше параметризованных. То есть
> `/batch/ready-to-ship-all` и `/cancellation-reasons` объявлять ДО `/{order_id}/...`
> (правило из DEVELOPMENT.md).

`POST /batch/ready-to-ship-all` определён в Эпике 22b (раздел 1.1) — тело пустое.

```python
# app/routers/orders.py  (добавить к существующему)

@router.post("/{order_id}/ready-to-ship", response_model=BatchReadyToShipResponse)
async def ready_to_ship_one(
    order_id: UUID,
    db:       AsyncSession = Depends(get_async_session),
    actor_id: UUID         = Depends(get_current_user_id),
):
    result = await order_service.ready_to_ship_one(db, order_id, actor_id)
    await db.commit()
    return BatchReadyToShipResponse(**result)


@router.post("/{order_id}/ship", response_model=ShipOrderResponse)
async def ship_order(
    order_id: UUID,
    db:       AsyncSession = Depends(get_async_session),
    actor_id: UUID         = Depends(get_current_user_id),
):
    order = await order_service.ship_order(db, order_id, actor_id)
    await db.commit()
    return ShipOrderResponse.model_validate(order)


@router.post("/{order_id}/cancel", response_model=ShipOrderResponse)
async def cancel_order(
    order_id: UUID,
    body:     CancelOrderRequest,
    db:       AsyncSession = Depends(get_async_session),
    actor_id: UUID         = Depends(get_current_user_id),
):
    order = await order_service.cancel_order(
        db, order_id, body.reason, body.comment, actor_id
    )
    kaspi_order_id = order.kaspi_meta.kaspi_order_id
    kaspi_code     = order.kaspi_meta.kaspi_code
    reason         = body.reason
    comment        = body.comment
    await db.commit()

    # Push ПОСЛЕ коммита — ошибка Kaspi не откатит БД (BR-22-04)
    await push_cancellation_to_kaspi(kaspi_order_id, kaspi_code, reason, comment)

    return ShipOrderResponse.model_validate(order)


@router.get("/cancellation-reasons", response_model=list[CancellationReasonResponse])
async def list_cancellation_reasons(db: AsyncSession = Depends(get_async_session)):
    reasons = (await db.scalars(
        select(KaspiCancellationReason)
        .where(KaspiCancellationReason.is_active == True)
        .order_by(KaspiCancellationReason.name_ru)
    )).all()
    return [CancellationReasonResponse.model_validate(r) for r in reasons]
```

---

## 9. Бизнес-правила

| # | Правило |
|---|---------|
| BR-22-01 | Передача (`ready_to_ship_all`) берёт ВСЕ `packing` заказы, КРОМЕ экспресса (`is_express = true`). Если обычных `packing` нет → 400. Экспресс передаётся поштучно с TWA (`ready_to_ship_one`) |
| BR-22-02 | PDF генерируется отдельно для каждого склада. Заказы без `warehouse_id` попадают в отдельный файл «Без склада» |
| BR-22-03 | Сортировка накладных: (1) multi-line по убыванию кол-ва линий, (2) single-line multi-qty по убыванию qty, (3) single-line single-qty по SKU. Уровень 3 обновляется на маршрутную сортировку после занесения адресной сетки |
| BR-22-04 | Kaspi push `CANCELLED` выполняется ПОСЛЕ коммита транзакции БД. Ошибка Kaspi API → Sentry alert, но статус в LIMIKO уже `cancelled`. Пуш идёт с `id + code + cancellationReason` |
| BR-22-05 | Ручная отмена (с пушем в Kaspi) возможна ТОЛЬКО для `preorder` и `packing`. Kaspi принимает `CANCELLED` лишь при `ACCEPTED_BY_MERCHANT`/`APPROVED_BY_BANK`; после `ASSEMBLE` (наш `ready_to_ship`+) — отклоняет. Любой другой статус → 400. Отмена отгруженных/доставленных приходит только со стороны Kaspi через sync (Epic 21) |
| BR-22-06 | `ship_order` снимает бронь (`reserved_quantity -= N`) и физически списывает (`quantity -= N`) на балансе склада заказа в состоянии `good`, затем пишет `inventory_transactions` с `location_id` (НЕ `warehouse_id`), `reference_type='sales_order_shipped'`. Если баланса нет (брони не было) — warning, отгрузка не блокируется |
| BR-22-07 | Накладная появляется только ПОСЛЕ пуша `ASSEMBLE`. `DELIVERY_PICKUP` заказы накладной не дают — ловим пустой `waybill`/except, в PDF не включаем. `telegram_chat_id`/`telegram_topic_id` должны быть настроены для каждого склада; если не настроены → warning, процесс не прерывается |
| BR-22-08 | Передача в Kaspi (`ASSEMBLE`) идёт с `numberOfSpace=1`. POST ASSEMBLE только запускает генерацию накладной — `waybill` в ответе POST = None. Реальная ссылка добывается GET-перечитыванием заказа по `code` с retry (4 попытки × 1.5 сек). Признак успешной передачи — `assembled=true`, НЕ смена `status`. Скачивание PDF — параллельно после сбора ссылок |
| BR-22-09 | Причина отмены (`cancellationReason`) обязательна при ручной отмене. Список причин загружается из `kaspi_cancellation_reasons`. Комментарий ≤1000 символов |
| BR-22-10 | Kaspi `COMPLETED` полученный через sync → `delivered`. Если заказ ещё не в `shipped` (edge case) — принудительно финализируем инвентарь |
| BR-22-11 | Возврат отгруженного отменённого заказа (`quantity += N`) — ВНЕ скоупа Эпика 22. При отмене `shipped`-заказа со стороны Kaspi остатки автоматически НЕ восстанавливаются. Хук на будущее: `kaspiDelivery.returnedToWarehouse` в sync Эпика 21 |

---

## 10. Alembic-миграция

```python
# migrations/versions/XXXX_epic22_order_status_management.py

def upgrade() -> None:
    # 1. Новые поля в sales_orders
    op.add_column("sales_orders",
        sa.Column("cancellation_reason",  sa.String(60)))
    op.add_column("sales_orders",
        sa.Column("cancellation_comment", sa.Text()))
    op.add_column("sales_orders",
        sa.Column("cancelled_by",         sa.String(20)))

    # 2. telegram_chat_id + telegram_topic_id в warehouses
    op.add_column("warehouses",
        sa.Column("telegram_chat_id",  sa.String(50)))
    op.add_column("warehouses",
        sa.Column("telegram_topic_id", sa.String(20)))

    # 3. Поля в kaspi_order_meta (накладная + дедлайн экспресса, Эпик 22b)
    op.add_column("kaspi_order_meta",
        sa.Column("waybill_filename",    sa.String(255)))
    op.add_column("kaspi_order_meta",
        sa.Column("express_deadline_at", sa.DateTime(timezone=True)))

    # 4. Справочник причин отмены
    op.create_table(
        "kaspi_cancellation_reasons",
        sa.Column("code",      sa.String(60),  primary_key=True),
        sa.Column("name_ru",   sa.String(200), nullable=False),
        sa.Column("is_active", sa.Boolean(),   nullable=False,
                  server_default="true"),
    )
    op.execute("""
        INSERT INTO kaspi_cancellation_reasons (code, name_ru) VALUES
        ('OUT_OF_STOCK',                   'Товара нет в наличии'),
        ('MERCHANT_OUT_OF_STOCK',          'Нет у продавца'),
        ('WRONG_PRICE',                    'Неверная цена'),
        ('BUYER_CANCELLATION_BY_MERCHANT', 'Покупатель отменил через продавца'),
        ('PRODUCT_NOT_FOUND',              'Товар не найден'),
        ('INCORRECT_ORDER',               'Некорректный заказ'),
        ('DELIVERY_NOT_AVAILABLE',         'Доставка недоступна')
    """)


def downgrade() -> None:
    op.drop_table("kaspi_cancellation_reasons")
    op.drop_column("kaspi_order_meta", "express_deadline_at")
    op.drop_column("kaspi_order_meta", "waybill_filename")
    op.drop_column("warehouses",    "telegram_topic_id")
    op.drop_column("warehouses",    "telegram_chat_id")
    op.drop_column("sales_orders",  "cancelled_by")
    op.drop_column("sales_orders",  "cancellation_comment")
    op.drop_column("sales_orders",  "cancellation_reason")
```

> **Поле `kaspi_order_id` НЕ создаём** — оно уже есть в `kaspi_order_meta`
> с Эпика 21 (хранит `raw["id"]`, ресурсный id Kaspi для пушей).
> `express_deadline_at` ранее был отдельной миграцией в 22b — теперь
> консолидирован сюда (одна миграция на эпик, урок Эпика 19).

---

## 10a. RBAC

Новый пермишен **`orders:cancel`** (остальные — `orders:read`, `orders:write` — уже
есть с Эпика 21). Что сделать:

```
1. Зарегистрировать 'orders:cancel' в app/core/permissions.py (реестр).
2. При деплое: docker compose exec backend python scripts/seed_admin.py
3. Проверить: SELECT code FROM permissions WHERE code LIKE 'orders:%';
   (ожидаем orders:read, orders:write, orders:cancel)
```

Это шаг (d) из чек-листа деплоя DEVELOPMENT.md.

---

## 11. Переменные окружения

```bash
# Telegram (добавить к существующим)
# Примечание: chat_id и topic_id хранятся в таблице warehouses,
# не в .env. Настраиваются через UI или прямым INSERT.
# Единственное что нужно в .env:
TELEGRAM_BOT_TOKEN=xxxxx:yyyyy   # токен бота (уже есть из Sprint 1)
```

---

## 12. Структура файлов

```
app/
├── integrations/
│   └── kaspi_client.py              # + cancel_order(+code), transmit_to_assembly(), download_waybill()
├── core/
│   └── permissions.py              # + 'orders:cancel'
├── models/
│   ├── kaspi_cancellation_reason.py # новая модель-справочник
│   ├── kaspi_order_meta.py          # + waybill_filename, express_deadline_at (kaspi_order_id уже есть)
│   └── sales_order.py               # + cancellation_reason, cancellation_comment, cancelled_by
├── schemas/
│   └── kaspi_order.py               # + BatchReadyToShipResponse/Ship/Cancel схемы
├── services/
│   ├── kaspi_order_service.py       # batch_ready_to_ship_all(), ready_to_ship_one(), ship_order(), cancel_order(), push_cancellation_to_kaspi()
│   ├── invoice_pdf_service.py       # build_merged_invoices() (пуш ASSEMBLE внутри), _sort_orders_for_invoice()
│   └── telegram_service.py          # send_to_telegram_group()
└── routers/
    └── orders.py                    # + /batch/ready-to-ship-all, /{id}/ready-to-ship, /{id}/ship, /{id}/cancel

migrations/
└── versions/XXXX_epic22_order_status_management.py
```
