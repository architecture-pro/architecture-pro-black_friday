## Задание 10.1. Выбор данных для Cassandra

Для переноса в Cassandra выбраны:

- **История заказов** — большой объём данных, частое чтение по `user_id`, высокая масштабируемость
- **Корзины** — интенсивные операции чтения и записи, доступ по идентификатору владельца
- **Пользовательские сессии** — быстрый доступ по `session_id`, автоматическое удаление по TTL
- **Каталог товаров** — масштабируемое чтение карточек товаров по идентификатору

В MongoDB предлагается оставить создание заказов и списание товарных остатков, поскольку эти операции требуют более строгой согласованности и транзакционной обработки

**Cassandra** используется для высоконагруженных операций с предсказуемыми запросами, MongoDB — для критичных транзакционных операций, такой подход снижает риски нарушения целостности данных при масштабировании.

## Задание 10.2. Модель данных Cassandra

### 1. История заказов

```sql
CREATE TABLE order_history (
    user_id UUID,
    order_id UUID,
    created_at TIMESTAMP,
    total_amount DECIMAL,
    status TEXT,
    PRIMARY KEY ((user_id), created_at, order_id)
) WITH CLUSTERING ORDER BY (created_at DESC, order_id ASC);
```

**Partition Key:** `user_id`. **Clustering Keys:** `created_at`, `order_id`

Быстрое получение истории пользователя с сортировкой по дате

### 2. Корзины

```sql
CREATE TABLE carts (
    owner_key TEXT,
    cart_id UUID,
    items TEXT,
    status TEXT,
    updated_at TIMESTAMP,
    PRIMARY KEY ((owner_key), cart_id)
);
```

**Partition Key:** `owner_key`. **Clustering Key:** `cart_id`

Поиск корзин по пользователю или гостевой сессии

### 3. Сессии

```sql
CREATE TABLE sessions (
    session_id UUID PRIMARY KEY,
    user_id UUID,
    created_at TIMESTAMP
) WITH default_time_to_live = 86400;
```

**Partition Key:** `session_id`. Clustering Key отсутствует

Быстрый доступ к сессии, автоматическое удаление через 24 часа

### 4. Каталог товаров

```sql
CREATE TABLE product_catalog (
    product_id UUID PRIMARY KEY,
    name TEXT,
    category TEXT,
    price DECIMAL,
    attributes MAP<TEXT, TEXT>
);
```

**Partition Key:** `product_id`. Clustering Key отсутствует

Быстрое чтение карточки товара по идентификатору

## Задание 10.3. Стратегии восстановления согласованности Cassandra

### Выбор стратегий

| Сущность          | Стратегия                            | Обоснование                                                     |
| ----------------- | ------------------------------------ | --------------------------------------------------------------- |
| `order_history`   | Hinted Handoff + Anti-Entropy Repair | Сохранность истории, допустима небольшая задержка синхронизации |
| `carts`           | Hinted Handoff + Read Repair         | Частые обновления, необходимо получать актуальные данные        |
| `sessions`        | Hinted Handoff                       | Приоритет — низкая задержка, данные временные                   |
| `product_catalog` | Read Repair + Anti-Entropy Repair    | Важно устранять расхождения цен и описаний товаров              |

### Уровни согласованности

- **LOCAL_QUORUM** — для истории заказов, корзин и каталога, когда важна согласованность внутри дата-центра
- **LOCAL_ONE** — для сессий, когда приоритетом является минимальная задержка
