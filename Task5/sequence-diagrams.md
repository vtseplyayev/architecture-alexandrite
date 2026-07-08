# Task 5 — Sequence-диаграммы кеширования дашборда MES

Кеш: **Redis**, паттерн **Cache-Aside**, инвалидация — **программная (событийная) по версии статуса** + короткий TTL как страховка.
Ключ списка: `orders:list:{status}:v{version}:{cursor}`; версия статуса хранится в `orders:ver:{status}`.

## 1. Чтение списка заказов (Cache-Aside)

```mermaid
sequenceDiagram
    actor Operator as Оператор
    participant MES as MES (React)
    participant API as MES API (C#)
    participant Redis as Redis (кеш)
    participant DB as MES DB (PostgreSQL)

    Operator->>MES: Открывает дашборд (фильтр по статусу)
    MES->>API: GET /orders?status=MANUFACTURING_APPROVED&cursor=...
    API->>Redis: GET orders:ver:{status}
    Redis-->>API: version = N
    API->>Redis: GET orders:list:{status}:vN:{cursor}
    alt Кеш HIT
        Redis-->>API: список заказов (JSON)
        API-->>MES: 200 OK (из кеша, быстро)
    else Кеш MISS
        Redis-->>API: nil
        API->>DB: SELECT ... WHERE status=? ORDER BY created_at DESC (keyset)
        DB-->>API: строки заказов
        API->>Redis: SETEX orders:list:{status}:vN:{cursor} TTL=30s (список)
        API-->>MES: 200 OK (из БД + прогрев кеша)
    end
    MES-->>Operator: Список новых заказов сверху
```

## 2. Изменение статуса заказа (запись + инвалидация)

Источник изменения — либо действие оператора (взял/выполнил заказ), либо
сообщение из очереди (напр. PRICE_CALCULATED, MANUFACTURING_APPROVED из CRM).

```mermaid
sequenceDiagram
    participant Src as Источник изменения<br/>(оператор / consumer очереди)
    participant API as MES API (C#)
    participant DB as MES DB (PostgreSQL)
    participant MQ as RabbitMQ
    participant Redis as Redis (кеш)

    Src->>API: Сменить статус заказа (order_id: old→new)
    API->>DB: UPDATE orders SET status=new WHERE id=? (транзакция)
    DB-->>API: OK
    API->>MQ: publish order.status_changed (outbox)
    Note over API,Redis: Инвалидация затронутых списков по версии
    API->>Redis: INCR orders:ver:{old_status}
    API->>Redis: INCR orders:ver:{new_status}
    Note right of Redis: старые ключи orders:list:*:vN больше<br/>не читаются (version сдвинулась) и уйдут по TTL
    API-->>Src: OK (статус изменён)

    Note over Redis: Следующее чтение по new/old статусу → MISS →<br/>перечитывает свежие данные из БД и прогревает кеш
```

> Приём «version bump»: вместо удаления множества ключей пагинации инкрементируем
> счётчик версии статуса — все старые страницы этого статуса мгновенно
> «инвалидируются» без сканирования ключей Redis, а протухшие записи удаляются по TTL.
