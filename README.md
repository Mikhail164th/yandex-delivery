# Yandex Delivery API

Техническая документация для интеграции с API Яндекс Доставки и Яндекс Еды.

## TL;DR

- **Яндекс.Еда для клиентов** — публичного API нет, только reverse engineering
- **Яндекс.Еда для ресторанов** — через Vendor Portal или POS-интеграции (iiko, r_keeper)
- **Яндекс Доставка** — полноценный B2B API, документация ниже

---

## Содержание

- [Яндекс Доставка API](#яндекс-доставка-api)
- [Яндекс Еда — Ресторанам](#яндекс-еда--ресторанам)
- [Яндекс Еда — Клиентский API](#яндекс-еда--клиентский-api)

---

# Яндекс Доставка API

B2B API для автоматизации доставки.

## Авторизация

```http
Authorization: Bearer <OAuth-token>
```

Токен получить: [dostavka.yandex.ru/auth](https://dostavka.yandex.ru/auth/) → Интеграции → Получить токен

## Среды

| Среда | URL |
|-------|-----|
| Production | `https://b2b.taxi.yandex.net` |
| Production (Other Day) | `https://b2b-authproxy.taxi.yandex.net` |
| Test | `https://b2b.taxi.tst.yandex.net` |

### Тестовые данные

Из [официальной документации](https://yandex.ru/support2/delivery-profile/ru/api/other-day/index):

```
Token: y2_AgAAAAD04omrAAAPeAAAAAACRpC94Qk6Z5rUTgOcTgYFECJllXYKFx8
Warehouse: fbed3aa1-2cc6-4370-ab4d-59c5cc9bb924
```

## Обязательные заголовки

```http
Authorization: Bearer <token>
Content-Type: application/json
Accept-Language: ru
```

## Express Delivery — Эндпоинты

Base: `https://b2b.taxi.yandex.net`

### Создание заказа

| Метод | Endpoint | Описание |
|-------|----------|----------|
| POST | `/b2b/cargo/integration/v2/offers/calculate` | Варианты доставки и цены |
| POST | `/b2b/cargo/integration/v2/claims/create?request_id={uuid}` | Создать заявку |
| POST | `/b2b/cargo/integration/v2/claims/info` | Статус заявки |
| POST | `/b2b/cargo/integration/v2/claims/accept` | Подтвердить (запускает поиск курьера) |
| POST | `/b2b/cargo/integration/v2/claims/cancel` | Отменить |

### Цены и тарифы

| Метод | Endpoint | Описание |
|-------|----------|----------|
| POST | `/b2b/cargo/integration/v2/check-price` | Предварительный расчёт |
| POST | `/b2b/cargo/integration/v2/tariffs` | Доступные тарифы в точке |

### Отслеживание

| Метод | Endpoint | Описание |
|-------|----------|----------|
| GET | `/b2b/cargo/integration/v2/claims/performer-position` | GPS курьера |
| POST | `/b2b/cargo/integration/v2/claims/points-eta` | ETA по точкам |
| GET | `/b2b/cargo/integration/v2/claims/tracking-links` | Ссылка отслеживания для клиента |
| POST | `/b2b/cargo/integration/v2/driver-voiceforwarding` | Телефон курьера |

### Подтверждение доставки

| Метод | Endpoint | Описание |
|-------|----------|----------|
| POST | `/b2b/cargo/integration/v2/claims/confirmation_code` | Код подтверждения |
| POST | `/b2b/cargo/integration/v2/claims/proof-of-delivery/info` | Фото/подпись доставки |

### Изменение заказа

| Метод | Endpoint | Описание |
|-------|----------|----------|
| POST | `/b2b/cargo/integration/v2/claims/edit` | Изменить (до подтверждения) |
| POST | `/b2b/cargo/integration/v2/claims/apply-changes/request` | Изменить (после подтверждения) |
| POST | `/b2b/cargo/integration/v2/claims/apply-changes/result` | Результат изменения |
| POST | `/b2b/cargo/integration/v2/claims/return` | Возврат |

### Массовые операции

| Метод | Endpoint | Описание |
|-------|----------|----------|
| POST | `/b2b/cargo/integration/v2/claims/search` | Поиск заявок |
| POST | `/b2b/cargo/integration/v2/claims/bulk_info` | Инфо по списку заявок |
| POST | `/b2b/cargo/integration/v2/claims/journal` | История изменений |

## Примеры запросов

### Получить тарифы

```bash
curl -X POST "https://b2b.taxi.tst.yandex.net/b2b/cargo/integration/v2/tariffs" \
  -H "Authorization: Bearer y2_AgAAAAD04omrAAAPeAAAAAACRpC94Qk6Z5rUTgOcTgYFECJllXYKFx8" \
  -H "Content-Type: application/json" \
  -H "Accept-Language: ru" \
  -d '{"start_point": [37.6173, 55.7558]}'
```

Ответ:
```json
{
  "available_tariffs": [
    {"name": "courier", "title": "Курьер", "minimal_price": 160.0},
    {"name": "express", "title": "Экспресс", "minimal_price": 190.0},
    {"name": "cargo", "title": "Грузовой", "minimal_price": 759.0}
  ]
}
```

### Создать заявку

```bash
curl -X POST "https://b2b.taxi.yandex.net/b2b/cargo/integration/v2/claims/create?request_id=$(uuidgen)" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept-Language: ru" \
  -d '{
    "items": [{
      "title": "Документы",
      "quantity": 1,
      "cost_value": "100",
      "cost_currency": "RUB",
      "pickup_point": 1,
      "droppof_point": 2,
      "size": {"length": 0.1, "width": 0.1, "height": 0.1},
      "weight": 0.5
    }],
    "route_points": [
      {
        "point_id": 1,
        "visit_order": 1,
        "type": "source",
        "contact": {"name": "Отправитель", "phone": "+79001234567"},
        "address": {
          "fullname": "Москва, ул. Тверская, 1",
          "coordinates": [37.6, 55.76]
        }
      },
      {
        "point_id": 2,
        "visit_order": 2,
        "type": "destination",
        "contact": {"name": "Получатель", "phone": "+79007654321"},
        "address": {
          "fullname": "Москва, ул. Арбат, 10",
          "coordinates": [37.59, 55.75]
        }
      }
    ]
  }'
```

### Проверить статус

```bash
curl -X POST "https://b2b.taxi.yandex.net/b2b/cargo/integration/v2/claims/info" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept-Language: ru" \
  -d '{"claim_id": "abc123"}'
```

## Статусы заказа

### Жизненный цикл

```
new → estimating → ready_for_approval → accepted → performer_lookup 
    → performer_found → pickup_arrived → pickuped 
    → delivery_arrived → delivered → delivered_finish
```

### Все статусы

| Статус | Описание | Действие |
|--------|----------|----------|
| `new` | Создана | — |
| `estimating` | Оценивается | Ждать |
| `ready_for_approval` | Готова к подтверждению | Вызвать `/claims/accept` в течение 10 мин |
| `accepted` | Подтверждена | — |
| `performer_lookup` | Поиск курьера | — |
| `performer_draft` | Поиск в процессе | — |
| `performer_found` | Курьер найден | — |
| `pickup_arrived` | Курьер на точке забора | — |
| `ready_for_pickup_confirmation` | Ждёт код забора | — |
| `pickuped` | Забрано | — |
| `delivery_arrived` | Курьер на точке доставки | — |
| `ready_for_delivery_confirmation` | Ждёт код доставки | — |
| `pay_waiting` | Ждёт оплату (наложенный платёж) | — |
| `delivered` | Доставлено | — |
| `delivered_finish` | Завершено | — |

### Возврат

| Статус | Описание |
|--------|----------|
| `returning` | Возвращается |
| `return_arrived` | Курьер на точке возврата |
| `ready_for_return_confirmation` | Ждёт код возврата |
| `returned` | Возвращено |
| `returned_finish` | Завершено с возвратом |

### Отмена

| Статус | Описание |
|--------|----------|
| `cancelled` | Отменено бесплатно |
| `cancelled_with_payment` | Отменено с оплатой |
| `cancelled_with_items_on_hands` | Отменено, товар у курьера |
| `cancelled_by_taxi` | Отменено курьером |

### Ошибки

| Статус | Описание |
|--------|----------|
| `failed` | Ошибка выполнения |
| `estimating_failed` | Ошибка оценки |
| `performer_not_found` | Курьер не найден |

## Enum-значения

### taxi_class (тариф)

| Значение | Описание | Макс. вес | Макс. габариты |
|----------|----------|-----------|----------------|
| `courier` | Пеший/вело курьер | 10 кг | 0.5×0.5×0.5 м |
| `express` | Авто курьер | 20 кг | Багажник легкового авто |
| `cargo` | Грузовой | 300 кг | Зависит от `cargo_type` |

### cargo_type (тип грузового)

| Значение | Описание | Грузоподъёмность | Объём |
|----------|----------|------------------|-------|
| `van` | Каблук | 300 кг | 1 м³ |
| `lcv_m` | Средний фургон | 700 кг | 3 м³ |
| `lcv_l` | Большой фургон | 1400 кг | 6 м³ |

### cargo_options (опции)

| Значение | Описание |
|----------|----------|
| `thermobag` | Термосумка |
| `auto_courier` | Только авто |
| `door_to_door` | От двери до двери |

### vat_code_str (НДС для фискализации)

| Значение | Описание |
|----------|----------|
| `vat_none` | Без НДС |
| `vat_0` | НДС 0% |
| `vat_10` | НДС 10% |
| `vat_20` | НДС 20% |
| `vat_10_110` | НДС 10/110 |
| `vat_20_120` | НДС 20/120 |

### payment_method (способ оплаты)

| Значение | Описание |
|----------|----------|
| `card` | Картой онлайн |
| `cash` | Наличными курьеру |
| `already_paid` | Уже оплачено |

---

# Other Day Delivery API

Доставка на следующий день или в выбранный интервал.

## Base URL

```
https://b2b-authproxy.taxi.yandex.net
```

## Эндпоинты

| Метод | Endpoint | Описание |
|-------|----------|----------|
| POST | `/api/b2b/platform/offers/info` | Доступные интервалы доставки |
| POST | `/api/b2b/platform/offers/create` | Создать оффер (зарезервировать слот) |
| POST | `/api/b2b/platform/offers/confirm` | Подтвердить оффер |
| POST | `/api/b2b/platform/offers/cancel` | Отменить оффер |
| POST | `/api/b2b/platform/request/info` | Статус заявки |
| POST | `/api/b2b/platform/request/cancel` | Отменить заявку |
| GET | `/api/b2b/platform/request/history` | История заявок |

## Пример: получить интервалы

```bash
curl -X POST "https://b2b-authproxy.taxi.yandex.net/api/b2b/platform/offers/info" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "info": {
      "operator_request_id": "my-order-123"
    },
    "source": {
      "platform_station": {
        "platform_id": "fbed3aa1-2cc6-4370-ab4d-59c5cc9bb924"
      }
    },
    "destination": {
      "type": "custom_location",
      "custom_location": {
        "coordinates": {"lat": 55.7558, "lon": 37.6173},
        "details": {"full_address": "Москва, ул. Тверская, 1"}
      }
    },
    "items": [{
      "count": 1,
      "name": "Товар",
      "article": "SKU-001",
      "billing_details": {"unit_price": 1000, "assessed_unit_price": 1000}
    }],
    "last_mile_policy": "self_pickup",
    "recipient_info": {
      "phone": "+79001234567",
      "first_name": "Иван"
    }
  }'
```

Ответ:
```json
{
  "offers": [
    {
      "offer_id": "offer-uuid",
      "delivery_interval": {
        "from": "2024-01-15T10:00:00+03:00",
        "to": "2024-01-15T14:00:00+03:00"
      },
      "price": {"total": "350.00", "currency": "RUB"}
    }
  ]
}
```

## Отличия от Express API

| Параметр | Express | Other Day |
|----------|---------|-----------|
| Base URL | b2b.taxi.yandex.net | b2b-authproxy.taxi.yandex.net |
| Время доставки | 30-90 мин | Выбранный интервал |
| Склад | Координаты в запросе | `platform_id` (заранее создан) |
| Создание | Сразу `/claims/create` | `/offers/info` → `/offers/create` → `/offers/confirm` |

---

# Callbacks (Webhooks)

## Настройка

Личный кабинет → Интеграции → Webhook URL

## Формат

```http
POST /your-webhook-endpoint
Content-Type: application/json
```

```json
{
  "claim_id": "abc123",
  "status": "performer_found",
  "updated_ts": "2024-01-15T12:00:00+03:00",
  "performer_info": {
    "courier_name": "Иван",
    "legal_name": "ООО Курьер",
    "car_model": "Lada Granta",
    "car_number": "А123БВ777"
  }
}
```

## Подтверждение

Ответить `200 OK` в течение 5 секунд. При ошибке — retry с экспоненциальным backoff.

---

# Наложенный платёж (COD)

## Параметры в items

```json
{
  "items": [{
    "title": "Товар",
    "cost_value": "1000",
    "cost_currency": "RUB",
    "fiscalization": {
      "article": "SKU-001",
      "vat_code_str": "vat_20",
      "supplier_inn": "7707083893"
    }
  }]
}
```

## Параметры в route_points

```json
{
  "type": "destination",
  "payment_on_delivery": {
    "payment_method": "cash",
    "customer": {
      "phone": "+79001234567"
    }
  }
}
```

## Статусы оплаты

| Статус | Описание |
|--------|----------|
| `pay_waiting` | Ожидание оплаты |
| `paid` | Оплачено |
| `payment_failed` | Ошибка оплаты |

---

# Фискализация

## fiscal_receipt_info

```json
{
  "fiscal_receipt_info": {
    "vat_code_str": "vat_20",
    "personal_tin": "123456789012",
    "title_for_receipt": "Доставка",
    "supplier_inn": "7707083893"
  }
}
```

## Поля

| Поле | Обязательное | Описание |
|------|--------------|----------|
| `vat_code_str` | Да | Код НДС |
| `personal_tin` | Нет | ИНН получателя (для B2B) |
| `title_for_receipt` | Нет | Название в чеке |
| `supplier_inn` | Нет | ИНН поставщика (для агентской схемы) |

---

# Мультиточечная доставка

## Типы точек

| Тип | Описание |
|-----|----------|
| `source` | Точка забора |
| `destination` | Точка доставки |
| `return` | Точка возврата |

## Пример: 1 забор → 3 доставки

```json
{
  "route_points": [
    {"point_id": 1, "visit_order": 1, "type": "source", ...},
    {"point_id": 2, "visit_order": 2, "type": "destination", ...},
    {"point_id": 3, "visit_order": 3, "type": "destination", ...},
    {"point_id": 4, "visit_order": 4, "type": "destination", ...}
  ],
  "items": [
    {"pickup_point": 1, "droppof_point": 2, ...},
    {"pickup_point": 1, "droppof_point": 3, ...},
    {"pickup_point": 1, "droppof_point": 4, ...}
  ]
}
```

## Ограничения

- Максимум 10 точек
- Все точки в одном городе
- Порядок посещения фиксирован (`visit_order`)

---

# Коды ошибок

## HTTP статусы

| Код | Описание |
|-----|----------|
| 400 | Невалидный запрос |
| 401 | Невалидный токен |
| 403 | Нет доступа |
| 404 | Заявка не найдена |
| 409 | Конфликт (дубликат request_id) |
| 429 | Rate limit |
| 500 | Внутренняя ошибка |

## Бизнес-ошибки

| Код | Описание |
|-----|----------|
| `estimating_failed` | Не удалось рассчитать маршрут |
| `performer_not_found` | Нет доступных курьеров |
| `address_not_found` | Адрес не найден |
| `out_of_zone` | Адрес вне зоны доставки |
| `too_heavy` | Превышен вес |
| `too_large` | Превышены габариты |

## Формат ошибки

```json
{
  "code": "estimating_failed",
  "message": "Не удалось построить маршрут между точками"
}
```

---

## Технические требования

- TLS 1.2+
- Координаты: `[longitude, latitude]` — долгота первая!
- `request_id` — уникальный UUID для идемпотентности
- Геокодирование: используй Yandex Geocoder API
- Таймаут: рекомендуется 30 секунд
- Retry: экспоненциальный backoff, максимум 3 попытки

---

# Яндекс Еда — Ресторанам

## Способы интеграции

| Способ | Описание | Автоматизация |
|--------|----------|---------------|
| Vendor Portal | Веб-интерфейс | Ручной |
| Мобильное приложение | "Яндекс Еда Вендор" | Ручной |
| POS-интеграция | iiko, r_keeper, Poster | Автоматический |

## Vendor Portal

URL: `https://vendor.yandex.ru/`

### Разделы

| URL | Раздел |
|-----|--------|
| `/orders` | Заказы |
| `/menu` | Меню |
| `/places` | Рестораны |
| `/schedule` | Расписание |
| `/shipping-zone` | Зоны доставки |
| `/promotion` | Продвижение |
| `/chats` | Поддержка |

### Авторизация

- Вход через Yandex ID
- Логин/пароль от менеджера Яндекс Еды
- Роль "Управляющий" для полного доступа

## Обработка заказов

### Поток

```
Новый заказ (звук + жёлтая подсветка)
    ↓
"Принять заказ" (проверить состав)
    ↓
"Заказ готов" (упаковано)
    ↓
"Заказ передан" (курьер Яндекса забрал)
    или
"Доставлено" (свой курьер доставил)
```

### Структура заказа

```json
{
  "order_number": "string",
  "items": [
    {
      "name": "string",
      "price": 100,
      "weight": 250,
      "quantity": 1
    }
  ],
  "total_amount": 100,
  "cutlery_count": 2,
  "courier": {
    "name": "Иван",
    "phone": "+79001234567"
  },
  "customer_comment": "без лука",
  "delivery_address": "ул. Пушкина, 1 (только для своих курьеров)",
  "customer_phone": "+79007654321 (только для своих курьеров)",
  "change_from": 1000
}
```

## Меню

### Структура

```
Категории (1-10, макс 3 кастомных)
└── Позиции
    ├── Название (≤5 слов)
    ├── Описание
    ├── Вес/объём
    ├── Цена
    ├── НДС
    ├── КБЖУ
    ├── Фото (опционально, +30% заказов)
    └── Опции (платные/бесплатные)
```

### Акцизные товары

Если POS поддерживает флаг акциза — установить в карточке.
Если нет — добавить `[AT]` к названию:

```
Вино красное [AT]
```

### Стоп-лист

Vendor Portal → Меню → Стоп-лист → Выбрать позиции → Указать период

## POS-интеграции

### iiko

API: `https://api-ru.iiko.services/docs`

Ключевые методы:
```
POST /api/1/access_token          # Авторизация
POST /api/1/delivery/create       # Создать заказ
GET  /api/1/delivery/by_id        # Статус заказа
POST /api/1/stop_lists            # Стоп-лист
GET  /api/1/nomenclature          # Меню
```

### r_keeper

Интеграция через модуль r_keeper Delivery. Документация по запросу.

### Poster

Интеграция через Poster Marketplace. Документация по запросу.

## Контакты

| Канал | Данные |
|-------|--------|
| Телефон | 8 (800) 600-13-10 (Пн-Вс 9:00-21:00 МСК) |
| Email | rest@eda.yandex.ru |
| Чат | vendor.yandex.ru/chats |

---

# Яндекс Еда — Клиентский API

## Статус

**Публичного API нет.** Используется внутренний API для мобильных приложений и сайта.

## Известные эндпоинты

Получены через анализ сетевого трафика:

| Endpoint | Метод | Описание |
|----------|-------|----------|
| `/api/v2/catalog` | GET | Список ресторанов |
| `/api/v2/catalog/{slug}/menu` | GET | Меню ресторана |
| `/eats/v1/layout-constructor/v1/layout` | POST | Главная страница |
| `/eats/v1/cart/v2/multi-carts` | POST | Корзина |
| `/eats/v1/launch/v2/native` | POST | Инициализация приложения |

### Пример: список ресторанов

```bash
curl "https://eda.yandex.ru/api/v2/catalog?latitude=55.7558&longitude=37.6173" \
  -H "User-Agent: Mozilla/5.0 ..."
```

Ответ:
```json
{
  "payload": {
    "foundPlaces": [
      {
        "place": {
          "id": 123,
          "name": "Ресторан",
          "slug": "restaurant-slug",
          "rating": 4.8,
          "minimalDeliveryCost": 0
        }
      }
    ]
  }
}
```

### Пример: меню ресторана

```bash
curl "https://eda.yandex.ru/api/v2/catalog/restaurant-slug/menu?latitude=55.7558&longitude=37.6173" \
  -H "User-Agent: Mozilla/5.0 ..."
```

Ответ:
```json
{
  "payload": {
    "categories": [
      {
        "name": "Пицца",
        "items": [
          {
            "name": "Маргарита",
            "description": "Томаты, моцарелла, базилик",
            "price": 500,
            "weight": "450 г"
          }
        ]
      }
    ]
  }
}
```

## Защита

- SmartCaptcha — блокирует автоматические запросы
- Rate limiting — агрессивный
- User-Agent — обязателен
- Cookies — нужны для авторизованных запросов

## Reverse Engineering

Для анализа API:

1. **DevTools** — Network tab на eda.yandex.ru
2. **mitmproxy** — перехват трафика мобильного приложения
3. **Frida** — runtime-анализ приложения

⚠️ Использование недокументированного API может нарушать ToS Яндекса.

---

## Ссылки

### Яндекс Доставка
- [Документация API](https://yandex.ru/support2/delivery-profile/ru/api/)
- [Портал разработчика](https://yandex.ru/dev/logistics/)
- [Личный кабинет](https://dostavka.yandex.ru/account)

### Яндекс Еда
- [Vendor Portal](https://vendor.yandex.ru/)
- [Документация для ресторанов](https://yandex.ru/support/eda-vendor-ru/)
- [iiko API](https://api-ru.iiko.services/docs)

### Готовые модули
- [iiko](https://yandex.ru/support2/delivery-profile/ru/modules/iiko)
- [r_keeper](https://yandex.ru/support2/delivery-profile/ru/modules/r-keeper)
- [Smartomato](https://yandex.ru/support2/delivery-profile/ru/modules/smartomato)
- [Quick Resto](https://yandex.ru/support2/delivery-profile/ru/modules/quick-resto)
