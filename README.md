# Yandex Delivery API

Неофициальная документация по API Яндекс Доставки.

> **Важно:** Отдельного публичного API "Яндекс.Еда" не существует. Интеграция с доставкой еды идёт через Yandex Delivery API.

## Содержание

- [Обзор](#обзор)
- [Авторизация](#авторизация)
- [Базовые URL](#базовые-url)
- [Express Delivery API](#express-delivery-api)
- [Other Day Delivery API](#other-day-delivery-api)
- [Статусы заказов](#статусы-заказов)
- [Примеры](#примеры)

## Обзор

Yandex Delivery (Яндекс Доставка) — часть экосистемы Yandex Go. API позволяет автоматизировать создание заказов на доставку, отслеживание и управление.

**Официальная документация:**
- https://yandex.ru/support2/delivery-profile/ru/api/
- https://yandex.ru/dev/logistics/

**Типы доставки:**

| Тип | Описание | Время |
|-----|----------|-------|
| Express | Срочная городская доставка | В течение часов |
| Same-day | Доставка в тот же день | До конца дня |
| Other-day | Плановая доставка | На следующий день и позже |

## Авторизация

```
Authorization: Bearer <OAuth-token>
```

**Получение токена:**
1. Войти в [личный кабинет](https://dostavka.yandex.ru/auth/)
2. Вкладка **Интеграции**
3. **Активировать интеграцию** → **Получить токен**

Токен бессрочный.

## Базовые URL

| Среда | URL |
|-------|-----|
| Production | `https://b2b.taxi.yandex.net` |
| Test | `https://b2b.taxi.tst.yandex.net` |

**Тестовые данные:** см. [официальную документацию](https://yandex.ru/support2/delivery-profile/ru/api/express/quickstart)

## Express Delivery API

### Основные методы

| Метод | Endpoint | Описание |
|-------|----------|----------|
| POST | `/b2b/cargo/integration/v2/offers/calculate` | Варианты доставки |
| POST | `/b2b/cargo/integration/v2/claims/create` | Создать заявку |
| POST | `/b2b/cargo/integration/v2/claims/info` | Информация о заявке |
| POST | `/b2b/cargo/integration/v2/claims/accept` | Подтвердить заявку |
| POST | `/b2b/cargo/integration/v2/claims/cancel` | Отменить заявку |

### Расчёт цены

| Метод | Endpoint | Описание |
|-------|----------|----------|
| POST | `/b2b/cargo/integration/v2/check-price` | Предварительная цена |
| POST | `/b2b/cargo/integration/v2/tariffs` | Доступные тарифы |

### Отслеживание

| Метод | Endpoint | Описание |
|-------|----------|----------|
| POST | `/b2b/cargo/integration/v2/driver-voiceforwarding` | Телефон курьера |
| GET | `/b2b/cargo/integration/v2/claims/performer-position` | Координаты курьера |
| POST | `/b2b/cargo/integration/v2/claims/points-eta` | ETA по точкам |
| GET | `/b2b/cargo/integration/v2/claims/tracking-links` | Ссылки отслеживания |

### Подтверждение

| Метод | Endpoint | Описание |
|-------|----------|----------|
| POST | `/b2b/cargo/integration/v2/claims/confirmation_code` | Код подтверждения |
| POST | `/b2b/cargo/integration/v2/claims/proof-of-delivery/info` | Подтверждение доставки |

### Редактирование

| Метод | Endpoint | Описание |
|-------|----------|----------|
| POST | `/b2b/cargo/integration/v2/claims/edit` | Редактировать (до подтверждения) |
| POST | `/b2b/cargo/integration/v2/claims/apply-changes/request` | Частичное редактирование |
| POST | `/b2b/cargo/integration/v2/claims/apply-changes/result` | Результат редактирования |
| POST | `/b2b/cargo/integration/v2/claims/return` | Инициировать возврат |

### Массовые операции

| Метод | Endpoint | Описание |
|-------|----------|----------|
| POST | `/b2b/cargo/integration/v2/claims/search` | Поиск заявок |
| POST | `/b2b/cargo/integration/v2/claims/bulk_info` | Инфо по нескольким заявкам |
| POST | `/b2b/cargo/integration/v2/claims/journal` | История изменений |

## Other Day Delivery API

Base URL: `https://b2b-authproxy.taxi.yandex.net`

Для плановой доставки в другой день. Документация: https://yandex.ru/support2/delivery-profile/ru/api/other-day/index

## Статусы заказов

### Основные

| Статус | Описание |
|--------|----------|
| `new` | Создана |
| `estimating` | Оценивается |
| `ready_for_approval` | Ожидает подтверждения (10 мин) |
| `accepted` | Подтверждена |
| `performer_lookup` | Поиск курьера |
| `performer_found` | Курьер найден |
| `pickup_arrived` | Курьер на точке забора |
| `pickuped` | Забрано |
| `delivery_arrived` | Курьер на точке доставки |
| `delivered` | Доставлено |
| `delivered_finish` | Завершено |

### Возврат

| Статус | Описание |
|--------|----------|
| `returning` | Возвращается |
| `return_arrived` | Курьер на точке возврата |
| `returned` | Возвращено |
| `returned_finish` | Завершено с возвратом |

### Отмена

| Статус | Описание |
|--------|----------|
| `cancelled` | Отменено бесплатно |
| `cancelled_with_payment` | Отменено с оплатой |
| `cancelled_by_taxi` | Отменено курьером |

### Ошибки

| Статус | Описание |
|--------|----------|
| `failed` | Ошибка выполнения |
| `estimating_failed` | Ошибка оценки |
| `performer_not_found` | Курьер не найден |

## Примеры

### Создание заявки

```bash
curl -X POST https://b2b.taxi.yandex.net/b2b/cargo/integration/v2/claims/create \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "items": [{
      "title": "Документы",
      "quantity": 1,
      "cost_value": "0",
      "cost_currency": "RUB"
    }],
    "route_points": [
      {
        "point_id": 1,
        "visit_order": 1,
        "type": "source",
        "contact": {"name": "Отправитель", "phone": "+79001234567"},
        "address": {"fullname": "Москва, ул. Тверская, 1", "coordinates": [37.6, 55.76]}
      },
      {
        "point_id": 2,
        "visit_order": 2,
        "type": "destination",
        "contact": {"name": "Получатель", "phone": "+79007654321"},
        "address": {"fullname": "Москва, ул. Арбат, 10", "coordinates": [37.59, 55.75]}
      }
    ]
  }'
```

### Проверка статуса

```bash
curl -X POST https://b2b.taxi.yandex.net/b2b/cargo/integration/v2/claims/info \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"claim_id": "abc123"}'
```

### Расчёт стоимости

```bash
curl -X POST https://b2b.taxi.yandex.net/b2b/cargo/integration/v2/check-price \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "items": [{"quantity": 1}],
    "route_points": [
      {"coordinates": [37.6, 55.76]},
      {"coordinates": [37.59, 55.75]}
    ]
  }'
```

## Технические требования

- TLS 1.2+
- Координаты: `[longitude, latitude]` (долгота, широта)
- Для адресов используй Yandex Geocoder API

## Готовые интеграции

Модули для POS-систем:
- [iiko](https://yandex.ru/support2/delivery-profile/ru/modules/iiko)
- [r_keeper](https://yandex.ru/support2/delivery-profile/ru/modules/r-keeper)
- [Smartomato](https://yandex.ru/support2/delivery-profile/ru/modules/smartomato)
- [Quick Resto](https://yandex.ru/support2/delivery-profile/ru/modules/quick-resto)

## Ссылки

- [Документация API](https://yandex.ru/support2/delivery-profile/ru/api/)
- [Портал разработчика](https://yandex.ru/dev/logistics/)
- [Личный кабинет](https://dostavka.yandex.ru/account)
- [Регистрация бизнеса](https://dostavka.yandex.ru/reg)
- [Yandex Routing API](https://yandex.ru/routing/doc/ru/vrp/)

## Лицензия

MIT
