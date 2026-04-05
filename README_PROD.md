# FunPay MVP: локальный запуск и использование на Windows + WSL Ubuntu

Этот файл теперь является одной точкой входа под твой сценарий:

- Windows
- Docker Desktop
- WSL Ubuntu
- проект лежит в `/home/max/funpay-mvp`
- запуск выполняется из терминала Ubuntu в WSL

Ниже уже учтены текущие изменения в репозитории:

- в `infra/docker-compose.prod.yml` добавлены `plugin_autobalance` и `plugin_autolotdeactivator`;
- в `.env` добавлен `APP_TIMEZONE=Europe/Moscow`;
- схема провайдеров зафиксирована так:
  - `AutoStars` -> HTTP/Webhook provider Stars
  - `AutoSteamPoints` -> HTTP provider Steam Points
  - `Steam Gifts Auto Lot` -> HTTP provider NS.Gifts
  - `AutoRent Steam` -> пул Steam-аккаунтов
  - `Fragment Stars Buyer` -> HTTP provider покупки Stars


## 1. Откуда запускать проект

Твой рабочий путь внутри WSL:

```bash
/home/max/funpay-mvp
```

Из Windows это тот же каталог:

```text
\\wsl.localhost\Ubuntu\home\max\funpay-mvp
```

Запускать `docker compose` нужно именно из Ubuntu в WSL, а не из PowerShell по UNC-пути.


## 2. Что должно быть установлено

Нужно:

- WSL Ubuntu
- Docker Desktop на Windows
- включенная WSL integration в Docker Desktop для твоего Ubuntu-дистрибутива

Проверка из Ubuntu:

```bash
docker --version
docker compose version
```

Если команды не работают, открой Docker Desktop:

1. `Settings`
2. `Resources`
3. `WSL Integration`
4. включи интеграцию для своего Ubuntu


## 3. Конкретные команды для запуска в WSL Ubuntu

Открыть Ubuntu и выполнить:

```bash
cd /home/max/funpay-mvp
docker --version
docker compose version

cd /home/max/funpay-mvp/infra
docker compose -f docker-compose.prod.yml config >/dev/null && echo "compose ok"
docker compose -f docker-compose.prod.yml up -d --build
docker compose -f docker-compose.prod.yml ps
```

Если нужно посмотреть логи:

```bash
cd /home/max/funpay-mvp/infra
docker compose -f docker-compose.prod.yml logs -f control_bot
docker compose -f docker-compose.prod.yml logs -f funpay_connector
docker compose -f docker-compose.prod.yml logs -f plugin_autostars
docker compose -f docker-compose.prod.yml logs -f plugin_autobalance
docker compose -f docker-compose.prod.yml logs -f plugin_autolotdeactivator
```

Если нужно остановить:

```bash
cd /home/max/funpay-mvp/infra
docker compose -f docker-compose.prod.yml down
```

Если нужно перезапустить после изменений:

```bash
cd /home/max/funpay-mvp/infra
docker compose -f docker-compose.prod.yml up -d --build
```


## 4. Что сейчас готово в твоем `.env`

На текущий момент для запуска уже заполнены и выглядят готовыми:

- `TG_BOT_TOKEN`
- `CONTROL_ADMIN_IDS_JSON`
- `POSTGRES_USER`
- `POSTGRES_PASSWORD`
- `POSTGRES_DB`
- `PG_DSN`
- `RABBITMQ_USER`
- `RABBITMQ_PASS`
- `AMQP_URL`
- `MASTER_KEY_B64`
- `LOG_LEVEL`
- `DRY_RUN`
- `AUTOSTARS_PROVIDER`
- `AUTOSTARS_HTTP_BASE_URL`
- `AUTOSTARS_HTTP_TOKEN`
- `AUTOSTARS_PROVIDER_BACKEND`
- `AUTOSTARS_PROVIDER_WEBHOOK_URL`
- `AUTOSTARS_PROVIDER_WEBHOOK_TOKEN`
- `FRAGMENT_BUYER_PROVIDER`
- `FRAGMENT_BUYER_HTTP_BASE_URL`
- `FRAGMENT_BUYER_HTTP_TOKEN`

Явно добавлен:

- `APP_TIMEZONE=Europe/Moscow`

Сейчас отсутствуют, но это не блокирует запуск проекта:

- `AUTOSTEAMPOINTS_PROVIDER_KIND`
- `AUTOSTEAMPOINTS_PROVIDER_BASE_URL`
- `AUTOSTEAMPOINTS_PROVIDER_TOKEN`
- `STEAMGIFTS_PROVIDER_KIND`
- `STEAMGIFTS_PROVIDER_BASE_URL`
- `STEAMGIFTS_PROVIDER_TOKEN`

Почему это не блокер:

- в текущем коде `AutoSteamPoints` настраивается через Telegram-команду `/autosteampoints_http_set`, а не из `.env`;
- в текущем коде `Steam Gifts Auto Lot` настраивается через `/steamgifts_http_set`, а не из `.env`.

Важно:

- если `DRY_RUN=0`, проект будет выполнять реальные действия;
- если хочешь первый запуск без реальных действий в FunPay, временно поставь `DRY_RUN=1`;
- если `PAYMENTS_PROVIDER_TOKEN` пустой, встроенная покупка подписок через Telegram не работает, и доступ к плагинам нужно выдавать через `/grant`.


## 5. Какие контейнеры должны подняться

После `docker compose ... up -d --build` ожидаем:

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
- `funpay_mvp_plugin_autobalance`
- `funpay_mvp_plugin_autolotdeactivator`
- `funpay_mvp_plugin_autopricemonitor`


## 6. Первый запуск бота после старта контейнеров

Последовательность для первого реального запуска:

1. Открой Telegram-бота и отправь:

```text
/start
```

2. Сохрани FunPay-аккаунт:

```text
/accountset <golden_key> [user_agent]
```

3. Проверь, что FunPay-аккаунт читается:

```text
/accountcheck
```

4. Включи слушатель событий:

```text
/listenon
```

5. Открой список плагинов:

```text
/plugins
```

6. Так как `PAYMENTS_PROVIDER_TOKEN` сейчас не заполнен, доступ к плагинам выдавай себе через админ-команду:

```text
/grant <your_tg_user_id> <plugin> <days>
```

Примеры:

```text
/grant <your_tg_user_id> autostars 365
/grant <your_tg_user_id> autobalance 365
/grant <your_tg_user_id> autolotdeactivator 365
```

7. После выдачи доступа включай нужный плагин через `/plugins`.

Поддерживаемые имена плагинов для `/grant`:

- `autostars`
- `autobalance`
- `autolotdeactivator`
- `autosteampoints`
- `autorent_steam`
- `fragment_stars_buyer`
- `steamgifts_autolot`
- `autorefund`
- `autolog`
- `autopricemonitor`


## 7. Как пользоваться ботом в целом

Базовые команды:

- `/start` - открыть стартовое меню
- `/accountset <golden_key> [user_agent]` - сохранить FunPay cookie
- `/accountcheck` - проверить доступ к FunPay и сохранить `funpay_user_id`
- `/listenon` - включить обработку событий
- `/listenoff` - выключить обработку событий
- `/plugins` - открыть список плагинов и включать/выключать их
- `/grant <tg_user_id> <plugin> <days>` - выдать доступ к плагину

Рекомендуемый базовый сценарий:

1. сохранить FunPay через `/accountset`
2. проверить через `/accountcheck`
3. выдать себе доступы через `/grant`
4. включить нужные плагины в `/plugins`
5. настроить каждый плагин его командами
6. держать включенным `/listenon`


## 8. Плагины и их провайдеры

### 8.1 AutoStars

Назначение:

- продает Telegram Stars по лотам FunPay;
- использует HTTP source;
- базовый source уже берется из `.env`;
- внутри provider microservice используется Webhook backend.

Что уже готово у тебя:

- `AUTOSTARS_PROVIDER=http`
- `AUTOSTARS_HTTP_BASE_URL=http://autostars_provider_http:8080`
- `AUTOSTARS_HTTP_TOKEN` заполнен
- `AUTOSTARS_PROVIDER_BACKEND=webhook`
- `AUTOSTARS_PROVIDER_WEBHOOK_URL` заполнен
- `AUTOSTARS_PROVIDER_WEBHOOK_TOKEN` заполнен

Основные команды:

- `/autostars_sources`
- `/autostars_lots`
- `/autostars_addlot <lot_id> <stars_per_unit> [source_id]`
- `/autostars_dellot <lot_id>`
- `/autostars_source_http_add <source_id> <base_url> <token> [daily_limit_stars] [max_stars_per_tx] [timeout_sec] [min_balance_alert]`
- `/autostars_source_del <source_id>`
- `/autostars_request_text <текст>`
- `/autostars_delivered_text <текст>`

Базовая настройка:

1. Включи плагин в `/plugins`.
2. Посмотри базовый source:

```text
/autostars_sources
```

3. Привяжи FunPay-лот к количеству Stars:

```text
/autostars_addlot <lot_id> <stars_per_unit>
```

Пример:

```text
/autostars_addlot 123456 100
```

4. Проверь результат:

```text
/autostars_lots
```

5. Если нужен отдельный резервный источник, добавь его:

```text
/autostars_source_http_add reserve1 https://provider.example/api TOKEN 0 0 20 0
```

6. Если хочешь поменять текст запроса юзернейма и успешной доставки:

```text
/autostars_request_text Напиши @username для отправки Stars
/autostars_delivered_text Stars отправлены: {stars}, username: {username}, tx: {tx}
```

Как использовать в работе:

- покупатель оплачивает FunPay-лот;
- плагин определяет `lot_id`;
- рассчитывает, сколько Stars надо отправить;
- берет токен источника;
- отправляет через HTTP provider;
- пишет продавцу и покупателю результат.


### 8.2 AutoSteamPoints

Назначение:

- обрабатывает заказы на Steam Points;
- использует внешний HTTP provider;
- в текущем проекте настраивается через Telegram, не через `.env`.

Основные команды:

- `/autosteampoints_help`
- `/autosteampoints_cfg`
- `/autosteampoints_http_set <base_url> <token> [timeout_sec]`
- `/autosteampoints_lots`
- `/autosteampoints_lots_add <lot_id> <points_per_unit>`
- `/autosteampoints_lots_del <lot_id>`
- `/autosteampoints_sla <seconds>`
- `/autosteampoints_replay`

Базовая настройка:

1. Включи плагин.
2. Сохрани HTTP provider:

```text
/autosteampoints_http_set https://provider.example/api TOKEN 25
```

3. Привяжи лот FunPay к количеству points за единицу:

```text
/autosteampoints_lots_add 123456 100
```

4. Поставь SLA ожидания:

```text
/autosteampoints_sla 600
```

5. Проверь конфиг:

```text
/autosteampoints_cfg
/autosteampoints_lots
```

Как использовать в работе:

- покупатель оплачивает лот;
- плагин отправляет заказ во внешний Steam Points provider;
- затем опрашивает provider по статусу;
- если provider временно без стока, задача уходит в ожидание;
- команда `/autosteampoints_replay` переводит зависшие `WAITING_STOCK` обратно в `NEW`.

Ожидаемая логика HTTP provider:

- `GET /balance`
- `POST /orders`
- `GET /orders/{provider_order_id}`


### 8.3 Steam Gifts Auto Lot

Назначение:

- синхронизирует лоты Steam Gift с внешним NS.Gifts provider;
- может обновлять цены/сток и автоматически доставлять товар после оплаты.

Важно:

- в текущем коде этот плагин тоже настраивается через Telegram-команды;
- переменные `STEAMGIFTS_PROVIDER_*` из `.env.example` сейчас не являются обязательными для запуска.

Основные команды:

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

Базовая настройка:

1. Включи плагин.
2. Зафиксируй тип provider:

```text
/steamgifts_provider http
```

3. Сохрани NS.Gifts provider:

```text
/steamgifts_http_set https://nsgifts.example/api TOKEN 25
```

4. Настрой период синхронизации:

```text
/steamgifts_sync_interval 1800
```

5. Настрой наценку и округление:

```text
/steamgifts_markup_default 10
/steamgifts_round_step 1.00
```

6. Привяжи FunPay-лот к SKU поставщика:

```text
/steamgifts_lots_add 123456 GAME_SKU RU,KZ 10
```

7. Проверь состояние:

```text
/steamgifts_cfg
/steamgifts_lots
```

Как использовать в работе:

- плагин периодически синхронизирует цену и доступность;
- при оплате он создаёт заказ у NS.Gifts provider;
- затем опрашивает provider до результата;
- если товар временно отсутствует, заказ можно перекинуть обратно на повтор через `/steamgifts_replay`.


### 8.4 AutoRent Steam

Назначение:

- сдаёт в аренду Steam-аккаунты;
- внешний provider для продажи не нужен;
- основой является твой пул Steam-аккаунтов.

Основные команды:

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

Базовая настройка:

1. Включи плагин.
2. Привяжи лот к игре и длительности аренды:

```text
/autorent_lots_add 123456 24 cs2
```

3. Добавь Steam-аккаунт в пул:

```text
/autorent_account_add mylogin mypassword cs2
```

Если один аккаунт подходит для нескольких игр:

```text
/autorent_account_add mylogin mypassword cs2,dota2
```

4. Проверь пул:

```text
/autorent_accounts
/autorent_lots
```

5. При необходимости настрой предупреждение перед окончанием аренды:

```text
/autorent_warn 600
```

6. Если используешь внешний rotator для смены паролей:

```text
/autorent_rotator_http_set https://rotator.example/api TOKEN
```

Как использовать в работе:

- покупатель оплачивает лот;
- плагин ищет свободный Steam-аккаунт с нужным `game_key`;
- выдаёт данные покупателю;
- помечает аккаунт занятым на время аренды;
- по окончании срока освобождает аккаунт;
- при наличии rotator может сменить пароль автоматически.


### 8.5 Fragment Stars Buyer

Назначение:

- автоматически покупает Stars по стратегии через HTTP provider;
- полезен как закупочный модуль;
- у тебя базовые `FRAGMENT_BUYER_*` в `.env` уже заполнены.

Основные команды:

- `/fragment_help`
- `/fragment_status`
- `/fragment_strategy_on <price_ton_per_star> <buy_stars> [min_interval_sec] [max_inventory_stars] [max_total_ton]`
- `/fragment_strategy_off`
- `/fragment_risk_set <daily_limit_ton> <slippage_bps> <stoploss_window_sec> <stoploss_drop_percent> <halt_minutes>`
- `/fragment_provider <mock|http>`
- `/fragment_http_set <base_url> <token> [timeout_sec]`
- `/fragment_http_clear`
- `/fragment_halt_clear`
- `/fragment_inventory_reset`

Базовая настройка:

1. Включи плагин.
2. Если хочешь использовать уже заполненный HTTP provider, просто включи HTTP режим:

```text
/fragment_provider http
```

3. Если нужно переопределить адрес или токен прямо из бота:

```text
/fragment_http_set https://provider.example/api TOKEN 25
```

4. Настрой риск-профиль:

```text
/fragment_risk_set 10 300 3600 5 60
```

Это значит:

- дневной лимит 10 TON
- slippage 300 bps
- окно stop-loss 3600 секунд
- stop-loss падение 5%
- пауза после stop-loss 60 минут

5. Включи стратегию:

```text
/fragment_strategy_on 0.015 50 300 1000 10
```

Это значит:

- покупать по цене не выше `0.015 TON/Star`
- размер одной покупки `50 Stars`
- минимальный интервал `300` секунд
- максимум на складе `1000 Stars`
- максимум суммарно `10 TON`

6. Следи за статусом:

```text
/fragment_status
```

Если стратегия остановилась:

```text
/fragment_halt_clear
```

Если хочешь сбросить локальный расчёт инвентаря:

```text
/fragment_inventory_reset
```

Как использовать в работе:

- этот плагин не продаёт, а закупает;
- он смотрит рыночную цену;
- при совпадении условий делает покупку через provider;
- ведёт локальный inventory и PnL;
- умеет останавливать себя по stop-loss или ошибкам покупки.


### 8.6 AutoBalance

Назначение:

- мониторит балансы провайдеров;
- отключает лоты при низком остатке;
- включает их обратно при восстановлении баланса;
- отправляет Telegram-уведомления.

Этот плагин теперь добавлен в `infra/docker-compose.prod.yml` и запускается вместе с остальными.

Основные команды:

- `/autobalance_help`
- `/autobalance_cfg`
- `/autobalance_status`
- `/autobalance_poll <seconds>`
- `/autobalance_provider_http_add <provider_id> <name> <url> <token> [json_path] [timeout_sec]`
- `/autobalance_provider_mock_add <provider_id> <name> <balance>`
- `/autobalance_provider_del <provider_id>`
- `/autobalance_resource_set <resource_key> <provider_id> <low> <high> [unit]`
- `/autobalance_resource_del <resource_key>`
- `/autobalance_resource_lots_add <resource_key> <lot_id1> [lot_id2 ...]`
- `/autobalance_resource_lots_del <resource_key> <lot_id1> [lot_id2 ...]`

Как устроено:

- `provider` = откуда брать баланс;
- `resource` = какой ресурс контролируется и какими порогами;
- `lots` = какие лоты надо отключать/включать при смене состояния.

Логика порогов:

- если баланс `<= low`, связанные лоты отключаются;
- если баланс `>= high`, связанные лоты включаются обратно.

Пример для Stars через `autostars_provider_http`:

```text
/autobalance_provider_http_add stars Stars http://autostars_provider_http:8080/balance <AUTOSTARS_HTTP_TOKEN> balance 20
/autobalance_resource_set stars stars 500 700 Stars
/autobalance_resource_lots_add stars 123456 987654
```

Пример для Steam Points provider:

```text
/autobalance_provider_http_add steampoints SteamPoints https://provider.example/api/balance TOKEN available_points 20
/autobalance_resource_set steampoints steampoints 1000 3000 points
/autobalance_resource_lots_add steampoints 111111 222222
```

Пример для NS.Gifts:

```text
/autobalance_provider_http_add nsgifts NS.Gifts https://nsgifts.example/api/balance TOKEN balance 20
/autobalance_resource_set gifts nsgifts 1 3 gifts
/autobalance_resource_lots_add gifts 333333
```

После настройки проверь:

```text
/autobalance_cfg
/autobalance_status
```


### 8.7 AutoLotDeactivator

Назначение:

- автоматически отключает и включает лоты;
- умеет работать в режиме только уведомлений;
- дополнительно открывает интерфейс `Delete Lots Manager`.

Этот плагин теперь тоже добавлен в `infra/docker-compose.prod.yml`.

Основные команды:

- `/autolot_cfg`
- `/autolot_mode auto|notify`
- `/autolot_lot_autooff <lot_id>`
- `/autolot_lot_autoon <lot_id>`
- `/autolot_lot_list`
- `/dlm`
- `/lots_manager`

Базовая настройка:

1. Включи плагин.
2. Выставь режим:

```text
/autolot_mode auto
```

или:

```text
/autolot_mode notify
```

3. Если конкретный лот не должен переключаться автоматически, отключи для него авто:

```text
/autolot_lot_autooff 123456
```

4. Если нужно вернуть авто обратно:

```text
/autolot_lot_autoon 123456
```

5. Посмотреть список исключений:

```text
/autolot_lot_list
/autolot_cfg
```

6. Для массового управления лотами открой:

```text
/dlm
```

или:

```text
/lots_manager
```

Как использовать в работе:

- режим `auto` подходит, если ты хочешь реально выключать и включать лоты;
- режим `notify` подходит, если пока хочешь только сигналы в Telegram без автоматических действий;
- `DLM` нужен для ручного массового удаления или управления лотами из интерфейса бота.


### 8.8 AutoLog

Назначение:

- собирает статистику по событиям и заказам;
- умеет делать экспорт.

Основные команды:

- `/autolog_stats`
- `/autolog_stats week`
- `/autolog_stats month`
- `/autolog_export`
- `/autolog_export csv week`
- `/autolog_export xlsx month`

Как использовать:

```text
/autolog_stats
/autolog_stats week
/autolog_export csv month
```

Что получаешь:

- количество оплат;
- выручку;
- топ-лот;
- возвраты;
- экспорт в `csv` или `xlsx`.


### 8.9 AutoPriceMonitor

Назначение:

- следит за ценами конкурентов;
- может только уведомлять или менять цену автоматически.

Основные команды:

- `/apm_help`
- `/apm_cfg`
- `/apm_poll 300`
- `/apm_add_url <name> <url> <my_lot_id> <my_price> [strategy]`
- `/apm_add_category <name> <chips|lots> <category_id> <my_lot_id> <my_price> [strategy]`
- `/apm_status`
- `/apm_status <target_id>`
- `/apm_enable <target_id>`
- `/apm_disable <target_id>`
- `/apm_del <target_id>`
- `/apm_set <target_id> <field> <value>`
- `/apm_keywords <target_id> ...`
- `/apm_ignore_add <target_id> <seller1> [seller2 ...]`
- `/apm_ignore_del <target_id> <seller1> [seller2 ...]`

Стратегии:

- `notify_only` - только уведомлять
- `aggressive` - опускать цену чуть ниже минимума
- `conservative` - опускать цену осторожнее

Базовый сценарий:

1. Добавить цель:

```text
/apm_add_category Robux100 chips 123 999999 100 notify_only
```

2. Посмотреть `target_id`:

```text
/apm_status
```

3. До-настроить цель:

```text
/apm_set <target_id> strategy aggressive
/apm_set <target_id> floor_price 95
/apm_set <target_id> undercut_step 1
```

4. Уточнить ключевые слова:

```text
/apm_keywords <target_id> Robux 100 | Cheap Robux
```

5. Исключить продавцов:

```text
/apm_ignore_add <target_id> SomeSeller AnotherSeller
```


### 8.10 AutoRefund

Назначение:

- автоматизирует возвраты и ответы по спорным сообщениям;
- работает по внутренним правилам;
- сейчас не имеет отдельного набора Telegram-команд в `control_bot`.

Что важно знать:

- плагин есть в `prod`-compose;
- он использует внутренний конфиг по умолчанию;
- базовый режим по умолчанию ориентирован на `dry_run`;
- правила проверяют жалобы вроде "не получил", "refund", "спор" и похожие сигналы;
- часть кейсов переводится в ручную проверку, часть может оформлять возврат по правилу.

Как использовать:

1. Выдать себе доступ и включить плагин через `/plugins`.
2. Следить за логами:

```bash
docker compose -f docker-compose.prod.yml logs -f plugin_autorefund
```

3. Для ежедневной работы помнить, что отдельного блока `/autorefund_*` команд в текущем боте нет.


## 9. Что делать сразу после настройки каждого плагина

После настройки любого плагина делай один и тот же короткий цикл:

1. выдай доступ через `/grant`, если он ещё не выдан;
2. включи плагин в `/plugins`;
3. заполни конфиг командами самого плагина;
4. проверь статус или конфиг;
5. посмотри его контейнерный лог;
6. сделай один тестовый заказ на отдельном лоте.


## 10. Где смотреть логи

Самые полезные команды:

```bash
cd /home/max/funpay-mvp/infra

docker compose -f docker-compose.prod.yml logs -f control_bot
docker compose -f docker-compose.prod.yml logs -f funpay_connector
docker compose -f docker-compose.prod.yml logs -f outbox_dispatcher
docker compose -f docker-compose.prod.yml logs -f plugin_autostars
docker compose -f docker-compose.prod.yml logs -f plugin_autosteampoints
docker compose -f docker-compose.prod.yml logs -f plugin_steamgifts_autolot
docker compose -f docker-compose.prod.yml logs -f plugin_autorent_steam
docker compose -f docker-compose.prod.yml logs -f plugin_fragment_stars_buyer
docker compose -f docker-compose.prod.yml logs -f plugin_autobalance
docker compose -f docker-compose.prod.yml logs -f plugin_autolotdeactivator
docker compose -f docker-compose.prod.yml logs -f plugin_autolog
docker compose -f docker-compose.prod.yml logs -f plugin_autopricemonitor
docker compose -f docker-compose.prod.yml logs -f plugin_autorefund
```


## 11. Быстрая диагностика проблем

Если бот не отвечает:

- смотри `control_bot`
- проверь `TG_BOT_TOKEN`

Если бот отвечает, но FunPay не обрабатывается:

- выполни `/accountcheck`
- проверь, что включён `/listenon`
- смотри лог `funpay_connector`

Если плагин не включается:

- проверь, выдан ли доступ через `/grant`
- проверь, что он включён в `/plugins`

Если `AutoSteamPoints` или `Steam Gifts` не работают:

- проверь, что provider задан именно через команды бота;
- не полагайся на пустые переменные `AUTOSTEAMPOINTS_PROVIDER_*` и `STEAMGIFTS_PROVIDER_*` в `.env`, потому что текущий runtime их не использует как основной источник конфигурации.

Если `AutoBalance` не переключает лоты:

- проверь `/autobalance_cfg`
- проверь `/autobalance_status`
- проверь правильность `json_path`
- убедись, что к ресурсу реально привязаны нужные `lot_id`

Если `AutoLotDeactivator` не даёт открыть менеджер:

- проверь, что выдан доступ к `autolotdeactivator`
- проверь, что плагин включён
- затем снова вызови `/dlm`


## 12. Короткий рабочий сценарий на каждый день

Если тебе нужен минимальный реальный сценарий без лишнего:

1. В Ubuntu:

```bash
cd /home/max/funpay-mvp/infra
docker compose -f docker-compose.prod.yml up -d --build
docker compose -f docker-compose.prod.yml ps
```

2. В Telegram:

```text
/accountcheck
/listenon
/plugins
```

3. При необходимости:

```text
/grant <your_tg_user_id> autostars 365
/grant <your_tg_user_id> autobalance 365
/grant <your_tg_user_id> autolotdeactivator 365
```

4. Проверить конкретные рабочие плагины:

```text
/autostars_lots
/autobalance_status
/autolot_cfg
/fragment_status
```

Этого достаточно, чтобы поднимать проект локально из WSL, включать бота и дальше работать уже по секциям выше.
