Отличный вопрос! Это то, с чем сталкивается каждый администратор PostgreSQL. В отличие от Oracle, в PostgreSQL многие вещи делаются через системные представления (views), а не динамические performance-views. Объясню подробно.

### Философский подход

В PostgreSQL вся информация о текущей активности находится в системных каталогах (`pg_catalog`) и представлениях статистики (`pg_stat_*`). Для доступа к этой информации обычно нужны права суперпользователя или роль с соответствующими привилегиями (например, `pg_monitor` в версиях 10+).

---

### Ключевые команды и представления

Вот основные инструменты для проверки активности, от простого к сложному.

#### 1. Быстрый взгляд: утилита `pg_activity` (лучше `top` для PostgreSQL)

Это не встроенная команда, а отдельная утилита, которую **очень рекомендуется** установить. Она похожа на `htop` для вашей БД.

*   **Установка:** (например, на Ubuntu/Debian)
    ```bash
    sudo apt-get install pgtop
    ```
    Или через pip:
    ```bash
    pip install pg_activity
    ```

*   **Запуск:**
    ```bash
    pg_activity -U postgres -d mydb
    ```
    Вы увидите в реальном времени:
    *   Процессы (сессии) PostgreSQL.
    *   Текущий выполняемый SQL-запрос для каждого процесса.
    *   Процент использования CPU и I/O каждым процессом.
    *   Время выполнения запроса.
    *   Состояние процесса (`active`, `idle`, `idle in transaction`).

**Это часто самый быстрый и наглядный способ понять, что происходит.**

#### 2. Стандартные SQL-запросы к системным представлениям

Если `pg_activity` нет под рукой, используйте эти запросы.

**A. Просмотр всех текущих подключений и их запросов**

Основное представление — **`pg_stat_activity`**. Это аналог `V$SESSION` в Oracle.

```sql
-- Посмотреть все активные запросы (самый важный запрос)
SELECT
    pid,                    -- ID процесса (аналог SID в Oracle)
    usename,                -- Имя пользователя
    application_name,       -- Имя приложения (например, psql, pgAdmin, имя app-сервера)
    client_addr,            -- IP-адрес клиента
    state,                  -- Состояние: active, idle, idle in transaction
    backend_type,           -- Тип: client backend, autovacuum worker, etc.
    query,                  -- Текст текущего или последнего запроса
    query_start,            -- Время начала выполнения запроса
    age(now(), query_start) AS duration -- Длительность выполнения
FROM pg_stat_activity
WHERE state = 'active'      -- Показывать только выполняющиеся запросы
ORDER BY duration DESC;
```

**Б. Поиск "долгих" запросов**

Этот запрос помогает сразу найти потенциальные проблемы.

```sql
-- Найти все запросы, выполняющиеся дольше 5 минут
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    query,
    query_start,
    age(now(), query_start) AS duration
FROM pg_stat_activity
WHERE state = 'active'
  AND now() - query_start > interval '5 minutes'
ORDER BY duration DESC;
```

**B. Показать все подключения (включая бездействующие)**

Просто чтобы понять общую нагрузку по подключениям.

```sql
-- Общая статистика по состояниям подключений
SELECT state, count(*), application_name
FROM pg_stat_activity
GROUP BY state, application_name
ORDER BY count DESC;

-- Или просто список всех подключений
SELECT datname, usename, application_name, state, count(*)
FROM pg_stat_activity
GROUP BY datname, usename, application_name, state;
```

#### 3. Анализ блокировок (кто кого блокирует)

Как и в любой БД, в PostgreSQL могут быть блокировки. Для их анализа нужны более сложные запросы.

```sql
-- Классический запрос для поиска блокировок
SELECT
    -- Информация о блокирующем процессе (blocker)
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocked_activity.query AS blocked_statement,
    -- Информация о блокирующем процессе (blocker)
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocking_activity.query AS blocking_statement
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.DATABASE IS NOT DISTINCT FROM blocked_locks.DATABASE
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.GRANTED; -- Запрос ищет именно неудовлетворенные блокировки
```

#### 4. Просмотр ожиданий (Wait Events)

В современных версиях PostgreSQL (10+) появилась возможность смотреть, на чем именно ожидают процессы (аналог `V$SESSION_WAIT` в Oracle). Это крайне полезно для диагностики.

```sql
-- Показать активные процессы и их ожидания
SELECT
    pid,
    usename,
    query,
    state,
    wait_event_type, -- Тип ожидания: Lock, IO, LWLock, etc.
    wait_event       -- Конкретное событие: например, relation (ожидание блокировки таблицы)
FROM pg_stat_activity
WHERE wait_event is NOT NULL;
```

---

### Сводная таблица основных представлений

| Представление | Для чего используется? | Аналог в Oracle |
| :--- | :--- | :--- |
| **`pg_stat_activity`** | **Основное представление.** Показывает все сессии, их статус и текущий/последний запрос. | `V$SESSION`, `V$SQL` |
| **`pg_locks`** | Информация о текущих блокировках. | `V$LOCK` |
| **`pg_stat_database`** | Статистика на уровне БД: число транзакций, commits, rollbacks, темп чтения/записи. | `V$DATABASE` |
| **`pg_stat_all_tables`** | Статистика по таблицам: число последовательных сканирований, индексных сканирований, обновлений, удалений и т.д. | `V$SEGMENT_STATISTICS` |

### Итог для собеседования

"Для просмотра текущей активности в PostgreSQL я использую иерархический подход:

1.  **Для быстрого и наглядного анализа:** Я предпочитаю утилиту **`pg_activity`**, если она установлена. Это самый эффективный способ.
2.  **Для детального SQL-анализа:** Я запрашиваю системное представление **`pg_stat_activity`**, чтобы увидеть все активные запросы, их длительность и источник. Это основа.
3.  **Для диагностики проблем:** Если есть подозрение на блокировки, я использую сложный JOIN между **`pg_locks`** и **`pg_stat_activity`**, чтобы построить цепочку `blocker -> waiter`. Для поиска узких мест в производительности в современных версиях PG я смотрю на столбцы `wait_event_type` и `wait_event` в `pg_stat_activity`.

Главное отличие от Oracle — отсутствие единого мощного инструмента типа OEM "из коробки", но благодаря таким утилитам, как `pg_activity`, и хорошо структурированным системным представлениям, мониторинг в PostgreSQL очень эффективен."