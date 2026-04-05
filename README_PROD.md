# FunPay MVP: локальный запуск на Windows + WSL Ubuntu

Этот файл переписан под твой реальный сценарий:

- Windows
- WSL Ubuntu
- проект лежит внутри Linux-файловой системы WSL
- запуск идет из каталога `/home/max/funpay-mvp`

Если нужен полный разбор архитектуры и всех плагинов, смотри [RUNBOOK.md](/home/max/funpay-mvp/RUNBOOK.md).


## 1. Откуда запускать проект

Твой проект находится в WSL по пути:

```bash
/home/max/funpay-mvp
```

Из Windows он виден как:

```text
\\wsl.localhost\Ubuntu\home\max\funpay-mvp
```

Запускать проект лучше именно из WSL, а не из Windows PowerShell по UNC-пути.

Правильный вариант:

1. Открыть Windows Terminal
2. Зайти в профиль Ubuntu
3. Выполнить:

```bash
cd /home/max/funpay-mvp
```


## 2. Что должно быть установлено на локальной машине

### Обязательно

- WSL Ubuntu
- Docker Desktop на Windows
- включенная интеграция Docker Desktop с WSL

### Что проверить

Внутри WSL выполни:

```bash
docker --version
docker compose version
```

Если команды не работают внутри Ubuntu, проверь в Docker Desktop:

1. `Settings`
2. `Resources`
3. `WSL Integration`
4. включена интеграция для твоего Ubuntu-дистрибутива


## 3. Какие файлы использовать

Для запуска используй именно:

```text
infra/docker-compose.prod.yml
```

Почему именно его:

- он ближе к текущему состоянию проекта;
- в нем перечислены реальные основные сервисы;
- он лучше подходит для твоего локального запуска, чем старый `infra/docker-compose.yml`.


## 4. Подготовка `.env`

Из корня проекта:

```bash
cd /home/max/funpay-mvp
cp .env.example .env
```

Теперь открой `.env` и заполни минимум:

- `TG_BOT_TOKEN`
- `CONTROL_ADMIN_IDS_JSON`
- `POSTGRES_PASSWORD`
- `RABBITMQ_PASS`
- `MASTER_KEY_B64`
- `PG_DSN`
- `AMQP_URL`

### Очень важный момент

Если ты меняешь пароль Postgres или RabbitMQ, нужно синхронно поменять его и внутри DSN/URL.

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


## 5. Как сгенерировать `MASTER_KEY_B64` в WSL

В Ubuntu:

```bash
python - <<'PY'
import os, base64
print(base64.b64encode(os.urandom(32)).decode())
PY
```

Вставь результат в:

```env
MASTER_KEY_B64=...
```

Этот ключ нужен для шифрования:

- `golden_key`
- токенов провайдеров
- паролей Steam-аккаунтов

Если потом поменять `MASTER_KEY_B64`, старые зашифрованные данные перестанут читаться.


## 6. Минимально важные переменные `.env`

### Telegram

```env
TG_BOT_TOKEN=
CONTROL_ADMIN_IDS_JSON=[123456789]
```

`CONTROL_ADMIN_IDS_JSON` — это Telegram user id админов, которые смогут использовать `/grant`.

### База

```env
POSTGRES_USER=funpay
POSTGRES_PASSWORD=
POSTGRES_DB=funpay_mvp
PG_DSN=postgresql://funpay:CHANGE_ME@postgres:5432/funpay_mvp
```

### RabbitMQ

```env
RABBITMQ_USER=funpay
RABBITMQ_PASS=
AMQP_URL=amqp://funpay:CHANGE_ME@rabbitmq:5672/
```

### Общие

```env
MASTER_KEY_B64=
LOG_LEVEL=INFO
APP_TIMEZONE=Europe/Moscow
DRY_RUN=0
```

### Если хочешь начать безопасно

Для первого локального запуска можно поставить:

```env
DRY_RUN=1
```

Тогда `outbox_dispatcher` не будет реально отправлять действия в FunPay, а будет только логировать.


## 7. Как запускать проект локально

Из WSL:

```bash
cd /home/max/funpay-mvp/infra
docker compose -f docker-compose.prod.yml up -d --build
```

Если ты уже находишься в корне проекта:

```bash
cd infra
docker compose -f docker-compose.prod.yml up -d --build
```


## 8. Как проверить, что все поднялось

Выполни:

```bash
cd /home/max/funpay-mvp/infra
docker compose -f docker-compose.prod.yml ps
```

Ты должен увидеть основные контейнеры:

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


## 9. Как остановить проект

```bash
cd /home/max/funpay-mvp/infra
docker compose -f docker-compose.prod.yml down
```


## 10. Как перезапустить после изменений

```bash
cd /home/max/funpay-mvp/infra
docker compose -f docker-compose.prod.yml up -d --build
```


## 11. Где смотреть логи на локальной машине

### Общие сервисы

```bash
docker logs -f funpay_mvp_control_bot
docker logs -f funpay_mvp_funpay_connector
docker logs -f funpay_mvp_plugin_router
docker logs -f funpay_mvp_outbox_dispatcher
docker logs -f funpay_mvp_entitlement_enforcer
```

### Плагины

```bash
docker logs -f funpay_mvp_plugin_autostars
docker logs -f funpay_mvp_plugin_autosteampoints
docker logs -f funpay_mvp_plugin_steamgifts_autolot
docker logs -f funpay_mvp_plugin_autorent_steam
docker logs -f funpay_mvp_plugin_autorefund
docker logs -f funpay_mvp_plugin_fragment_stars_buyer
docker logs -f funpay_mvp_plugin_autolog
docker logs -f funpay_mvp_plugin_autopricemonitor
```


## 12. Первый запуск после старта контейнеров

После того как контейнеры поднялись:

1. Открой Telegram-бота
2. Выполни:

```text
/start
```

3. Сохрани FunPay аккаунт:

```text
/accountset <golden_key> [user_agent]
```

4. Проверь аккаунт:

```text
/accountcheck
```

5. Включи слушатель:

```text
/listenon
```

Если платежи еще не настроены, выдай себе доступ к плагинам через admin-команду:

```text
/grant <tg_user_id> <plugin> <days>
```

Пример:

```text
/grant 123456789 autostars 30
```


## 13. Что важно именно для твоего Windows + WSL сценария

### 1. Не запускай compose из Windows-каталога `\\wsl.localhost\...`

Нужно заходить в Ubuntu и запускать команды там.

### 2. Не редактируй `.env` из нескольких мест одновременно

Лучше редактировать либо:

- из редактора, открытого в WSL;
- либо аккуратно из Windows, но без параллельной работы в Linux-shell.

### 3. Если Docker внутри WSL не виден

Проблема почти всегда в Docker Desktop WSL integration.

### 4. Если контейнеры стартуют, но бот молчит

Смотри:

- `funpay_mvp_control_bot`
- `funpay_mvp_outbox_dispatcher`

### 5. Если бот отвечает, но FunPay не слушается

Смотри:

- `/accountcheck`
- `/listenon`
- лог `funpay_mvp_funpay_connector`


## 14. Почему раньше `README_PROD.md` заканчивался на строке про HTTP/Webhook

Потому что файл у тебя фактически был очень короткий и заканчивался именно там. Это был не "обрыв отображения", а просто старый укороченный текст без локального runbook.

Сейчас файл переписан и содержит полноценную инструкцию именно под локальный запуск на Windows + WSL Ubuntu.


## 15. Что еще стоит знать

Для полного описания:

- как использовать бота;
- какие плагины реально рабочие;
- какие плагины сейчас только частично подключены;
- какие есть ограничения в коде;

смотри:

[RUNBOOK.md](/home/max/funpay-mvp/RUNBOOK.md)

