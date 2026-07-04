# Спецификация 7 — Модели товаров и Дизайн-задачи (`models`, `design_tasks`)

---

## Архитектурный контекст

Модель — центральный объект PIM. Все остальные сущности либо питают её (справочники, поставщики), либо вырастают из неё (SKU, медиа, задачи). Карточка модели — самый тяжёлый GET в Sprint 1: агрегирует данные из 5 таблиц.

```
suppliers ──┐
categories ─┤
collections─┼──► models ──► products (SKU)
                    │──► media
                    └──► design_tasks ──► accounts (assignee)
```

**Жизненный цикл модели — State Machine:**

```
                  ┌─────────────┐
         ┌───────►│ development │◄──────────────────────┐
         │        └──────┬──────┘                       │
         │  [отзыв]      │ [активировать]               │ [только admin]
         │               ▼                              │
         │        ┌─────────────┐                       │
         │        │   active    │──────────────────────►│
         │        └──────┬──────┘  [снять с продаж]     │
         │               │                              │
         │               ▼                              │
         │        ┌──────────────┐                      │
         └────────│ discontinued │                      │
                  └──────┬───────┘                      │
                         │ [архивировать]               │
                         ▼                              │
                  ┌─────────────┐                       │
                  │  archived   │───────────────────────┘
                  └─────────────┘
```

Разрешённые переходы: `development → active`, `active → discontinued`, `discontinued → archived`, `discontinued → development` (отзыв), `archived → development` (только `rbac:roles:manage`).

---

## 1. Сценарий использования (Use Case)

### Матрица прав доступа

| Действие | Endpoint | Permission |
|----------|----------|------------|
| Создание модели | `POST /api/v1/models` | `pim:models:create` |
| Редактирование модели | `PATCH /api/v1/models/{id}` | `pim:models:update` |
| Смена lifecycle_status | `PATCH /api/v1/models/{id}/status` | `pim:models:update` |
| Список моделей | `GET /api/v1/models` | `pim:models:read` |
| Карточка модели | `GET /api/v1/models/{id}` | `pim:models:read` |
| Создание дизайн-задачи | `POST /api/v1/models/{id}/tasks` | `pim:tasks:manage` |
| Смена статуса задачи | `PATCH /api/v1/tasks/{task_id}/status` | `pim:tasks:manage` |

---

### Бизнес-правила

#### 7.1 — Создание модели

| # | Правило | Уровень | Ошибка |
|---|---------|---------|--------|
| BR-71-01 | `article` уникален по всей таблице `models` | DB + Service | 400 |
| BR-71-02 | `name` — обязательное поле, не пустая строка | Service | 422 |
| BR-71-03 | `supplier_id`, если передан — должен существовать и иметь `is_active = true` | Service | 400 |
| BR-71-04 | `category_id`, если передан — должен существовать | Service | 404 |
| BR-71-05 | `collection_id`, если передан — должен существовать | Service | 404 |
| BR-71-06 | `lifecycle_status` при создании всегда `development` — клиент не управляет этим полем | Service | — |
| BR-71-07 | UUID генерируется базой (`gen_random_uuid()`), клиент не передаёт | DB | — |
| BR-71-08 | `created_at` и `updated_at` проставляет база (`NOW()`) | DB | — |

#### 7.2 — Редактирование модели

| # | Правило | Ошибка |
|---|---------|--------|
| BR-72-01 | `article` нельзя изменить если `lifecycle_status = 'active'` или `'discontinued'` | 400 |
| BR-72-02 | `category_id` нельзя изменить если `lifecycle_status != 'development'` | 400 |
| BR-72-03 | `supplier_id` нельзя изменить если `lifecycle_status = 'active'` | 400 |
| BR-72-04 | Если передан `supplier_id` — поставщик должен существовать и быть активным | 400 |
| BR-72-05 | Если передан `category_id` — категория должна существовать | 404 |
| BR-72-06 | Если передан `collection_id` — коллекция должна существовать | 404 |
| BR-72-07 | `lifecycle_status` нельзя менять через `PATCH /models/{id}` — только через `/status` | 422 |

#### 7.3 — Смена lifecycle_status

| # | Правило | Ошибка |
|---|---------|--------|
| BR-73-01 | Переход `development → active` требует минимум 1 SKU | 400 |
| BR-73-02 | Переход `development → active` требует минимум 1 медиафайл | 400 |
| BR-73-03 | Переход `archived → development` требует права `rbac:roles:manage` | 403 |
| BR-73-04 | Запрещённые переходы: `development → discontinued`, `development → archived`, `active → archived` | 400 |
| BR-73-05 | При переходе в `archived` все незакрытые `design_tasks` → `cancelled` в той же транзакции | — |
| BR-73-06 | `reason` обязателен для `discontinued` и `archived` | 422 |

#### 7.4 — Список моделей

| # | Правило |
|---|---------|
| BR-74-01 | Фильтрация по `category_id` включает все дочерние категории рекурсивно (CTE) |
| BR-74-02 | Поиск по `search` — `article` с приоритетом точного совпадения, `name` через ilike |

#### 7.5 — Карточка модели

| # | Правило |
|---|---------|
| BR-75-01 | Вложенные `products` — только активные (`is_active = true`) если не передан `?include_inactive_skus=true` |
| BR-75-02 | Вложенные `design_tasks` — только незакрытые (`status NOT IN ('done', 'cancelled')`) по умолчанию |

#### 7.6 — Создание дизайн-задачи

| # | Правило | Ошибка |
|---|---------|--------|
| BR-76-01 | Модель должна существовать | 404 |
| BR-76-02 | Нельзя создавать задачи для `archived` модели | 400 |
| BR-76-03 | `assigned_to` — UUID аккаунта (`accounts.id`), не сотрудника. Если передан — аккаунт должен быть активным | 400 |
| BR-76-04 | `due_date` не может быть в прошлом | 400 |

#### 7.7 — Смена статуса дизайн-задачи

| # | Правило | Ошибка |
|---|---------|--------|
| BR-77-01 | Разрешённые переходы: `pending → in_progress`, `in_progress → review`, `review → done`, `review → in_progress`, любой → `cancelled` | 400 |
| BR-77-02 | Возврат из `done` — запрещён | 400 |
| BR-77-03 | `cancelled` — необратимый статус | 400 |

---

## 2. UI-компоненты

### Страница: Создание модели (`/models/new`)

**Layout:** Одностраничная форма, 2 колонки, кнопки в footer.

**Секция 1 — Основная информация:**

| Поле | Тип | Обязательное | Примечание |
|------|-----|:---:|-----------|
| Название (`name`) | `Input[text]` | ✅ | Placeholder: "Пальто женское зимнее" |
| Артикул (`article`) | `Input[text]` | ✅ | Inline-валидация уникальности debounce 500ms |
| Описание (`description`) | `Textarea` | ❌ | Rows: 4 |

**Секция 2 — Классификация:**

| Поле | Тип | Обязательное | Примечание |
|------|-----|:---:|-----------|
| Поставщик (`supplier_id`) | `Select` с поиском | ❌ | `GET /api/v1/suppliers?is_active=true` |
| Категория (`category_id`) | `TreeSelect` | ❌ | `GET /api/v1/categories/tree` |
| Коллекция (`collection_id`) | `Select` | ❌ | `GET /api/v1/collections` — отображает `name (season year)` |

**UX:** `lifecycle_status` на форме не отображается. После успеха — toast + редирект на `/models/{id}`. Ошибка дубля артикула — inline под полем.

---

### Страница: Список моделей (`/models`)

**Панель фильтров:**

| Элемент | Тип | Query-параметр |
|---------|-----|---------------|
| Поиск | `Input[text]` debounce | `search` |
| Поставщик | `Select` с поиском | `supplier_id` |
| Категория | `TreeSelect` | `category_id` |
| Коллекция | `Select` | `collection_id` |
| Статус ЖЦ | `MultiSelect` | `lifecycle_status[]` |

**Строка Card List:**

| Колонка | Содержимое |
|---------|-----------|
| Фото | Миниатюра 48×48 из `media` или placeholder |
| Артикул + Название | `article` жирным, `name` под ним |
| Категория / Поставщик | `category.name` / `supplier.name` |
| Статус | Badge: `development`→серый, `active`→зелёный, `discontinued`→жёлтый, `archived`→красный |
| SKU / Задачи | Счётчики активных SKU и открытых задач |

---

### Карточка модели (`/models/{id}`) — 4 вкладки

**Header (всегда виден):**
```
[Фото]  Пальто женское зимнее
        Артикул: PLT-W-2024-01
        [● active]  Поставщик: ООО Текстиль Групп
        Категория: Пальто  /  Коллекция: AW 2024

        [Редактировать]  [Сменить статус ▾]
```

**Вкладка 1 — Основное:** Форма редактирования. Поля блокируются по статусу (BR-72-01–03) с tooltip.

| Поле | При `active` |
|------|-------------|
| Название, Описание, Коллекция | Редактируемо |
| Артикул, Поставщик, Категория | 🔒 Заблокировано |

**Вкладка 2 — SKU:** Таблица SKU + кнопка "+ Добавить SKU" (детали — Эпик 8).

**Вкладка 3 — Галерея:** Grid 4×N, drag-and-drop для `sort_order` (детали — Эпик 10).

**Вкладка 4 — Задачи:** Kanban `pending → in_progress → review → done` + кнопка "+ Создать задачу".

---

### Модал: Смена статуса модели

| Элемент | Описание |
|---------|----------|
| Текущий статус | Badge read-only |
| Новый статус | `Select` — только разрешённые переходы |
| Причина (`reason`) | `Textarea` — обязательна для `discontinued` и `archived` |
| Предупреждение | При `→ archived`: "Все открытые задачи будут отменены" |

---

### Модал: Создание дизайн-задачи

| Поле | Тип | Обязательное |
|------|-----|:---:|
| Описание (`description`) | `Textarea` | ✅ |
| Исполнитель (`assigned_to`) | `Select` по аккаунтам | ❌ |
| Срок (`due_date`) | `DatePicker` (запрет прошлых дат) | ❌ |

---

## 3. API-контракт

### Pydantic-схемы

```python
# app/schemas/model.py

from pydantic import BaseModel, Field, field_validator, model_validator
from uuid import UUID
from typing import Optional
from datetime import date, datetime
from enum import Enum


class ModelLifecycleStatus(str, Enum):
    development  = "development"
    active       = "active"
    discontinued = "discontinued"
    archived     = "archived"


class DesignTaskStatus(str, Enum):
    pending     = "pending"
    in_progress = "in_progress"
    review      = "review"
    done        = "done"
    cancelled   = "cancelled"


# Графы переходов — единый источник правды
ALLOWED_TRANSITIONS: dict[ModelLifecycleStatus, set[ModelLifecycleStatus]] = {
    ModelLifecycleStatus.development:  {ModelLifecycleStatus.active},
    ModelLifecycleStatus.active:       {ModelLifecycleStatus.discontinued},
    ModelLifecycleStatus.discontinued: {
        ModelLifecycleStatus.archived,
        ModelLifecycleStatus.development,
    },
    ModelLifecycleStatus.archived:     {ModelLifecycleStatus.development},
}

TASK_ALLOWED_TRANSITIONS: dict[DesignTaskStatus, set[DesignTaskStatus]] = {
    DesignTaskStatus.pending:     {DesignTaskStatus.in_progress, DesignTaskStatus.cancelled},
    DesignTaskStatus.in_progress: {DesignTaskStatus.review, DesignTaskStatus.cancelled},
    DesignTaskStatus.review:      {DesignTaskStatus.done, DesignTaskStatus.in_progress, DesignTaskStatus.cancelled},
    DesignTaskStatus.done:        set(),
    DesignTaskStatus.cancelled:   set(),
}


# ── 7.1 CREATE ─────────────────────────────────────────────────────────────
class ModelCreateRequest(BaseModel):
    name:          str           = Field(..., min_length=1, max_length=255)
    article:       str           = Field(..., min_length=1, max_length=100)
    description:   Optional[str] = Field(default=None)
    supplier_id:   Optional[UUID] = Field(default=None)
    category_id:   Optional[int]  = Field(default=None, ge=1)
    collection_id: Optional[int]  = Field(default=None, ge=1)

    @field_validator("article")
    @classmethod
    def normalize_article(cls, v: str) -> str:
        return v.strip().upper()

    @field_validator("name")
    @classmethod
    def strip_name(cls, v: str) -> str:
        return v.strip()

    model_config = {
        "json_schema_extra": {
            "example": {
                "name": "Пальто женское зимнее",
                "article": "PLT-W-2024-01",
                "description": "Классическое пальто из шерсти",
                "supplier_id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
                "category_id": 5,
                "collection_id": 2
            }
        }
    }


# ── 7.2 UPDATE ─────────────────────────────────────────────────────────────
class ModelUpdateRequest(BaseModel):
    name:          Optional[str]  = Field(default=None, min_length=1, max_length=255)
    article:       Optional[str]  = Field(default=None, min_length=1, max_length=100)
    description:   Optional[str]  = Field(default=None)
    supplier_id:   Optional[UUID] = Field(default=None)
    category_id:   Optional[int]  = Field(default=None, ge=1)
    collection_id: Optional[int]  = Field(default=None, ge=1)

    @field_validator("article")
    @classmethod
    def normalize_article(cls, v: Optional[str]) -> Optional[str]:
        return v.strip().upper() if v else v

    @model_validator(mode="after")
    def at_least_one(self) -> "ModelUpdateRequest":
        if not any(v is not None for v in self.model_dump().values()):
            raise ValueError("At least one field must be provided")
        return self


# ── 7.3 STATUS TRANSITION ──────────────────────────────────────────────────
class ModelStatusTransitionRequest(BaseModel):
    status: ModelLifecycleStatus
    reason: Optional[str] = Field(default=None, max_length=500)

    @model_validator(mode="after")
    def reason_required_for_terminal(self) -> "ModelStatusTransitionRequest":
        terminal = {ModelLifecycleStatus.discontinued, ModelLifecycleStatus.archived}
        if self.status in terminal and not self.reason:
            raise ValueError(f"reason is required when transitioning to '{self.status.value}'")
        return self


# ── RESPONSE SCHEMAS ───────────────────────────────────────────────────────
class SupplierShortResponse(BaseModel):
    id:   UUID
    name: str
    model_config = {"from_attributes": True}


class CategoryShortResponse(BaseModel):
    id:   int
    name: str
    model_config = {"from_attributes": True}


class CollectionShortResponse(BaseModel):
    id:     int
    name:   str
    season: Optional[str]
    year:   Optional[int]
    model_config = {"from_attributes": True}


class ColorShortResponse(BaseModel):
    id:       int
    name:     str
    hex_code: Optional[str]
    model_config = {"from_attributes": True}


class SizeShortResponse(BaseModel):
    id:   int
    name: str
    model_config = {"from_attributes": True}


class ProductShortResponse(BaseModel):
    id:                     UUID
    sku:                    str
    barcode:                Optional[str]
    color:                  Optional[ColorShortResponse]
    size:                   Optional[SizeShortResponse]
    current_purchase_price: Optional[float]
    current_selling_price:  Optional[float]
    is_active:              bool
    model_config = {"from_attributes": True}


class MediaShortResponse(BaseModel):
    id:         UUID
    url:        str
    sort_order: int
    product_id: Optional[UUID]
    model_config = {"from_attributes": True}


class AssigneeShortResponse(BaseModel):
    id:        UUID
    username:  str
    full_name: str
    model_config = {"from_attributes": True}


class DesignTaskResponse(BaseModel):
    id:          UUID
    status:      DesignTaskStatus
    description: Optional[str]
    assigned_to: Optional[AssigneeShortResponse]
    due_date:    Optional[date]
    created_at:  datetime
    updated_at:  datetime
    model_config = {"from_attributes": True}


class ModelResponse(BaseModel):
    """Базовая схема — для списка и ответов на CREATE/UPDATE."""
    id:               UUID
    name:             str
    article:          str
    description:      Optional[str]
    lifecycle_status: ModelLifecycleStatus
    supplier:         Optional[SupplierShortResponse]
    category:         Optional[CategoryShortResponse]
    collection:       Optional[CollectionShortResponse]
    sku_count:        int = 0
    created_at:       datetime
    updated_at:       datetime
    model_config = {"from_attributes": True}


class ModelDetailResponse(ModelResponse):
    """Расширенная схема для GET /models/{id}."""
    products:     list[ProductShortResponse]
    media:        list[MediaShortResponse]
    design_tasks: list[DesignTaskResponse]


class PaginatedModelsResponse(BaseModel):
    items: list[ModelResponse]
    total: int
    page:  int
    size:  int
    pages: int


# ── 7.6 CREATE TASK ────────────────────────────────────────────────────────
class DesignTaskCreateRequest(BaseModel):
    description: str            = Field(..., min_length=1, max_length=2000)
    assigned_to: Optional[UUID] = Field(default=None)
    due_date:    Optional[date] = Field(default=None)

    @field_validator("due_date")
    @classmethod
    def not_past(cls, v: Optional[date]) -> Optional[date]:
        if v and v < date.today():
            raise ValueError("due_date cannot be in the past")
        return v


# ── 7.7 TASK STATUS TRANSITION ─────────────────────────────────────────────
class TaskStatusTransitionRequest(BaseModel):
    status:  DesignTaskStatus
    comment: Optional[str] = Field(default=None, max_length=500)
```

---

### Эндпоинты и HTTP-коды

#### `POST /api/v1/models` — 7.1

| Код | Ситуация |
|-----|----------|
| `201 Created` | `ModelResponse` |
| `400` | Дубль `article`; `supplier` неактивен |
| `404` | `category_id` / `collection_id` не найдены |
| `422` | Pydantic-валидация |

```json
// 201
{
  "id": "a1b2c3d4-...",
  "name": "Пальто женское зимнее",
  "article": "PLT-W-2024-01",
  "lifecycle_status": "development",
  "supplier": null,
  "category": null,
  "collection": null,
  "sku_count": 0,
  "created_at": "2024-10-15T10:30:00Z",
  "updated_at": "2024-10-15T10:30:00Z"
}
```

#### `PATCH /api/v1/models/{id}` — 7.2

| Код | Ситуация |
|-----|----------|
| `200 OK` | `ModelResponse` |
| `400` | Нарушение BR-72-01–04 |
| `404` | Модель / категория / коллекция не найдены |
| `422` | `lifecycle_status` в теле |

#### `PATCH /api/v1/models/{id}/status` — 7.3

| Код | Ситуация |
|-----|----------|
| `200 OK` | `ModelResponse` с новым статусом |
| `400` | Недопустимый переход; нет SKU/медиа для `→ active` |
| `403` | `archived → development` без `rbac:roles:manage` |
| `404` | Модель не найдена |

```json
{ "detail": "Transition to 'active' requires at least one active SKU" }
{ "detail": "Transition 'development' → 'discontinued' is not allowed" }
```

#### `GET /api/v1/models` — 7.4

```
GET /api/v1/models
  ?search=PLT
  &supplier_id=<uuid>
  &category_id=2
  &lifecycle_status=active
  &lifecycle_status=development
  &page=1&size=25
```

| Код | Ситуация |
|-----|----------|
| `200 OK` | `PaginatedModelsResponse` |

#### `GET /api/v1/models/{id}` — 7.5

```
GET /api/v1/models/{id}?include_inactive_skus=false
```

| Код | Ситуация |
|-----|----------|
| `200 OK` | `ModelDetailResponse` со всеми вложенными массивами |
| `404` | Модель не найдена |

#### `POST /api/v1/models/{id}/tasks` — 7.6

| Код | Ситуация |
|-----|----------|
| `201 Created` | `DesignTaskResponse` |
| `400` | Модель `archived`; `assigned_to` неактивен; `due_date` в прошлом |
| `404` | Модель не найдена |

#### `PATCH /api/v1/tasks/{task_id}/status` — 7.7

| Код | Ситуация |
|-----|----------|
| `200 OK` | `DesignTaskResponse` |
| `400` | Недопустимый переход |
| `404` | Задача не найдена |

---

## 4. Взаимодействие с БД (Data Flow PostgreSQL)

### Маппинг таблиц

**`models`:**

| Поле запроса | Колонка | Источник |
|---|---|---|
| `name` | `name` | Клиент |
| `article` | `article` | Клиент (`.strip().upper()`) |
| `description` | `description` | Клиент (nullable) |
| `supplier_id` | `supplier_id` | Клиент (nullable) |
| `category_id` | `category_id` | Клиент (nullable) |
| `collection_id` | `collection_id` | Клиент (nullable) |
| — | `lifecycle_status` | Хардкод `development` при CREATE; сервис при `/status` |
| — | `id` | `gen_random_uuid()` — база |
| — | `created_at` / `updated_at` | `NOW()` — база / сервис |

**`design_tasks`:**

| Поле запроса | Колонка | Источник |
|---|---|---|
| `description` | `description` | Клиент |
| `assigned_to` | `assigned_to` | Клиент (UUID аккаунта, nullable) |
| `due_date` | `due_date` | Клиент (nullable) |
| — | `model_id` | Из URL `{id}` |
| — | `status` | Хардкод `pending` при CREATE |

---

### Сервисный слой — псевдокод

```python
# app/services/model_service.py

# ── 7.1 CREATE ─────────────────────────────────────────────────────────────
async def create_model(db: AsyncSession, payload: ModelCreateRequest) -> Model:

    # BR-71-01: уникальность артикула
    existing = await db.scalar(select(Model).where(Model.article == payload.article))
    if existing:
        raise HTTPException(400, f"Article '{payload.article}' already exists")

    # BR-71-03: поставщик активен
    if payload.supplier_id:
        supplier = await db.get(Supplier, payload.supplier_id)
        if not supplier or not supplier.is_active:
            raise HTTPException(400, "Supplier is inactive or not found")

    # BR-71-04/05: FK-ссылки
    if payload.category_id:
        if not await db.get(Category, payload.category_id):
            raise HTTPException(404, f"Category with id={payload.category_id} not found")

    if payload.collection_id:
        if not await db.get(Collection, payload.collection_id):
            raise HTTPException(404, f"Collection with id={payload.collection_id} not found")

    new_model = Model(
        **payload.model_dump(),
        lifecycle_status=ModelLifecycleStatus.development  # BR-71-06
    )
    db.add(new_model)
    try:
        await db.commit()
    except IntegrityError:
        await db.rollback()
        raise HTTPException(400, f"Article '{payload.article}' already exists")

    await db.refresh(new_model)
    return new_model


# ── 7.2 UPDATE ─────────────────────────────────────────────────────────────
async def update_model(db: AsyncSession, model_id: UUID, payload: ModelUpdateRequest) -> Model:

    model = await db.get(Model, model_id)
    if not model:
        raise HTTPException(404, f"Model with id={model_id} not found")

    data = payload.model_dump(exclude_unset=True)

    if "article" in data and model.lifecycle_status in ("active", "discontinued"):
        raise HTTPException(400, f"Cannot change article: model is in '{model.lifecycle_status}' status")

    if "category_id" in data and model.lifecycle_status != "development":
        raise HTTPException(400, "Cannot change category: only allowed in 'development' status")

    if "supplier_id" in data and model.lifecycle_status == "active":
        raise HTTPException(400, "Cannot change supplier: model is in 'active' status")

    if "supplier_id" in data and data["supplier_id"]:
        supplier = await db.get(Supplier, data["supplier_id"])
        if not supplier or not supplier.is_active:
            raise HTTPException(400, "Supplier not found or is inactive")

    if "category_id" in data and data["category_id"]:
        if not await db.get(Category, data["category_id"]):
            raise HTTPException(404, f"Category id={data['category_id']} not found")

    if "collection_id" in data and data["collection_id"]:
        if not await db.get(Collection, data["collection_id"]):
            raise HTTPException(404, f"Collection id={data['collection_id']} not found")

    for field, value in data.items():
        setattr(model, field, value)
    model.updated_at = datetime.now(timezone.utc)

    try:
        await db.commit()
    except IntegrityError:
        await db.rollback()
        raise HTTPException(400, f"Article '{data.get('article')}' already exists")

    await db.refresh(model)
    return model


# ── 7.3 STATUS TRANSITION ──────────────────────────────────────────────────
async def transition_model_status(
    db: AsyncSession,
    model_id: UUID,
    payload: ModelStatusTransitionRequest,
    jwt_permissions: list[str],
) -> Model:

    model = await db.get(Model, model_id)
    if not model:
        raise HTTPException(404, f"Model with id={model_id} not found")

    current = ModelLifecycleStatus(model.lifecycle_status)
    target  = payload.status
    allowed = ALLOWED_TRANSITIONS.get(current, set())

    if target not in allowed:
        raise HTTPException(
            400,
            f"Transition '{current.value}' → '{target.value}' is not allowed. "
            f"Allowed: {[s.value for s in allowed]}"
        )

    # BR-73-03
    if current == ModelLifecycleStatus.archived and "rbac:roles:manage" not in jwt_permissions:
        raise HTTPException(403, "Transition from 'archived' requires 'rbac:roles:manage'")

    # BR-73-01/02
    if target == ModelLifecycleStatus.active:
        sku_count = await db.scalar(
            select(func.count(Product.id))
            .where(Product.model_id == model_id)
            .where(Product.is_active == True)
        )
        if not sku_count:
            raise HTTPException(400, "Transition to 'active' requires at least one active SKU")

        media_count = await db.scalar(
            select(func.count(Media.id)).where(Media.model_id == model_id)
        )
        if not media_count:
            raise HTTPException(400, "Transition to 'active' requires at least one media file")

    # BR-73-05: при архивации отменяем все открытые задачи
    if target == ModelLifecycleStatus.archived:
        await db.execute(
            update(DesignTask)
            .where(DesignTask.model_id == model_id)
            .where(DesignTask.status.in_(["pending", "in_progress", "review"]))
            .values(status="cancelled", updated_at=datetime.now(timezone.utc))
        )

    model.lifecycle_status = target.value
    model.updated_at       = datetime.now(timezone.utc)
    await db.commit()
    await db.refresh(model)
    return model


# ── 7.4 LIST ───────────────────────────────────────────────────────────────
async def list_models(
    db: AsyncSession,
    search: Optional[str],
    supplier_id: Optional[UUID],
    category_id: Optional[int],
    collection_id: Optional[int],
    lifecycle_status: Optional[list[str]],
    page: int,
    size: int,
) -> PaginatedModelsResponse:

    query = (
        select(Model)
        .options(
            joinedload(Model.supplier),
            joinedload(Model.category),
            joinedload(Model.collection),
        )
    )

    if supplier_id:
        query = query.where(Model.supplier_id == supplier_id)
    if collection_id:
        query = query.where(Model.collection_id == collection_id)
    if lifecycle_status:
        query = query.where(Model.lifecycle_status.in_(lifecycle_status))

    # BR-74-01: рекурсивный фильтр по категории
    if category_id:
        descendant_ids = await _get_category_descendant_ids(db, category_id)
        query = query.where(Model.category_id.in_(descendant_ids))

    # BR-74-02: поиск с приоритетом точного совпадения article
    if search:
        pattern = f"%{search}%"
        query = query.where(
            Model.article.ilike(pattern) | Model.name.ilike(pattern)
        ).order_by(
            (Model.article.ilike(search)).desc(),
            Model.name
        )
    else:
        query = query.order_by(Model.created_at.desc())

    total = await db.scalar(select(func.count(Model.id)).where(
        *query.whereclause.clauses if query.whereclause is not None else []
    ))
    offset  = (page - 1) * size
    models  = (await db.scalars(query.limit(size).offset(offset))).unique().all()

    # Агрегат sku_count одним запросом
    model_ids  = [m.id for m in models]
    sku_counts = {}
    if model_ids:
        rows = (await db.execute(
            select(Product.model_id, func.count(Product.id).label("cnt"))
            .where(Product.model_id.in_(model_ids))
            .where(Product.is_active == True)
            .group_by(Product.model_id)
        )).all()
        sku_counts = {row.model_id: row.cnt for row in rows}

    items = []
    for m in models:
        resp = ModelResponse.model_validate(m)
        resp.sku_count = sku_counts.get(m.id, 0)
        items.append(resp)

    return PaginatedModelsResponse(
        items=items, total=total, page=page, size=size,
        pages=math.ceil(total / size) if total > 0 else 0,
    )


async def _get_category_descendant_ids(db: AsyncSession, category_id: int) -> list[int]:
    result = await db.execute(text("""
        WITH RECURSIVE descendants AS (
            SELECT id FROM categories WHERE id = :root_id
            UNION ALL
            SELECT c.id FROM categories c
            JOIN descendants d ON c.parent_id = d.id
        )
        SELECT id FROM descendants
    """), {"root_id": category_id})
    return [row[0] for row in result.fetchall()]


# ── 7.5 GET DETAIL ─────────────────────────────────────────────────────────
async def get_model_detail(
    db: AsyncSession,
    model_id: UUID,
    include_inactive_skus: bool = False,
) -> ModelDetailResponse:
    """
    joinedload  — для одиночных FK (supplier, category, collection)
    selectinload — для массивов (products, media, design_tasks)
    Причина: joinedload на массивы создаёт декартово произведение строк.
    """
    model = await db.scalar(
        select(Model)
        .where(Model.id == model_id)
        .options(
            joinedload(Model.supplier),
            joinedload(Model.category),
            joinedload(Model.collection),
            selectinload(Model.media).order_by(Media.sort_order),
            selectinload(Model.design_tasks).options(
                joinedload(DesignTask.assignee).joinedload(Account.employee)
            ),
            selectinload(Model.products).options(
                joinedload(Product.color),
                joinedload(Product.size),
            ),
        )
    )
    if not model:
        raise HTTPException(404, f"Model with id={model_id} not found")

    products = model.products if include_inactive_skus else [p for p in model.products if p.is_active]
    open_statuses = {"pending", "in_progress", "review"}
    tasks = [t for t in model.design_tasks if t.status in open_statuses]

    task_responses = []
    for task in tasks:
        assignee = None
        if task.assignee and task.assignee.employee:
            emp = task.assignee.employee
            assignee = AssigneeShortResponse(
                id=task.assignee.id,
                username=task.assignee.username,
                full_name=f"{emp.first_name} {emp.last_name}",
            )
        task_responses.append(
            DesignTaskResponse.model_validate(task, update={"assigned_to": assignee})
        )

    sku_count = sum(1 for p in model.products if p.is_active)
    return ModelDetailResponse(
        **ModelResponse.model_validate(model, update={"sku_count": sku_count}).model_dump(),
        products=     [ProductShortResponse.model_validate(p) for p in products],
        media=        [MediaShortResponse.model_validate(m) for m in model.media],
        design_tasks= task_responses,
    )


# ── 7.6 CREATE TASK ────────────────────────────────────────────────────────
async def create_design_task(
    db: AsyncSession, model_id: UUID, payload: DesignTaskCreateRequest
) -> DesignTask:

    model = await db.get(Model, model_id)
    if not model:
        raise HTTPException(404, f"Model with id={model_id} not found")
    if model.lifecycle_status == "archived":
        raise HTTPException(400, "Cannot create tasks for archived models")

    if payload.assigned_to:
        account = await db.get(Account, payload.assigned_to)
        if not account or not account.is_active:
            raise HTTPException(400, "Assignee account not found or is inactive")

    task = DesignTask(
        model_id=model_id,
        description=payload.description,
        assigned_to=payload.assigned_to,
        due_date=payload.due_date,
        status=DesignTaskStatus.pending.value,
    )
    db.add(task)
    await db.commit()
    await db.refresh(task)
    return task


# ── 7.7 TASK STATUS TRANSITION ─────────────────────────────────────────────
async def transition_task_status(
    db: AsyncSession, task_id: UUID, payload: TaskStatusTransitionRequest
) -> DesignTask:

    task = await db.get(DesignTask, task_id)
    if not task:
        raise HTTPException(404, f"Task with id={task_id} not found")

    current = DesignTaskStatus(task.status)
    target  = payload.status
    allowed = TASK_ALLOWED_TRANSITIONS.get(current, set())

    if not allowed:
        raise HTTPException(400, f"Transition from '{current.value}' is not allowed: task is already {current.value}")

    if target not in allowed:
        raise HTTPException(
            400,
            f"Transition '{current.value}' → '{target.value}' is not allowed. "
            f"Allowed: {[s.value for s in allowed]}"
        )

    task.status     = target.value
    task.updated_at = datetime.now(timezone.utc)
    await db.commit()
    await db.refresh(task)
    return task
```

---

### selectinload vs joinedload — правило

| Ситуация | Стратегия | Причина |
|----------|-----------|---------|
| Одиночный FK (supplier, category, collection) | `joinedload` | Один JOIN, нет дублирования |
| Массив (products, media, design_tasks) | `selectinload` | Отдельный SELECT, нет декартова произведения |
| Объект внутри массива (task → assignee → employee) | `selectinload` → `joinedload` | Внешний для массива, внутренний для одного |

---

### Транзакционность

| Операция | Таблицы | Стратегия |
|----------|---------|-----------|
| CREATE model | `suppliers/categories/collections` (SELECT) + `models` (INSERT) | Одиночный `commit` + `IntegrityError` guard |
| UPDATE model | Те же SELECT + `models` (UPDATE) | Одиночный `commit` |
| STATUS → active | `products` (COUNT) + `media` (COUNT) + `models` (UPDATE) | 2 COUNT + 1 UPDATE, одна транзакция |
| STATUS → archived | `design_tasks` (UPDATE bulk) + `models` (UPDATE) | Оба UPDATE в одной транзакции — BR-73-05 |
| GET detail | 4 SELECT (selectinload) | Read-only |
| CREATE task | `models` (SELECT) + `accounts` (SELECT) + `design_tasks` (INSERT) | Одиночный commit |
| TRANSITION task | `design_tasks` (UPDATE) | Одиночный commit |

---

## Структура файлов

```
app/
├── routers/
│   ├── models.py        # 7.1: POST /models
│   │                    # 7.2: PATCH /models/{id}
│   │                    # 7.3: PATCH /models/{id}/status
│   │                    # 7.4: GET /models
│   │                    # 7.5: GET /models/{id}
│   │                    # 7.6: POST /models/{id}/tasks
│   └── tasks.py         # 7.7: PATCH /tasks/{task_id}/status
├── services/
│   └── model_service.py # весь сервисный слой + _get_category_descendant_ids
└── schemas/
    └── model.py         # все схемы + ALLOWED_TRANSITIONS + TASK_ALLOWED_TRANSITIONS
```
