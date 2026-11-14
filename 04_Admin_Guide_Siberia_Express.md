# Руководство по развертыванию и администрированию Siberia-Express Backend

**Версия продукта:** 4.2  
**Целевая аудитория:** DevOps-инженеры, системные администраторы  
**Назначение документа:** Регламент подготовки инфраструктуры, конфигурирования переменных окружения и запуска серверной части платформы доставки «Сибирь-Экспресс» в production- и staging-средах.

## Архитектура системы

Серверная часть **Siberia-Express Backend v4.2** построена на микросервисной архитектуре и поставляется в виде набора изолированных Docker-контейнеров:

- **API Gateway & Reverse Proxy (Nginx 1.25):** Терминация TLS/SSL, балансировка нагрузки, ограничение частоты запросов (Rate Limiting) и маршрутизация трафика.
- **Application Layer (Python 3.11 / FastAPI):** Асинхронные сервисы управления заказами, каталогом, групповыми корзинами, программой лояльности и интеграцией с платежными шлюзами.
- **Primary Database (PostgreSQL 16):** Основное реляционное хранилище транзакционных данных, профилей пользователей и складских остатков дарксторов.
- **In-Memory Store & Broker (Redis 7):** Кэширование сессий, хранение временных состояний групповых корзин в реальном времени и брокер сообщений для фоновых воркеров.

---
## 1. Системные и сетевые требования

### Аппаратные и программные требования

Для стабильной работы под нагрузкой (до 1 500 RPS) хост-машина должна соответствовать минимальным спецификациям, приведенным в таблице.

|Компонент|Минимальные требования (Staging)|Рекомендуемые требования (Production)|
|---|---|---|
|**CPU**|4 vCPU (x86_64)|8+ vCPU (x86_64, частота от 2.8 ГГц)|
|**RAM**|8 ГБ|16–32 ГБ ECC|
|**Дисковая подсистема**|60 ГБ SSD|200+ ГБ NVMe RAID-1 (минимум 3 000 IOPS)|
|**Операционная система**|Ubuntu 22.04 LTS / Astra Linux SE 1.7|Ubuntu 22.04.4 LTS / Astra Linux SE 1.7.5|
|**Контейнеризация**|Docker Engine 24.0+, Docker Compose v2.20+|Docker Engine 26.0+, Docker Compose v2.24+|

### Сетевые порты и политика безопасности (UFW)

Доступ к внутренним базам данных и кэшу из внешней сети должен быть строго заблокирован на уровне межсетевого экрана хоста (UFW) и облачных групп безопасности.

|Порт / Протокол|Сервис|Направление и доступ|Правило UFW|
|---|---|---|---|
|`80/tcp`|Nginx (HTTP)|Входящий (`0.0.0.0/0`) — редирект на 443|`ufw allow 80/tcp`|
|`443/tcp`|Nginx (HTTPS)|Входящий (`0.0.0.0/0`) — клиентский API|`ufw allow 443/tcp`|
|`22/tcp`|OpenSSH|Входящий (только из подсети VPN/Bastion)|`ufw allow from <ADMIN_SUBNET> to any port 22`|
|`5432/tcp`|PostgreSQL 16|Только внутренняя сеть Docker (`172.20.0.0/16`)|`ufw deny 5432/tcp`|
|`6379/tcp`|Redis 7|Только внутренняя сеть Docker (`172.20.0.0/16`)|`ufw deny 6379/tcp`|

---

## 2. Конфигурация окружения и развертывание через Docker Compose

Развертывание выполняется от имени пользователя, состоящего в группе `docker`, с правами на запись в директорию `/opt/siberia-express`.

### Шаг 1. Клонирование репозитория

Склонируйте репозиторий проекта и переключитесь на стабильный релизный тег версии `v4.2.0`:

bash

```
sudo mkdir -p /opt/siberia-express && sudo chown $USER:$USER /opt/siberia-express
git clone git@git.siberia-express.internal:backend/core-platform.git /opt/siberia-express
cd /opt/siberia-express
git checkout tags/v4.2.0 -b release-4.2
```
### Шаг 2. Настройка переменных окружения

Создайте рабочий файл `.env` на основе шаблона `.env.example` и ограничьте права доступа к нему:

bash

```
cp .env.example .env
chmod 600 .env
```

Сгенерируйте криптостойкий ключ для подписи JWT-токенов и заполните конфигурационный файл `.env` согласно эталонному листингу:

bash

```
openssl rand -hex 32
```

Пример валидного файла `.env`:

ini

```
# Application Settings
APP_ENV=production
APP_DEBUG=false
API_WORKERS=4

# PostgreSQL 16 Configuration
DB_HOST=postgres
DB_PORT=5432
DB_NAME=siberia_express_prod
DB_USER=se_backend_user
DB_PASSWORD=ReplaceWithStrongDbPassword98!

# Redis 7 Configuration
REDIS_URL=redis://:ReplaceWithStrongRedisPass77@redis:6379/0

# Security & Authentication
JWT_SECRET=9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30

# External Integrations
PAYMENT_GATEWAY_URL=https://acquiring-bank.ru
PAYMENT_GATEWAY_TOKEN=live_sk_8842a9c0e11b4f3d9a7c6e5b4a3d2c1f
SMS_PROVIDER_API_KEY=sms_prod_77129901ab
```

> [!CAUTION]  
> **Требования информационной безопасности при работе с секретами**  
> Категорически запрещается коммитить заполненный файл `.env`, приватные TLS-ключи и боевые токены платежного шлюза (`PAYMENT_GATEWAY_TOKEN`) в систему контроля версий Git. Убедитесь, что `.env` внесен в `.gitignore`.  
> Значение `JWT_SECRET` должно содержать не менее 32 байт энтропии, сгенерированных через CSPRNG (`openssl rand -hex 32`). Использование дефолтных или словарных значений приведет к компрометации сессий всех пользователей.

### Шаг 3. Сборка образов и запуск стека контейнеров

Запустите сборку актуальных образов микросервисов и инициализируйте инфраструктурный стек в фоновом режиме:

bash

```
docker compose -f docker-compose.prod.yml up -d --build
```

### Шаг 4. Применение миграций БД и проверка состояния (Healthcheck)

После перехода контейнеров в статус `healthy` выполните накатку актуальных миграций схемы базы данных (Alembic) и проверьте доступность API:

bash

```
docker compose -f docker-compose.prod.yml ps
docker compose -f docker-compose.prod.yml exec api alembic upgrade head
curl -sS -k https://localhost/api/v4/health | jq .
```

Ожидаемый ответ эндпоинта `/api/v4/health`:

json

```
{
  "status": "ok",
  "version": "4.2.0",
  "database": "connected",
  "redis": "connected"
}
```
## 3. Мониторинг, логирование и диагностика сбоев (Troubleshooting)

### Проверка состояния сервисов и просмотр логов

Для оперативного контроля статуса запущенных контейнеров и анализа журналов событий в реальном времени используйте следующие команды из директории `/opt/siberia-express`:

bash

```
# Проверка статуса, аптайма и состояния healthcheck всех контейнеров
docker compose -f docker-compose.prod.yml ps

# Мониторинг потребления CPU, RAM и сетевого I/O в реальном времени
docker stats --no-stream

# Потоковый вывод последних 200 строк логов основного API-сервиса
docker compose -f docker-compose.prod.yml logs -f --tail=200 api

# Вывод логов обратного прокси Nginx и СУБД PostgreSQL за последние 30 минут
docker compose -f docker-compose.prod.yml logs --since 30m nginx postgres
```
### Матрица диагностики типовых инцидентов

|Симптом / Код ошибки|Вероятная причина|Команда / Действие для устранения|
|---|---|---|
|**HTTP `502 Bad Gateway` на стороне Nginx**|Аварийное завершение (OOM Kill) или перезапуск воркеров Uvicorn/FastAPI; несоответствие upstream-адреса.|1. Проверить статус и exit-код контейнера:  <br>`docker compose -f docker-compose.prod.yml ps api`  <br>2. Найти причину падения в логах:  <br>`docker compose -f docker-compose.prod.yml logs --tail=100 api`  <br>3. Принудительно перезапустить сервис:  <br>`docker compose -f docker-compose.prod.yml restart api`|
|**Ошибка `Connection refused` при обращении к БД (`5432`)**|Контейнер `postgres` не запущен, находится в стадии инициализации или переполнен пул соединений (`max_connections`).|1. Проверить готовность СУБД к приему подключений:  <br>`docker compose -f docker-compose.prod.yml exec postgres pg_isready -U se_backend_user`  <br>2. Проверить наличие свободного места на диске с volume БД:  <br>`df -h /var/lib/docker`  <br>3. Перезапустить контейнер БД при зависании:  <br>`docker compose -f docker-compose.prod.yml restart postgres`|
|**Таймаут (`504`) или `401/403` от платежного шлюза при оплате**|Истек срок действия или отозван `PAYMENT_GATEWAY_TOKEN`; исходящий трафик на порт `443` заблокирован фаерволом.|1. Проверить сетевую связность изнутри контейнера API:  <br>`docker compose -f docker-compose.prod.yml exec api curl -I -m 5 https://acquiring-bank.ru`  <br>2. Обновить `PAYMENT_GATEWAY_TOKEN` в файле `/opt/siberia-express/.env`.  <br>3. Пересоздать контейнер для применения новых переменных:  <br>`docker compose -f docker-compose.prod.yml up -d --force-recreate api`|

---
## 4. Резервное копирование и обновление (Backup & Maintenance)

### Создание резервной копии PostgreSQL

Перед выполнением любых регламентных работ или обновлением релиза необходимо создать консистентный дамп основной базы данных в сжатом формате (`gzip`):

bash

```
mkdir -p /var/backups/siberia-express
docker compose -f /opt/siberia-express/docker-compose.prod.yml exec -T postgres \
  pg_dump -U se_backend_user -d siberia_express_prod --clean --if-exists | \
  gzip > /var/backups/siberia-express/db_backup_$(date +%Y%m%d_%H%M%S).sql.gz
```

Для проверки целостности созданного архива выполните тест распаковки без извлечения файлов:

bash

```
gzip -t /var/backups/siberia-express/db_backup_*.sql.gz && echo "Backup archive OK"
```

### Регламент безопасного обновления версии

Обновление выполняется без удаления постоянных томов (`docker volumes`), что гарантирует сохранность данных PostgreSQL и Redis:

1. **Создайте резервную копию БД** по инструкции выше.
2. **Получите актуальный код** из репозитория и переключитесь на целевой тег релиза:

bash

```
cd /opt/siberia-express
git fetch --all --tags
git checkout tags/v4.2.1 -b release-4.2.1
```

3. **Сверьте переменные окружения:** сравните `.env` с обновленным `.env.example` на случай появления новых обязательных параметров:

bash

```
diff -u <(cut -d= -f1 .env | sort) <(cut -d= -f1 .env.example | sort)
```

4. **Пересоберите образы и перезапустите контейнеры** с минимальным временем простоя:

bash

```
docker compose -f docker-compose.prod.yml up -d --build --remove-orphans
```

5. **Примените миграции базы данных** и убедитесь в успешном ответе healthcheck:

bash

```
docker compose -f docker-compose.prod.yml exec api alembic upgrade head
curl -sS -k https://localhost/api/v4/health
```

> [!IMPORTANT]  
> **Автоматизация ночного резервного копирования**  
> Настройте ежедневное автоматическое создание дампов базы данных в период минимальной пользовательской активности (**03:00 по местному времени**) с помощью планировщика `cron` и ротацией архивов старше 14 дней.  
> Добавьте в `crontab -e` от имени пользователя с правами доступа к Docker следующую задачу:
> 
> cron
> 
> ```
> 0 3 * * * docker compose -f /opt/siberia-express/docker-compose.prod.yml exec -T postgres pg_dump -U se_backend_user -d siberia_express_prod | gzip > /var/backups/siberia-express/daily_$(date +\%Y\%m\%d).sql.gz && find /var/backups/siberia-express -name "daily_*.sql.gz" -mtime +14 -delete
> ```
> 
> Рекомендуется дополнительно настроить синхронизацию каталога `/var/backups/siberia-express` с внешним S3-совместимым объектным хранилищем.

---

#AdminGuide #DevOps #Docker #SystemAdministration #PostgreSQL #Backend #СибирьЭкспресс #DocsAsCode

