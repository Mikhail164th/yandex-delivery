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

## Технические требования

- TLS 1.2+
- Координаты: `[longitude, latitude]` — долгота первая!
- `request_id` — уникальный UUID для идемпотентности
- Геокодирование: используй Yandex Geocoder API

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
