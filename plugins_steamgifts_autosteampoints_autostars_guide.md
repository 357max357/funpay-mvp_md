# Описание и тестирование плагинов `steamgifts_autolot`, `autosteampoints`, `autostars`

Ниже — описание трёх плагинов: для чего они нужны, что обязательно настроить, что вводить в настройках и как их тестировать.

По логам все три нужных сервиса подняты:
- `plugin_steamgifts_autolot`
- `plugin_autosteampoints`
- `plugin_autostars`
- отдельно стартует `autostars_provider_http`

---

## 1) `steamgifts_autolot`

### Для чего нужен
Этот плагин автоматизирует продажу Steam Gifts через внешнего HTTP-поставщика.

Что он умеет:
- принимает заказ по привязанному `lot_id`
- смотрит, к какому `ns_sku` этот лот привязан
- если от покупателя не хватает данных, запрашивает их в FunPay-чате
- может запросить:
  - ссылку на Steam-профиль
  - email
  - регион в формате `REGION: RU`
- отправляет заказ поставщику
- если поставщик ответил `DELIVERED`, отправляет покупателю данные
- если поставщик вернул `OUT_OF_STOCK`, может деактивировать лот и уведомить продавца
- умеет периодически синкать каталог поставщика: цены, наличие, доступные регионы

### Что обязательно настроить
Минимум:
1. `Provider kind = http`
2. `HTTP provider`
3. хотя бы один `Лот`

В меню это по смыслу:
- `Provider`
- `HTTP`
- `Лот`

### Что вводить
Пример для `Provider`:

```text
http
```

Пример для `HTTP provider`:

```text
http://host.docker.internal:9003
TESTTOKEN
25
```

Где:
- `base_url = http://host.docker.internal:9003`
- `token = TESTTOKEN`
- `timeout_sec = 25`

Пример для `Лот`:

```text
12345678
demo_game_1
RU,KZ
12
```

Где:
- `12345678` — `lot_id` на FunPay
- `demo_game_1` — `ns_sku` у поставщика
- `RU,KZ` — разрешённые регионы
- `12` — markup в процентах

### Как проверять
Есть 3 нормальных сценария.

### Сценарий A — проверка запроса данных у покупателя
Сделай тестовый заказ по привязанному лоту, но не пиши в заказе Steam-профиль и регион.

Ожидаемое поведение:
- плагин возьмёт заказ
- в FunPay-чат отправит запрос контактных данных
- если для лота указан `allowed_regions`, он может отдельно запросить регион

Для ответа покупателя подойдёт одно сообщение, например:

```text
https://steamcommunity.com/id/testplayer
REGION: RU
```

### Сценарий B — успешная доставка
Для этого нужен тестовый HTTP-провайдер, который умеет отвечать так.

`GET /catalog`

```json
[
  {
    "sku": "demo_game_1",
    "title": "Demo Game",
    "price": 100,
    "currency": "RUB",
    "stock": 5,
    "allowed_regions": ["RU", "KZ"]
  }
]
```

`POST /orders`

```json
{
  "order_id": "sg_1001",
  "status": "DELIVERED",
  "delivered": {
    "gift_link": "https://example.com/gift/abc123",
    "instructions": "Открой ссылку и прими подарок"
  }
}
```

`GET /orders/sg_1001`

```json
{
  "order_id": "sg_1001",
  "status": "DELIVERED",
  "delivered": {
    "gift_link": "https://example.com/gift/abc123",
    "instructions": "Открой ссылку и прими подарок"
  }
}
```

Ожидаемое поведение:
- заказ переходит в обработку
- покупатель получает сообщение с данными подарка
- если включён `autocomplete_order`, заказ закрывается автоматически

### Сценарий C — нет в наличии
Провайдер отвечает так:

```json
{
  "order_id": "sg_1002",
  "status": "OUT_OF_STOCK"
}
```

Ожидаемое:
- покупателю уходит сообщение, что товара нет в наличии
- продавцу уходит уведомление
- если включено `auto_disable_out_of_stock`, лот деактивируется

### Что смотреть в логах

```powershell
docker compose logs -f plugin_steamgifts_autolot outbox_dispatcher
```

Что ищешь:
- запрос контактных данных
- создание provider order
- `OUT_OF_STOCK`
- `DELIVERED`
- постановку buyer/seller сообщений в outbox

---

## 2) `autosteampoints`

### Для чего нужен
Этот плагин продаёт Steam Community Points через внешнего HTTP-поставщика.

Что он делает:
- ловит заказ по нужному `lot_id`
- вычисляет, сколько points нужно выдать
- если Steam-профиль не указан — просит покупателя прислать профиль
- пытается валидировать профиль
- просит подтверждение профиля ответом `Да/Нет`
- после подтверждения создаёт заказ у поставщика
- если поставщик выдал points — отправляет покупателю сообщение об успешной выдаче
- если `OUT_OF_STOCK` — ставит задачу в ожидание, уведомляет продавца и может деактивировать лот
- есть `Replay`, чтобы потом повторно запустить зависшие `WAITING_STOCK`

### Важный момент
В текущем коде встроенный mock для `autosteampoints` отключён.

То есть для него нужен именно HTTP-провайдер.

### Что обязательно настроить
Минимум:
1. `HTTP provider`
2. `Лот`
3. желательно `SLA`

### Что вводить
Пример для `Provider`:

```text
http://host.docker.internal:9002
TESTTOKEN
25
```

Пример для `Лот`:

```text
23456789
1000
```

Где:
- `23456789` — `lot_id`
- `1000` — сколько points выдавать за 1 единицу товара

Пример для `SLA`:

```text
120
```

### Как проверять

### Сценарий A — проверка запроса профиля
Сделай заказ по этому лоту, не указывая профиль в заказе.

Ожидаемое:
- плагин попросит прислать Steam-профиль
- после ссылки попросит подтверждение
- ответ `Да` — пойдёт дальше
- ответ `Нет` — попросит прислать другой профиль

Пример сообщения покупателя:

```text
https://steamcommunity.com/id/testplayer
```

Потом:

```text
Да
```

### Сценарий B — успешная выдача
Тестовый HTTP-провайдер должен уметь:

`GET /balance`

```json
{
  "available_points": 50000
}
```

`POST /orders`

```json
{
  "order_id": "sp_1001",
  "status": "COMPLETED"
}
```

`GET /orders/sp_1001`

```json
{
  "order_id": "sp_1001",
  "status": "COMPLETED"
}
```

Ожидаемое:
- после подтверждения профиля уходит сообщение `processing`
- потом покупателю уходит `delivered`

### Сценарий C — нехватка стока
Провайдер может вернуть:

через `/balance`:

```json
{
  "available_points": 0
}
```

или через `/orders`:

```json
{
  "order_id": "sp_1002",
  "status": "OUT_OF_STOCK",
  "message": "no stock"
}
```

Ожидаемое:
- job уйдёт в `WAITING_STOCK`
- покупателю придёт сообщение о временном отсутствии стока
- продавцу придёт уведомление
- лот может деактивироваться

Потом:
- пополняешь сток на провайдере
- жмёшь `Replay`
- задача снова переводится в `NEW`

### Что смотреть в логах

```powershell
docker compose logs -f plugin_autosteampoints outbox_dispatcher
```

Ищи:
- запрос профиля
- подтверждение профиля
- `processing`
- `WAITING_STOCK`
- `COMPLETED`
- seller notify

---

## 3) `autostars`

### Для чего нужен
Этот плагин автоматизирует выдачу Telegram Stars.

Как он работает:
- ловит заказ по настроенному `lot_id`
- считает, сколько Stars надо отправить
- если в заказе нет `@username`, просит покупателя прислать его в чат
- после этого вызывает HTTP source
- source проверяет баланс
- создаёт transfer
- дальше плагин ждёт подтверждения через `GET /transfer/{tx_id}`
- после `CONFIRMED` отправляет buyer-сообщение об успешной доставке
- может:
  - ждать баланс (`WAITING_BALANCE`)
  - ждать дневной лимит (`WAITING_LIMIT`)
  - ждать подтверждение провайдера (`WAITING_CONFIRM`)

### Что обязательно настроить
Минимум:
1. `Лот`
2. `Source` — либо env default, либо отдельный HTTP source

`autostars_provider_http` уже стартует в инфраструктуре.

### Что вводить

### Вариант A — использовать default source из env
Тогда настраиваешь только лот:

```text
34567890
50
default
```

Где:
- `34567890` — `lot_id`
- `50` — `stars_per_unit`
- `default` — source из `.env`

### Вариант B — добавить отдельный source
Пример для `Source`:

```text
stars_test
http://autostars_provider_http:8080
TESTTOKEN
10000
500
20
200
```

Где:
1. `source_id = stars_test`
2. `base_url = http://autostars_provider_http:8080`
3. `token = TESTTOKEN`
4. `daily_limit_stars = 10000`
5. `max_stars_per_tx = 500`
6. `timeout_sec = 20`
7. `min_balance_alert = 200`

Потом лот:

```text
34567890
50
stars_test
```

### Как проверять

### Сценарий A — заказ без username
Сделай тестовый заказ по нужному лоту, без `@username` в описании.

Ожидаемое:
- плагин переведёт job в `WAITING_BUYER`
- в FunPay-чат уйдёт сообщение с просьбой прислать `@username`

Потом покупатель пишет:

```text
@my_test_user
```

После этого, если заказ уже `PAID`, плагин пойдёт в выдачу.

### Сценарий B — заказ с username сразу
Если в описании заказа уже есть:

```text
@my_test_user
```

то шаг с запросом username будет пропущен.

### Сценарий C — успешная выдача
У `autostars` тестирование двухуровневое.

#### Уровень 1 — сам plugin
Он ходит в:
- `GET /balance`
- `POST /transfer`
- `GET /transfer/{tx_id}`

#### Уровень 2 — `autostars_provider_http`
Этот сервис уже внутри себя дёргает внешний webhook-интегратор.

Внешний webhook должен вернуть что-то вроде:

успех:

```json
{
  "status": "CONFIRMED",
  "external_tx_id": "stars_demo_1",
  "delivered_ts": "2026-04-17T12:00:00+00:00"
}
```

ожидание:

```json
{
  "status": "PENDING"
}
```

ошибка:

```json
{
  "status": "FAILED",
  "error": "insufficient_balance"
}
```

### Как быстро проверить `autostars_provider_http`
Сначала баланс:

```powershell
curl -H "Authorization: Bearer TESTTOKEN" http://localhost:8080/balance
```

Ожидаемый ответ:

```json
{"balance":10000}
```

Потом тестовый transfer:

```powershell
curl -X POST "http://localhost:8080/transfer" `
  -H "Authorization: Bearer TESTTOKEN" `
  -H "Content-Type: application/json" `
  -H "Idempotency-Key: demo-1" `
  -d "{\"to_username\":\"@my_test_user\",\"stars\":50,\"idempotency_key\":\"demo-1\"}"
```

Ожидаемо:
- либо сразу `PENDING`
- либо `FAILED`, если webhook/backend не настроен корректно

### Сценарий D — недостаток баланса
Если balance меньше, чем нужно Stars:
- job уйдёт в `WAITING_BALANCE`
- продавцу уйдёт уведомление о нехватке баланса

### Сценарий E — дневной лимит
Если превысить `daily_limit_stars`:
- job уйдёт в `WAITING_LIMIT`
- продавцу уйдёт уведомление

### Что смотреть в логах

```powershell
docker compose logs -f plugin_autostars autostars_provider_http outbox_dispatcher
```

Ищи:
- запрос username
- balance check
- transfer create
- `WAITING_CONFIRM`
- `CONFIRMED`
- `FAILED`
- seller notify

---

## Как лучше тестировать по порядку

### Сначала `autostars`
Потому что у него самая прозрачная цепочка:
- лот
- source
- username
- transfer
- confirm

### Потом `autosteampoints`
Потому что там уже есть ветка:
- запрос профиля
- подтверждение
- stock / replay

### Потом `steamgifts_autolot`
Потому что он самый сложный:
- контактные данные
- регион
- каталог
- цены
- out-of-stock
- доставочный payload

---

## Самый короткий практический план

### `autostars`
1. Добавить source
2. Добавить лот
3. Купить лот
4. Прислать `@username`
5. Смотреть `plugin_autostars + autostars_provider_http + outbox_dispatcher`

### `autosteampoints`
1. Настроить HTTP provider
2. Добавить лот
3. Купить лот
4. Прислать Steam-профиль
5. Ответить `Да`
6. Смотреть `plugin_autosteampoints + outbox_dispatcher`

### `steamgifts_autolot`
1. Настроить HTTP provider
2. Добавить лот с `ns_sku`
3. Купить лот
4. Прислать профиль и `REGION: RU`
5. Смотреть `plugin_steamgifts_autolot + outbox_dispatcher`
