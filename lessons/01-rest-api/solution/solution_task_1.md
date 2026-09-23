# Решение: Система управления заказами

## 1. Диаграмма последовательности для GET /orders/{id}

```mermaid
sequenceDiagram
    participant Client as Клиент
    participant API as Order API
    participant DB as База данных
    
    Client->>API: GET /orders/{id}
    activate API
    
    API->>DB: SELECT * FROM orders WHERE id = {id}
    activate DB
    
    alt Заказ найден
        DB-->>API: Order data
    else Заказ не найден
        DB-->>API: null
    end
    
    deactivate DB
    
    alt Заказ найден
        API-->>Client: 200 OK + Order
    else Заказ не найден
        API-->>Client: 404 Not Found
    end
    
    deactivate API
```

## 2. Диаграмма последовательности для POST /orders

```mermaid
sequenceDiagram
    participant Client as Клиент
    participant API as Order API
    participant DB as База данных
    participant Delivery as API Доставки
    
    activate API
    Client->>API: POST /orders {customer_name, items, address}
    
    API->>API: Валидация данных
    
    alt Данные невалидны
        API-->>Client: 400 Bad Request
    else Данные валидны
        activate DB
        API->>DB: INSERT INTO orders (status='pending')
        DB-->>API: order_id
        deactivate DB
        
        loop Retry до 3 раз
            alt Успех
                activate Delivery
                API->>Delivery: POST /reserve-slot
                Delivery-->>API: 200 OK {slot_id}
                deactivate Delivery
                
                activate DB
                API->>DB: UPDATE orders SET status='confirmed'
                DB-->>API: OK
                deactivate DB
            else Таймаут / Ошибка сети
                activate Delivery
                API->>Delivery: POST /reserve-slot
                Delivery-->>API: 504 Gateway Timeout
                deactivate Delivery
                
                Note over API: Ждём 1 сек перед retry
            end
        end
        
        alt Успешное резервирование
            API-->>Client: 201 Created {order_id, slot_id}
        else Все попытки провалились
            activate DB
            API->>DB: DELETE FROM orders WHERE id = order_id
            DB-->>API: OK
            deactivate DB
            API-->>Client: 503 Service Unavailable
        end
    end
    deactivate API
```

## 3. Таблица контрактов API

**GET /orders/{id}**
| Имя параметра | Тип данных | Обязательность | Описание |
| :--- | :--- | :--- | :--- |
| **Входные** |
| *id (path)* | *integer* | *Да* | *ID заказа* |
| *Выходные (200 OK)* |
| *order_id* | *integer* | *Да* | *ID заказа* |
| *customer_name* | *string* | *Да* | *Имя клиента* |
| *items* | *array* | *Да* | *Список товаров* |
| *status* | *string* | *Да* | *Статус заказа (pending/confirmed)* |
| *slot_id* | *string* | *Нет* | *ID слота доставки (если подтверждён)* |
| **Выходные  (404 Not Found)** |
| *error* | *integer* | *Да* | *Сообщение об ошибке ("Order not found")* |

**POST /orders**
| Имя параметра | Тип данных | Обязательность | Описание |
| :--- | :--- | :--- | :--- |
| **Входные** |
| *customer_name* | *string* | *Да* | *Имя клиента* |
| *items* | *array* | *Да* | *Список товаров* |
| *items[].product_id* | *integer* | *Да* | *ID товара* |
| *items* | *integer* | *Да* | *Количество* |
| *address* | *string* | *Да* | *Адрес доставки* |
| **Выходные (201 Created)** |
| *order_id* | *integer* | *Да* | *ID заказа* |
| *slot_id* | *string* | *Да* | *ID слота доставки* |
| *status* | *string* | *Да* | *Статус заказа* |
| **Выходные  (404 Not Found)** |
| *error* | *integer* | *Да* | *Сообщение об ошибке валидации* |
| **Выходные (503 Service Unavailable)** |
| *error* | *integer* | *Да* | *Сообщение об ошибке ("Delivery service unavailable")* |

## 4. Таблица маппинга

| Наше поле (Order API) | Поле API Доставки | Преобразование |
| :--- | :--- | :--- |
| *order_id* | *external_order_id* | *Конвертация int → string* |
| *address* | *delivery_address* | *Без изменений* |
| *items[].weight* | *total_weight* | *Сумма весов всех товаров (items × quantity)* |
| *customer_name* | *recipient_name* | *Без изменений* |
| *-* | *priority* | *По умолчанию: "standard"* |
| *-* | *request_id* | *Генерация нового UUID для идемпотентности* |
| *order_id* | *callback_url* | *Формирование URL* |