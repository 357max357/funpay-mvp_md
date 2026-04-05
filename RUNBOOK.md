# FunPay MVP: запуск и использование

## 1. Что это за проект

Это набор микросервисов для автоматизации магазина на FunPay.

Базовый поток такой:

1. `funpay_connector` слушает события аккаунта FunPay.
2. `plugin_router` раскладывает события по включенным плагинам.
3. Каждый плагин обрабатывает свои заказы и кладет действия в `outbox`.
4. `outbox_dispatcher` выполняет действия:
   - отправляет сообщения в чат FunPay;
   - меняет статус заказа;
   - включает/выключает лоты;
   - шлет уведомления в Telegram.
5. `control_bot` управляет аккаунтом FunPay, подписками и конфигами плагинов.
6. `entitlement_enforcer` выключает плагины, если истек доступ.
7. `postgres` хранит конфиги, заказы, state и очереди.
8. `rabbitmq` передает события между сервисами.


## 2. Что реально работает в текущем состоянии репозитория

### Рекомендуемый способ запуска

Используй `infra/docker-compose.prod.yml`.

Именно этот compose соответствует реальному набору зарегистрированных плагинов и не пытается поднимать заведомо лишние контейнеры из старой схемы.

### Плагины, которые реально запускаются через `docker-compose.prod.yml`

- `autostars`
- `autosteampoints`
- `steamgifts_autolot`
- `autorent_steam`
- `autorefund`
- `fragment_stars_buyer`
- `autolog`
- `autopricemonitor`

### Плагины, код которых есть, но в `docker-compose.prod.yml` они не добавлены

- `autobalance`
- `autolotdeactivator`

Их команды в боте есть, логика в `plugin_service` есть, но по умолчанию эти два контейнера в `prod`-compose не стартуют. Если просто включить их через бота, события не будут обрабатываться, пока ты не добавишь соответствующие сервисы в compose.

### Плагины, которые есть в каталоге бота, но не подключены как рабочие сервисы

- `autoreply`
- `ask_tg_username`
- `antifraud`
- `autodelivery`
- `autoreview`
- `priceguard`
- `dailysummary`
- `blacklist`
- `unpaidreminder`

В `control_bot` у них есть карточки в витрине, но в `plugin_service.plugins.registry` для них нет handler/ticker, поэтому как полноценные плагины они сейчас не используются.


## 3. Сервисы и их роль

### Базовые сервисы

- `postgres` — база проекта.
- `rabbitmq` — брокер событий.
- `funpay_connector` — слушает аккаунт FunPay.
- `plugin_router` — маршрутизирует события в плагины.
- `outbox_dispatcher` — исполняет отложенные действия.
- `control_bot` — Telegram-бот управления.
- `entitlement_enforcer` — выключает плагины после окончания доступа.

### Внешние/провайдерные сервисы

- `autostars_provider_http` — HTTP-провайдер для AutoStars.

### Контейнеры плагинов

Каждый плагин запускается отдельным контейнером `plugin_service` с переменной `PLUGIN_NAME`.


## 4. Что нужно для запуска

### Обязательное

- Docker
- Docker Compose
- Telegram-бот и его токен
- Действующий аккаунт FunPay
- Значение cookie `golden_key` от авторизованной сессии FunPay

### Для конкретных плагинов дополнительно

- AutoStars: HTTP/Webhook-провайдер Stars
- AutoSteamPoints: HTTP-провайдер Steam Points
- Steam Gifts Auto Lot: HTTP-провайдер NS.Gifts
- AutoRent Steam: пул Steam-аккаунтов
- Fragment Stars Buyer: HTTP-провайдер покупки Stars


## 5. Подготовка `.env`

В корне проекта уже есть шаблон `.env.example`.

Сделай:

```bash
cp .env.example .env
```

Заполни минимум:

- `TG_BOT_TOKEN`
- `CONTROL_ADMIN_IDS_JSON`
- `POSTGRES_PASSWORD`
- `RABBITMQ_PASS`
- `MASTER_KEY_B64`
- `PG_DSN`
- `AMQP_URL`

Очень важный момент: в `.env.example` пароли и DSN заданы отдельно. Если меняешь:

- `POSTGRES_PASSWORD`, обязательно синхронно поменяй пароль внутри `PG_DSN`
- `RABBITMQ_PASS`, обязательно синхронно поменяй пароль внутри `AMQP_URL`

Пример:

```env
POSTGRES_USER=funpay
POSTGRES_PASSWORD=strong_pg_password
POSTGRES_DB=funpay_mvp
PG_DSN=postgresql://funpay:strong_pg_password@postgres:5432/funpay_mvp

RABBITMQ_USER=funpay
RABBITMQ_PASS=strong_rabbit_password
AMQP_URL=amqp://funpay:strong_rabbit_password@rabbitmq:5672/
```

### Генерация `MASTER_KEY_B64`

```bash
python - <<'PY'
import os, base64
print(base64.b64encode(os.urandom(32)).decode())
PY
```

Этот ключ используется для шифрования:

- `golden_key`
- токенов провайдеров
- паролей Steam-аккаунтов

Если потом поменять `MASTER_KEY_B64`, старые зашифрованные данные перестанут читаться.


## 6. Переменные окружения по плагинам

### Общие

- `LOG_LEVEL`
- `APP_TIMEZONE`
- `DRY_RUN`
- `ENTITLEMENT_ENFORCER_INTERVAL_SEC`

`DRY_RUN` читает именно `outbox_dispatcher`, поэтому:

- `DRY_RUN=1` — действия логируются, но реально не отправляются в FunPay
- `DRY_RUN=0` — рабочий режим

### AutoStars

- `AUTOSTARS_PROVIDER=http`
- `AUTOSTARS_HTTP_BASE_URL=http://autostars_provider_http:8080`
- `AUTOSTARS_HTTP_TOKEN=...`
- `AUTOSTARS_PROVIDER_BACKEND=webhook`
- `AUTOSTARS_PROVIDER_WEBHOOK_URL=...`
- `AUTOSTARS_PROVIDER_WEBHOOK_TOKEN=...`

### Fragment Stars Buyer

- `FRAGMENT_BUYER_PROVIDER=http`
- `FRAGMENT_BUYER_HTTP_BASE_URL=...`
- `FRAGMENT_BUYER_HTTP_TOKEN=...`

### AutoSteamPoints

- `AUTOSTEAMPOINTS_PROVIDER_KIND=http`
- `AUTOSTEAMPOINTS_PROVIDER_BASE_URL=...`
- `AUTOSTEAMPOINTS_PROVIDER_TOKEN=...`

### SteamGifts

- `STEAMGIFTS_PROVIDER_KIND=http`
- `STEAMGIFTS_PROVIDER_BASE_URL=...`
- `STEAMGIFTS_PROVIDER_TOKEN=...`

### Платежи в Telegram

Если хочешь продавать плагины через Telegram invoice:

- `PAYMENTS_PROVIDER_TOKEN`
- `PAYMENTS_CURRENCY`

Если платежи не настроены, для тестов доступ можно выдавать командой `/grant`.


## 7. Запуск проекта

Из корня репозитория:

```bash
cd infra
docker compose -f docker-compose.prod.yml up -d --build
```

### Остановка

```bash
cd infra
docker compose -f docker-compose.prod.yml down
```

### Обновление после изменений

```bash
cd infra
docker compose -f docker-compose.prod.yml up -d --build
```


## 8. Что должно подняться

После старта проверь:

```bash
cd infra
docker compose -f docker-compose.prod.yml ps
```

Ожидаемые контейнеры:

- `funpay_mvp_postgres`
- `funpay_mvp_rabbitmq`
- `funpay_mvp_funpay_connector`
- `funpay_mvp_plugin_router`
- `funpay_mvp_entitlement_enforcer`
- `funpay_mvp_outbox_dispatcher`
- `funpay_mvp_control_bot`
- `funpay_mvp_autostars_provider_http`
- `funpay_mvp_plugin_autostars`
- `funpay_mvp_plugin_autosteampoints`
- `funpay_mvp_plugin_steamgifts_autolot`
- `funpay_mvp_plugin_autorent_steam`
- `funpay_mvp_plugin_autorefund`
- `funpay_mvp_plugin_fragment_stars_buyer`
- `funpay_mvp_plugin_autolog`
- `funpay_mvp_plugin_autopricemonitor`


## 9. Логи и диагностика

Самые полезные логи:

```bash
docker logs -f funpay_mvp_control_bot
docker logs -f funpay_mvp_funpay_connector
docker logs -f funpay_mvp_plugin_router
docker logs -f funpay_mvp_outbox_dispatcher
docker logs -f funpay_mvp_plugin_autostars
docker logs -f funpay_mvp_plugin_autosteampoints
docker logs -f funpay_mvp_plugin_steamgifts_autolot
docker logs -f funpay_mvp_plugin_autorent_steam
docker logs -f funpay_mvp_plugin_autorefund
docker logs -f funpay_mvp_plugin_fragment_stars_buyer
docker logs -f funpay_mvp_plugin_autolog
docker logs -f funpay_mvp_plugin_autopricemonitor
```

Если бот не шлет уведомления в Telegram:

- проверь `TG_BOT_TOKEN`
- проверь `outbox_dispatcher`
- проверь, что `control_bot` вообще стартовал без ошибки

Если события из FunPay не приходят:

- проверь `/accountcheck`
- проверь, что аккаунт включен через `/listenon`
- смотри `funpay_connector`

Если плагин включен, но ничего не происходит:

- убедись, что есть entitlement
- убедись, что плагин реально присутствует в `docker-compose.prod.yml`
- смотри лог конкретного контейнера плагина


## 10. Первый запуск через Telegram-бота

### Шаг 1. Запусти бота

После поднятия контейнеров открой своего Telegram-бота.

Команда:

```text
/start
```

### Шаг 2. Сохрани аккаунт FunPay

```text
/accountset <golden_key> [user_agent]
```

Что сюда подается:

- `<golden_key>` — cookie `golden_key` из браузера, где ты авторизован на FunPay
- `[user_agent]` — необязателен, но лучше передавать тот же User-Agent, что у браузера

Пример:

```text
/accountset abcdef1234567890 Mozilla/5.0 (Windows NT 10.0; Win64; x64)
```

### Шаг 3. Проверь авторизацию

```text
/accountcheck
```

Ожидаемый результат:

- бот вернет username и id аккаунта FunPay

### Шаг 4. Включи слушатель

```text
/listenon
```

Если нужно временно остановить чтение FunPay:

```text
/listenoff
```

### Шаг 5. Выдай себе доступ к плагинам

Если платежи не настроены, используй админскую команду:

```text
/grant <tg_user_id> <plugin> <days>
```

Пример:

```text
/grant 123456789 autostars 30
```

### Шаг 6. Открой список плагинов

```text
/plugins
```

Там можно:

- открыть карточку плагина
- включить/выключить плагин
- купить плагин, если настроены Telegram payments


## 11. Базовые команды бота

- `/start`
- `/help`
- `/accountset <golden_key> [user_agent]`
- `/accountcheck`
- `/listenon`
- `/listenoff`
- `/plugins`
- `/grant <tg_user_id> <plugin> <days>` — только для admin из `CONTROL_ADMIN_IDS_JSON`


## 12. Использование плагинов

## 12.1 AutoStars

### Для чего нужен

Автоматически обрабатывает заказы на Telegram Stars:

- определяет количество Stars по `lot_id`
- получает `@username` покупателя
- дергает HTTP-провайдер
- отслеживает подтверждение выдачи
- может автоматически закрыть заказ

### Что нужно настроить

1. Выдать entitlement на `autostars`
2. Включить плагин в `/plugins`
3. Настроить хотя бы один лот
4. Настроить источник Stars

### Команды

- `/autostars_lots`
- `/autostars_addlot <lot_id> <stars_per_unit> [source_id]`
- `/autostars_dellot <lot_id>`
- `/autostars_sources`
- `/autostars_source_http_add <source_id> <base_url> <token> [daily_limit_stars] [max_stars_per_tx] [timeout_sec] [min_balance_alert]`
- `/autostars_source_del <source_id>`
- `/autostars_request_text <текст>`
- `/autostars_delivered_text <текст>`

### Минимальная настройка

```text
/autostars_addlot 123456 50
```

Это значит:

- если покупают 1 единицу лота `123456`, надо отправить 50 Stars
- если покупают 3 единицы, плагин посчитает 150 Stars

Если нужен отдельный источник:

```text
/autostars_source_http_add reserve1 http://autostars_provider_http:8080 SECRET_TOKEN 0 0 20 0
/autostars_addlot 123456 50 reserve1
```

### Как работает сценарий

1. Приходит заказ по привязанному `lot_id`.
2. Плагин ищет `@username`:
   - сначала в описании заказа;
   - если не нашел, пишет покупателю текст из `request_username_text`.
3. Когда заказ становится `PAID` и username известен, плагин отправляет заявку провайдеру.
4. Если провайдер отвечает `PENDING`, плагин ждет подтверждения.
5. После `CONFIRMED`:
   - пишет покупателю сообщение о доставке;
   - помечает задачу как done;
   - может закрыть заказ автоматически.

### Что важно знать

- `source_id=default` берет настройки из `.env`
- есть дневной лимит, лимит на транзакцию и alert по низкому балансу
- если баланса нет, плагин уходит в ожидание и периодически пытается продолжить


## 12.2 AutoSteamPoints

### Для чего нужен

Обрабатывает заказы на Steam Community Points:

- запрашивает Steam-профиль у покупателя
- валидирует профиль
- просит подтвердить профиль
- создает заказ у внешнего поставщика
- при успехе закрывает заказ
- при ошибке может автоматически оформить возврат

### Команды

- `/autosteampoints_help`
- `/autosteampoints_cfg`
- `/autosteampoints_http_set <base_url> <token> [timeout_sec]`
- `/autosteampoints_lots`
- `/autosteampoints_lots_add <lot_id> <points_per_unit>`
- `/autosteampoints_lots_del <lot_id>`
- `/autosteampoints_sla <секунды>`
- `/autosteampoints_replay`

### Минимальная настройка

```text
/autosteampoints_http_set https://provider.example/api TOKEN 25
/autosteampoints_lots_add 234567 1000
/autosteampoints_sla 600
```

### Как работает сценарий

1. При оплате заказа плагин определяет количество points по `lot_id`.
2. Ищет Steam-профиль:
   - в описании заказа;
   - если не находит, просит покупателя прислать ссылку или SteamID64.
3. Проверяет профиль и просит ответить `Да` или `Нет`.
4. После подтверждения отправляет заказ провайдеру.
5. Если поставщик вернул `COMPLETED`, заказ закрывается.
6. Если поставщик вернул `OUT_OF_STOCK`, покупатель получает уведомление, лот может быть деактивирован.
7. Если включен `auto_refund`, при провале плагин делает отмену заказа.

### Полезно знать

- `replay` переводит задачи `WAITING_STOCK` обратно в `NEW`
- есть авто-реактивация лотов после восстановления баланса поставщика


## 12.3 Steam Gifts Auto Lot

### Для чего нужен

Синхронизирует лоты FunPay с поставщиком Steam Gift и делает авто-доставку:

- хранит связь `lot_id -> ns_sku`
- может обновлять цены и отключать лоты при `OUT_OF_STOCK`
- при заказе просит Steam-профиль или email
- может запрашивать регион
- после доставки отправляет payload покупателю

### Команды

- `/steamgifts_help`
- `/steamgifts_cfg`
- `/steamgifts_provider http`
- `/steamgifts_http_set <base_url> <token> [timeout_sec]`
- `/steamgifts_http_clear`
- `/steamgifts_sync_interval <sec>`
- `/steamgifts_markup_default <pct>`
- `/steamgifts_round_step <step>`
- `/steamgifts_lots`
- `/steamgifts_lots_add <lot_id> <ns_sku> [allowed_regions_csv] [markup_pct]`
- `/steamgifts_lots_del <lot_id>`
- `/steamgifts_replay`

### Минимальная настройка

```text
/steamgifts_provider http
/steamgifts_http_set https://nsgifts.example/api TOKEN 25
/steamgifts_markup_default 10
/steamgifts_round_step 1.00
/steamgifts_lots_add 345678 sku_elden_ring RU,KZ,TR 12.5
```

### Как работает сценарий

1. При оплате заказа плагин понимает, какой `ns_sku` нужен.
2. Ищет у покупателя:
   - Steam-профиль;
   - или email;
   - при необходимости регион.
3. Если данных не хватает, пишет в чат и ждет.
4. Когда данных достаточно, создает заказ у провайдера.
5. Если поставщик вернул доставку, плагин отправляет payload покупателю.
6. Если включен `autocomplete_order`, заказ помечается `DONE`.

### Отдельный режим синхронизации каталога

Плагин периодически:

- тянет каталог провайдера;
- обновляет цены FunPay;
- может менять описание;
- может выключать лот при отсутствии SKU или стока;
- может включать лот обратно после восстановления.


## 12.4 AutoRent Steam

### Для чего нужен

Автоматически выдает логин/пароль от Steam-аккаунта на время аренды.

### Что обязательно настроить

1. Привязать `lot_id` к `game_key` и длительности аренды
2. Добавить пул Steam-аккаунтов

### Команды

- `/autorent_help`
- `/autorent_cfg`
- `/autorent_lots`
- `/autorent_lots_add <lot_id> <hours> <game_key>`
- `/autorent_lots_del <lot_id>`
- `/autorent_warn <seconds>`
- `/autorent_default_duration <seconds>`
- `/autorent_account_add <login> <password> <game_key1,game_key2,...>`
- `/autorent_accounts`
- `/autorent_account_disable <id>`
- `/autorent_account_enable <id>`
- `/autorent_account_del <id>`
- `/autorent_rotator_off`
- `/autorent_rotator_http_set <base_url> <token>`

### Минимальная настройка

```text
/autorent_lots_add 456789 24 cs2
/autorent_account_add rent_acc_01 super_password cs2
/autorent_warn 600
```

### Как работает сценарий

1. При создании заказа бот может отправить покупателю подтверждение, что аренда готовится.
2. После оплаты ищется подходящий аккаунт из пула.
3. Если аккаунт найден:
   - покупатель получает логин/пароль;
   - создается lease с временем окончания;
   - заказ идет в успешную выдачу.
4. Если аккаунтов нет, заказ уходит в `WAITING_STOCK`.
5. Перед окончанием аренды плагин может прислать предупреждение.
6. После окончания аренды:
   - если подключен внешний rotator, пароль меняется автоматически;
   - если rotator выключен, аккаунт деактивируется вручную после освобождения.

### Важное ограничение

Внутри проекта смены пароля Steam нет. Для безопасного revoke нужен внешний rotator по HTTP.


## 12.5 AutoRefund

### Для чего нужен

Реагирует на фразы вида:

- "не получил"
- "возврат"
- "refund"
- "спор"

И в зависимости от правил:

- ставит `MANUAL`
- отвечает покупателю
- оформляет `CANCEL`

### Как устроен сейчас

У плагина нет отдельных команд в `control_bot`. Он живет через `plugin_cfg` и default-конфиг в коде.

По умолчанию:

- режим `dry_run`
- если доказательства доставки нет, может подготовить возврат
- если доказательство доставки есть, переводит кейс в ручную проверку

### Что считается доказательством доставки

Сейчас используется `db.has_delivery_proof(...)`, в том числе через state других плагинов.

### Практически использовать так

1. Выдать entitlement на `autorefund`
2. Включить его в `/plugins`
3. Сначала оставить режим `dry_run`
4. Проверять уведомления продавцу в Telegram

Если хочешь реальную автоматическую отмену, нужно отдельно изменить `plugin_cfg` для `autorefund` и перевести `mode=active`.


## 12.6 Fragment Stars Buyer

### Для чего нужен

Автоматически покупает Stars по стратегии:

- отслеживает рыночную цену
- покупает, если цена ниже порога
- считает inventory и среднюю себестоимость
- умеет стоп-лосс и лимиты

### Команды

- `/fragment_help`
- `/fragment_status`
- `/fragment_strategy_on <price_ton_per_star> <buy_stars> [min_interval_sec] [max_inventory_stars] [max_total_ton]`
- `/fragment_strategy_off`
- `/fragment_risk_set <daily_limit_ton> <slippage_bps> <stoploss_window_sec> <stoploss_drop_percent> <halt_minutes>`
- `/fragment_provider <mock|http>`
- `/fragment_http_set <base_url> <token> [timeout_sec]`
- `/fragment_http_clear`
- `/fragment_halt_clear`
- `/fragment_mock_setprice <ton_per_star>`
- `/fragment_mock_setavail <stars>`
- `/fragment_inventory_reset`

### Минимальная настройка

```text
/fragment_provider http
/fragment_http_set https://fragment-buyer.example/api TOKEN 20
/fragment_risk_set 10 200 3600 5 60
/fragment_strategy_on 0.03 100 180 5000 3
```

### Как работает

1. Плагин периодически получает рыночную цену.
2. Если цена с учетом slippage ниже порога, пробует купить Stars.
3. Перед покупкой проверяет:
   - паузу между покупками;
   - дневной лимит;
   - лимит инвентаря;
   - stop-loss.
4. После покупки обновляет inventory и PnL.
5. При ошибке ставит halt на заданное время.

### Для тестов

Можно использовать mock:

```text
/fragment_provider mock
/fragment_mock_setprice 0.025
/fragment_mock_setavail 1000
```


## 12.7 AutoLog

### Для чего нужен

Логирует события заказов и умеет выгружать статистику.

### Команды

- `/autolog_stats`
- `/autolog_stats week`
- `/autolog_stats month`
- `/autolog_export`
- `/autolog_export csv week`
- `/autolog_export xlsx month`

### Что делает

- пишет события `fp.order.created`, `fp.order.paid`, `fp.order.status_changed`
- ведет snapshot заказа
- подтягивает факт доставки из state других плагинов
- отдает:
   - число созданных заказов
   - число оплат
   - выручку
   - топ-лот
   - число возвратов
   - CSV/XLSX-экспорт


## 12.8 AutoPriceMonitor

### Для чего нужен

Следит за ценами конкурентов на FunPay и:

- просто уведомляет;
- либо автоматически снижает цену твоего лота.

### Команды

- `/apm_help`
- `/apm_cfg`
- `/apm_poll <sec>`
- `/apm_add_url <name> <url> <my_lot_id> <my_price> [strategy]`
- `/apm_add_category <name> <chips|lots> <category_id> <my_lot_id> <my_price> [strategy]`
- `/apm_enable <target_id>`
- `/apm_disable <target_id>`
- `/apm_del <target_id>`
- `/apm_set <target_id> <field> <value>`
- `/apm_keywords <target_id> ...`
- `/apm_ignore_add <target_id> <seller1> [seller2 ...]`
- `/apm_ignore_del <target_id> <seller1> [seller2 ...]`
- `/apm_status`
- `/apm_status <target_id>`

### Стратегии

- `notify_only` — только уведомления
- `aggressive` — цена чуть ниже лучшего конкурента
- `conservative` — снижение в пределах ограничений и `floor_price`

### Пример

```text
/apm_add_category Robux100 chips 123 999999 100 notify_only
/apm_status
/apm_set 1a2b3c strategy aggressive
/apm_set 1a2b3c floor_price 95
/apm_set 1a2b3c undercut_step 1
/apm_keywords 1a2b3c Robux 100
```

### Как работает

1. Плагин качает листинг FunPay.
2. Находит офферы конкурентов.
3. Исключает:
   - твой `my_lot_id`
   - продавцов из ignore-списка
   - продавцов с низким числом отзывов
4. Находит лучшую цену.
5. Если конкурент дешевле:
   - уведомляет продавца;
   - либо ставит изменение цены в outbox.


## 12.9 AutoBalance

### Статус

Логика и команды есть, но в `docker-compose.prod.yml` контейнер плагина не запускается.

### Для чего нужен

Мониторит баланс поставщиков и на основе `low/high`:

- выключает лоты;
- включает их обратно;
- уведомляет продавца.

### Команды

- `/autobalance_help`
- `/autobalance_cfg`
- `/autobalance_status`
- `/autobalance_poll <секунды>`
- `/autobalance_provider_http_add <provider_id> <name> <url> <token> [json_path] [timeout_sec]`
- `/autobalance_provider_mock_add <provider_id> <name> <balance>`
- `/autobalance_provider_del <provider_id>`
- `/autobalance_resource_set <resource_key> <provider_id> <low> <high> [unit]`
- `/autobalance_resource_del <resource_key>`
- `/autobalance_resource_lots_add <resource_key> <lot_id1> [lot_id2 ...]`
- `/autobalance_resource_lots_del <resource_key> <lot_id1> [lot_id2 ...]`

### Чтобы реально использовать

Добавь в `docker-compose.prod.yml` контейнер `plugin_autobalance` по аналогии с `infra/docker-compose.yml`.


## 12.10 AutoLotDeactivator

### Статус

Логика и команды есть, но в `docker-compose.prod.yml` контейнер плагина не запускается.

### Для чего нужен

Получает сигналы по лотам и:

- автоматически выключает/включает лоты;
- или только рекомендует действие;
- поддерживает hold-time и отдельный список лотов с `auto=OFF`.

### Команды

- `/autolot_help`
- `/autolot_cfg`
- `/autolot_mode auto|notify`
- `/autolot_lot_autooff <lot_id>`
- `/autolot_lot_autoon <lot_id>`
- `/autolot_lot_list`
- `/dlm`
- `/lots_manager`

### Delete Lots Manager

Это интерактивный UI внутри Telegram-бота:

- загружает категории лотов из FunPay
- позволяет выбрать категории
- показывает предпросмотр
- удаляет лоты через outbox только после подтверждения

### Чтобы реально использовать

Добавь в `docker-compose.prod.yml` контейнер `plugin_autolotdeactivator`.


## 12.11 Плагины из каталога, которые сейчас не готовы к эксплуатации

Следующие плагины есть в `PLUGIN_CATALOG`, но в `plugin_service.plugins.registry` не зарегистрированы как рабочие:

- AutoReply
- Ask TG Username
- AntiFraud
- AutoDelivery
- AutoReview
- PriceGuard
- DailySummary
- Blacklist
- UnpaidReminder

Их можно видеть в `/plugins`, но как полноценные плагины текущего прод-контура они не готовы.


## 13. Рекомендуемый порядок ввода в эксплуатацию

Если ты поднимаешь проект впервые, делай так:

1. Настрой `.env`
2. Подними `docker-compose.prod.yml`
3. Проверь, что живы `postgres`, `rabbitmq`, `control_bot`, `funpay_connector`, `outbox_dispatcher`
4. В боте выполни:
   - `/accountset`
   - `/accountcheck`
   - `/listenon`
5. Выдай себе тестовый доступ через `/grant`
6. Включай по одному плагину
7. После включения каждого плагина делай тестовый заказ
8. Смотри логи конкретного плагина

Для первой проверки проще всего запускать в таком порядке:

1. `autolog`
2. `autostars`
3. `autosteampoints`
4. `steamgifts_autolot`
5. `autorent_steam`
6. `autorefund`
7. `autopricemonitor`
8. `fragment_stars_buyer`


## 14. Известные ограничения и несоответствия текущей версии

### 1. Использовать нужно именно `docker-compose.prod.yml`

Обычный `infra/docker-compose.yml` содержит старые/лишние plugin-контейнеры, часть из которых не зарегистрирована в `plugin_service.plugins.registry`.

### 2. Не все плагины из витрины реально реализованы

Каталог в боте шире, чем реально запускаемые plugin workers.

### 3. `autobalance` и `autolotdeactivator` написаны, но не стартуют в `prod`-compose

Для их реального использования контейнеры нужно добавить вручную.

### 4. У `fragment_stars_buyer` в коде используются outbox-события `tg_notify`

`outbox_dispatcher` обрабатывает `out.tg.notify_seller`, а не `tg_notify`, поэтому Telegram-уведомления Fragment Buyer в текущем виде выглядят несогласованными с dispatcher.

### 5. В `control_bot` есть обращения к entitlement-полям, которые не совпадают с `EntitlementRow`

Это затрагивает по меньшей мере команды вокруг Fragment/SteamGifts-статуса. Если соответствующие команды ведут себя странно, проблема именно здесь, а не в Docker.


## 15. Короткий чек-лист "проект жив"

Проект можно считать запущенным, если:

- бот отвечает на `/start`
- `/accountcheck` проходит успешно
- `/listenon` включен
- при новом заказе в `funpay_connector` видны события
- в `plugin_router` видно маршрутизацию
- в `outbox_dispatcher` уходят сообщения/статусы
- хотя бы один включенный плагин завершает свой сценарий до конца
