# Инструкция по тестированию FunPay-бота и плагинов

## 1. Как работает бот и плагины

Общая схема:

1. Пользователь привязывает аккаунт FunPay через `/accountset`.
2. `funpay_connector` слушает события FunPay.
3. `plugin_router` отправляет события только в те плагины, для которых:
   - у пользователя есть доступ (`entitlement`);
   - плагин включен у пользователя в `/plugins`.
4. Плагин:
   - либо обрабатывает событие заказа;
   - либо работает в фоне по таймеру.
5. Если плагин должен сделать действие в FunPay, он кладет задачу в outbox, а `outbox_dispatcher` выполняет ее.

### Важные правила

Плагин начнет реально работать только если одновременно выполнены оба условия:

- пользователю выдан доступ к плагину;
- пользователь вручную включил плагин в `/plugins`.

---

## 2. Базовые команды бота

### `/start`
Открывает стартовое меню бота.

### `/help`
Показывает справку и меню.

### `/accountset <golden_key> [user_agent]`
Сохраняет аккаунт FunPay.

Параметры:
- `golden_key` — cookie/ключ FunPay;
- `user_agent` — необязательно.

Пример:
```text
/accountset abcdef123456 Mozilla/5.0
```

### `/accountcheck`
Проверяет, что привязанный FunPay-аккаунт работает.

### `/listenon`
Включает слушатель событий FunPay.

### `/listenoff`
Выключает слушатель событий FunPay.

### `/plugins`
Показывает список плагинов. Через это меню пользователь может включать и выключать доступные плагины.

### `/grant <tg_user_id> <plugin> <days>`
Админ-команда. Выдает пользователю доступ к плагину без оплаты.

Параметры:
- `tg_user_id` — Telegram ID пользователя;
- `plugin` — код плагина;
- `days` — срок доступа в днях.

Пример:
```text
/grant 123456789 autostars 30
```

---

## 3. Реально поддерживаемые плагины

Рабочие плагины, которые есть в текущем коде:

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

Плагины, которые есть в compose/каталоге, но лучше не использовать для нормального теста:

- `autoreply`
- `ask_tg_username`
- `antifraud`
- `autodelivery`
- `autoreview`
- `priceguard`
- `dailysummary`
- `blacklist`
- `unpaidreminder`

---

## 4. Команды по рабочим плагинам

### 4.1 AutoStars (`autostars`)

Назначение: продажа Telegram Stars по лотам FunPay.

#### Команды

##### `/autostars_sources`
Показывает список источников отправки Stars.

##### `/autostars_lots`
Показывает список лотов AutoStars.

##### `/autostars_addlot <lot_id> <stars_per_unit> [source_id]`
Добавляет или обновляет лот.

Параметры:
- `lot_id` — ID лота FunPay;
- `stars_per_unit` — сколько Stars выдавать за 1 единицу;
- `source_id` — необязательно, по умолчанию `default`.

Пример:
```text
/autostars_addlot 123456 100
/autostars_addlot 123456 100 reserve1
```

##### `/autostars_dellot <lot_id>`
Удаляет привязку лота.

##### `/autostars_source_http_add <source_id> <base_url> <token> [daily_limit_stars] [max_stars_per_tx] [timeout_sec] [min_balance_alert]`
Добавляет резервный HTTP-source.

Пример:
```text
/autostars_source_http_add reserve1 https://provider.example/api TOKEN 0 0 20 0
```

##### `/autostars_source_del <source_id>`
Удаляет source.

##### `/autostars_request_text <текст>`
Меняет текст запроса `@username` у покупателя.

##### `/autostars_delivered_text <текст>`
Меняет текст успешной доставки.

Поддерживаются шаблоны:
- `{username}`
- `{stars}`
- `{tx}`

Пример:
```text
/autostars_delivered_text Stars отправлены: {stars}, username: {username}, tx: {tx}
```

#### Минимальный тест
1. Выдать доступ к `autostars`.
2. Включить плагин в `/plugins`.
3. Настроить лот:
```text
/autostars_addlot 123456 100
```
4. Сделать тестовый заказ.
5. Проверить логи `plugin_autostars` и `autostars_provider_http`.

---

### 4.2 Fragment Stars Buyer (`fragment_stars_buyer`)

Назначение: автоматическая закупка Stars по стратегии.

#### Команды

##### `/fragment_help`
Справка.

##### `/fragment_status`
Показывает текущий статус стратегии.

##### `/fragment_strategy_on <price_ton_per_star> <buy_stars> [min_interval_sec] [max_inventory_stars] [max_total_ton]`
Включает стратегию.

Пример:
```text
/fragment_strategy_on 0.015 50 300 1000 10
```

##### `/fragment_strategy_off`
Выключает стратегию.

##### `/fragment_risk_set <daily_limit_ton> <slippage_bps> <stoploss_window_sec> <stoploss_drop_percent> <halt_minutes>`
Настраивает риск-параметры.

Пример:
```text
/fragment_risk_set 10 300 3600 5 60
```

##### `/fragment_provider <mock|http>`
Выбирает тип провайдера.

##### `/fragment_http_set <base_url> <token> [timeout_sec]`
Сохраняет HTTP-провайдер.

##### `/fragment_http_clear`
Очищает HTTP-настройки.

##### `/fragment_halt_clear`
Снимает halt.

##### `/fragment_mock_setprice <ton_per_star>`
Задает mock-цену.

##### `/fragment_mock_setavail <stars>`
Задает доступный объем Stars в mock-режиме.

##### `/fragment_inventory_reset`
Сбрасывает inventory и историю.

#### Минимальный тест без реального провайдера
```text
/fragment_provider mock
/fragment_mock_setprice 0.010
/fragment_mock_setavail 1000
/fragment_strategy_on 0.015 50 60 500 5
/fragment_status
```

---

### 4.3 AutoSteamPoints (`autosteampoints`)

Назначение: обработка заказов Steam Points через внешний HTTP provider.

#### Команды

##### `/autosteampoints_help`
Справка.

##### `/autosteampoints_cfg`
Текущий конфиг.

##### `/autosteampoints_http_set <base_url> <token> [timeout_sec]`
Настраивает провайдер.

Пример:
```text
/autosteampoints_http_set https://provider.example/api TOKEN 25
```

##### `/autosteampoints_lots`
Показывает список лотов.

##### `/autosteampoints_lots_add <lot_id> <points_per_unit>`
Добавляет или обновляет лот.

Пример:
```text
/autosteampoints_lots_add 123456 100
```

##### `/autosteampoints_lots_del <lot_id>`
Удаляет лот.

##### `/autosteampoints_sla <seconds>`
Меняет SLA ожидания.

##### `/autosteampoints_replay`
Повторяет задачи в `WAITING_STOCK`.

#### Минимальный тест
1. Выдать доступ к `autosteampoints`.
2. Включить плагин.
3. Сохранить provider.
4. Привязать лот.
5. Сделать тестовый заказ.
6. Проверить `plugin_autosteampoints`.

---

### 4.4 Steam Gifts Auto Lot (`steamgifts_autolot`)

Назначение: синхронизация и автодоставка Steam Gift.

#### Команды

##### `/steamgifts_help`
Справка.

##### `/steamgifts_cfg`
Показывает конфиг.

##### `/steamgifts_provider http`
Задает тип провайдера.

##### `/steamgifts_http_set <base_url> <token> [timeout_sec]`
Настраивает HTTP provider.

##### `/steamgifts_http_clear`
Очищает настройки provider.

##### `/steamgifts_sync_interval <sec>`
Интервал синхронизации.

##### `/steamgifts_markup_default <pct>`
Дефолтная наценка.

##### `/steamgifts_round_step <step>`
Шаг округления цены.

##### `/steamgifts_lots`
Показывает маппинги `lot_id -> sku`.

##### `/steamgifts_lots_add <lot_id> <ns_sku> [allowed_regions_csv] [markup_pct]`
Добавляет или обновляет лот.

Пример:
```text
/steamgifts_lots_add 123456 GAME_SKU RU,KZ 10
```

##### `/steamgifts_lots_del <lot_id>`
Удаляет лот.

##### `/steamgifts_replay`
Повторяет задачи в `WAITING_STOCK`.

---

### 4.5 AutoRent Steam (`autorent_steam`)

Назначение: автоматическая аренда Steam-аккаунтов.

#### Команды

##### `/autorent_help`
Справка.

##### `/autorent_cfg`
Показывает конфиг.

##### `/autorent_lots`
Показывает маппинги лотов.

##### `/autorent_lots_add <lot_id> <hours> <game_key>`
Добавляет лот аренды.

Пример:
```text
/autorent_lots_add 123456 24 cs2
```

##### `/autorent_lots_del <lot_id>`
Удаляет лот.

##### `/autorent_warn <seconds>`
Настраивает предупреждение перед окончанием аренды.

##### `/autorent_default_duration <seconds>`
Длительность аренды по умолчанию.

#### Управление аккаунтами

##### `/autorent_account_add <login> <password> <game_key1,game_key2,...>`
Добавляет аккаунт в пул.

Пример:
```text
/autorent_account_add mylogin mypassword cs2,dota2
```

##### `/autorent_accounts`
Показывает аккаунты.

##### `/autorent_account_disable <id>`
Отключает аккаунт.

##### `/autorent_account_enable <id>`
Включает аккаунт.

##### `/autorent_account_del <id>`
Удаляет аккаунт.

#### Ротация

##### `/autorent_rotator_off`
Отключает rotator.

##### `/autorent_rotator_http_set <base_url> <token>`
Подключает HTTP rotator.

---

### 4.6 AutoBalance (`autobalance`)

Назначение: следит за балансами и управляет лотами.

#### Команды

##### `/autobalance_help`
Справка.

##### `/autobalance_cfg`
Показывает конфиг.

##### `/autobalance_status`
Показывает статусы балансов.

##### `/autobalance_poll <seconds>`
Интервал опроса.

##### `/autobalance_provider_http_add <provider_id> <name> <url> <token> [json_path] [timeout_sec]`
Добавляет HTTP provider.

##### `/autobalance_provider_mock_add <provider_id> <name> <balance>`
Добавляет mock provider.

Пример:
```text
/autobalance_provider_mock_add stars Stars 1000
```

##### `/autobalance_provider_del <provider_id>`
Удаляет provider.

##### `/autobalance_resource_set <resource_key> <provider_id> <low> <high> [unit]`
Создает ресурс с порогами.

Пример:
```text
/autobalance_resource_set stars stars 500 700 Stars
```

##### `/autobalance_resource_del <resource_key>`
Удаляет ресурс.

##### `/autobalance_resource_lots_add <resource_key> <lot_id1> [lot_id2 ...]`
Привязывает лоты к ресурсу.

##### `/autobalance_resource_lots_del <resource_key> <lot_id1> [lot_id2 ...]`
Отвязывает лоты.

#### Минимальный безопасный тест
```text
/autobalance_provider_mock_add stars Stars 1000
/autobalance_resource_set stars stars 500 700 Stars
/autobalance_resource_lots_add stars 123456
/autobalance_status
```

---

### 4.7 AutoLotDeactivator (`autolotdeactivator`)

Назначение: переключение лотов и менеджер удаления.

#### Команды

##### `/autolot_help`
Справка.

##### `/autolot_cfg`
Показывает конфиг.

##### `/autolot_mode auto|notify`
Выбирает режим:
- `auto`
- `notify`

##### `/autolot_lot_autooff <lot_id>`
Исключает лот из авто-переключения.

##### `/autolot_lot_autoon <lot_id>`
Возвращает авто-переключение.

##### `/autolot_lot_list`
Показывает список исключений.

##### `/dlm`
Открывает Delete Lots Manager.

##### `/lots_manager`
То же самое.

---

### 4.8 AutoLog (`autolog`)

Назначение: логирование и статистика.

#### Команды

##### `/autolog_stats`
Статистика за день.

##### `/autolog_stats week`
Статистика за неделю.

##### `/autolog_stats month`
Статистика за месяц.

##### `/autolog_export`
Экспорт по умолчанию.

##### `/autolog_export csv week`
Экспорт CSV за неделю.

##### `/autolog_export xlsx month`
Экспорт XLSX за месяц.

---

### 4.9 AutoPriceMonitor (`autopricemonitor`)

Назначение: мониторинг цен конкурентов.

#### Команды

##### `/apm_help`
Справка.

##### `/apm_cfg`
Текущий конфиг.

##### `/apm_poll <seconds>`
Интервал опроса.

##### `/apm_add_url <name> <url> <my_lot_id> <my_price> [strategy]`
Добавляет цель по URL.

Пример:
```text
/apm_add_url Robux100 https://funpay.com/chips/123/ 999999 100 notify_only
```

##### `/apm_add_category <name> <chips|lots> <category_id> <my_lot_id> <my_price> [strategy]`
Добавляет цель по категории.

Пример:
```text
/apm_add_category Robux100 chips 123 999999 100 notify_only
```

##### `/apm_status`
Показывает список целей.

##### `/apm_status <target_id>`
Подробный статус по цели.

##### `/apm_enable <target_id>`
Включает цель.

##### `/apm_disable <target_id>`
Выключает цель.

##### `/apm_del <target_id>`
Удаляет цель.

##### `/apm_set <target_id> <field> <value>`
Меняет параметр цели.

Поддерживаемые поля:
- `name`
- `url`
- `my_lot_id`
- `my_price`
- `strategy`
- `undercut_step`
- `floor_price`
- `max_down_pct`
- `max_changes_per_day`
- `min_minutes_between_changes`
- `min_reviews`

Примеры:
```text
/apm_set 1a2b3c strategy aggressive
/apm_set 1a2b3c floor_price 95
/apm_set 1a2b3c undercut_step 1
```

##### `/apm_keywords <target_id> ...`
Задает ключевые слова.

##### `/apm_ignore_add <target_id> <seller1> [seller2 ...]`
Добавляет игнорируемых продавцов.

##### `/apm_ignore_del <target_id> <seller1> [seller2 ...]`
Удаляет из ignore list.

---

### 4.10 AutoRefund (`autorefund`)

Назначение: автоматическая реакция на споры и возвраты.

Важно:
- отдельного набора Telegram-команд для конфигурации сейчас нет;
- плагин настраивается через внутренний конфиг;
- тестировать лучше по логам.

---

## 5. Как тестировать бота и плагины

### Вариант A — быстрый smoke test
Подходит для проверки, что бот живой.

Действия тестера:
```text
/start
/plugins
```

Проверяется:
- бот отвечает;
- открывается меню;
- видны плагины.

---

### Вариант B — нормальный тест с FunPay
1. `/start`
2. `/accountset <golden_key>`
3. `/accountcheck`
4. `/listenon`
5. получить от администратора доступ к плагину через `/grant`
6. открыть `/plugins`
7. включить нужный плагин
8. настроить его командами
9. сделать тестовый заказ

---

### Вариант C — безопасный тест без реальных действий
В `.env` можно выставить:
```text
DRY_RUN=1
```

Тогда логика будет выполняться, но реальные действия в FunPay не будут отправляться.

---

## 6. Как добавлять пользователей без подписки

### Основной способ — через `/grant`

#### Шаг 1. Убедиться, что ты админ
В `.env` должен быть указан твой Telegram ID:
```text
CONTROL_ADMIN_IDS_JSON=[твой_id]
```

#### Шаг 2. Узнать Telegram ID тестера
Нужен числовой `tg_user_id`.

#### Шаг 3. Выдать доступ
Пример:
```text
/grant 123456789 autostars 30
```

---

## 7. Готовый набор команд для выдачи доступа ко всем рабочим плагинам

Если Telegram ID тестера `123456789`:

```text
/grant 123456789 autostars 365
/grant 123456789 autobalance 365
/grant 123456789 autolotdeactivator 365
/grant 123456789 autosteampoints 365
/grant 123456789 autorent_steam 365
/grant 123456789 fragment_stars_buyer 365
/grant 123456789 steamgifts_autolot 365
/grant 123456789 autorefund 365
/grant 123456789 autolog 365
/grant 123456789 autopricemonitor 365
```

После этого пользователь должен:
1. открыть `/plugins`;
2. вручную включить нужные плагины.

---

## 8. Коды плагинов для `/grant`

Используй только эти коды:

```text
autostars
autobalance
autolotdeactivator
autosteampoints
autorent_steam
fragment_stars_buyer
steamgifts_autolot
autorefund
autolog
autopricemonitor
```

---

## 9. Рекомендуемый сценарий тестирования

### Только бот
```text
/start
/plugins
```

### Бот + FunPay
```text
/start
/accountset <golden_key>
/accountcheck
/listenon
```

### Бот + конкретный плагин
1. администратор выдает доступ через `/grant`;
2. пользователь включает плагин в `/plugins`;
3. пользователь настраивает плагин командами;
4. выполняется тестовый сценарий;
5. администратор проверяет логи.

---

## 10. Где смотреть логи

Из папки `infra`:

```powershell
docker compose logs -f control_bot
docker compose logs -f funpay_connector
docker compose logs -f plugin_router
docker compose logs -f outbox_dispatcher
docker compose logs -f plugin_autostars
docker compose logs -f plugin_autosteampoints
docker compose logs -f plugin_steamgifts_autolot
docker compose logs -f plugin_autorent_steam
docker compose logs -f plugin_fragment_stars_buyer
docker compose logs -f plugin_autobalance
docker compose logs -f plugin_autolotdeactivator
docker compose logs -f plugin_autolog
docker compose logs -f plugin_autopricemonitor
docker compose logs -f plugin_autorefund
```

---

## 11. Что лучше сделать перед тестами

1. Оставить только реально рабочие плагины.
2. Не выдавать тестерам доступ к legacy-плагинам.
3. Делать отдельный тестовый лот под каждый плагин.
4. Для фоновых сценариев сначала использовать:
   - `fragment_stars_buyer` в `mock`;
   - `autobalance` через `mock provider`;
   - `autopricemonitor` в `notify_only`.

---

## 12. Короткий сценарий для тестера

### Администратор
1. Выдает тестеру доступ через `/grant`.
2. Сообщает, какие плагины доступны.

### Тестер
1. `/start`
2. `/accountset <golden_key>`
3. `/accountcheck`
4. `/listenon`
5. `/plugins`
6. включает нужный плагин
7. настраивает его командами
8. делает тестовый заказ
