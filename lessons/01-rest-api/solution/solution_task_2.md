# Решение: Корпоративный портал профилей

## 1.Диаграмма последовательности для GET /employees/profile

```mermaid
sequenceDiagram
    participant Client as Клиент
    participant Auth as Auth Middleware
    participant AuthDB as Auth DB
    participant Service as Employee Service
    participant ProfileDB as Profile DB
    
    activate Client
    activate Auth
    activate Service
    
    Client->>Auth: GET /employees/profile<br/>Authorization: Basic base64(login:password)
    
    Auth->>Auth: Декодирование Base64
    
    alt Заголовок отсутствует или неверный формат
        Auth-->>Client: 401 Unauthorized<br/>WWW-Authenticate: Basic realm="portal"
    else Формат верный
        alt Ошибка соединения с Auth DB
            activate AuthDB
            Auth->>AuthDB: Попытка соединения
            AuthDB-->>Auth: Connection timeout / Error
            deactivate AuthDB
            Auth-->>Client: 500 Internal Server Error
        else Соединение успешно
            activate AuthDB
            Auth->>AuthDB: SELECT password_hash FROM users WHERE login = ?
            AuthDB-->>Auth: password_hash или null
            deactivate AuthDB
            
            alt Логин не найден
                Auth-->>Client: 401 Unauthorized
            else Логин найден
                Auth->>Auth: Сравнение хэшей (bcrypt.verify)
                
                alt Пароль неверный
                    Auth-->>Client: 401 Unauthorized
                else Пароль верный
                    alt Ошибка соединения с Profile DB
                        activate ProfileDB
                        Auth->>Service: Передача запроса с employee_id
                        Service->>ProfileDB: Попытка соединения
                        ProfileDB-->>Service: Connection timeout / Error
                        deactivate ProfileDB
                        Service-->>Client: 500 Internal Server Error
                    else Соединение успешно
                        activate ProfileDB
                        Auth->>Service: Передача запроса с employee_id
                        Service->>ProfileDB: SELECT * FROM profiles WHERE employee_id = ?
                        ProfileDB-->>Service: profile_data или null
                        deactivate ProfileDB
                        
                        alt Профиль не найден
                            Service-->>Client: 404 Not Found
                        else Профиль найден
                            alt Ошибка при маппинге данных
                                Service->>Service: Exception при обработке
                                Service-->>Client: 500 Internal Server Error
                            else Маппинг успешен
                                Service->>Service: Маппинг данных
                                Service-->>Client: 200 OK + Profile
                            end
                        end
                    end
                end
            end
        end
    end
    
    deactivate Service
    deactivate Auth
    deactivate Client
```

## 2. Таблица контрактов API

### Входные параметры (Input)

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `Authorization` | string | Да | Заголовок авторизации |
| Содержимое заголовка (декодированное) | string | Да | `login:password` |
| `login` | string | Да | Логин сотрудника |
| `password` | string | Да | Пароль сотрудника |

### Выходные параметры (Output)

| HTTP-код | Параметр | Тип | Описание |
|---|---|---|---|
| **200 OK** | `id` | integer | ID сотрудника |
|  | `full_name` | string | ФИО |
|  | `email` | string | Корпоративная почта |
|  | `position` | string | Должность |
|  | `department_name` | string | Название отдела |
|  | `hire_date` | string (date) | Дата найма (YYYY-MM-DD) |
|  | `phone` | string | Рабочий телефон |
| **401 Unauthorized** | `error` | string | Сообщение об ошибке |
|  | `www_authenticate` | string | `Basic realm="portal"` |
| **404 Not Found** | `error` | string | "Profile not found" |
| **500 Internal Server Error** | `error` | string | Сообщение об ошибке |

## 3. Таблица маппинга данных

| Наше поле (Profile DB) | Поле API | Преобразование |
|---|---|---|
| `employee_id` | `id` | Переименование |
| `first_name` + `last_name` + `middle_name` | `full_name` | Объединение строк |
| `email` | `email` | Без изменений |
| `position_id` | `position` | JOIN со справочником `positions`: получаем `position_name` |
| `department_id` | `department_name` |JOIN со справочником `departments`: получаем `department_name` |
| `hire_date` | `hire_date` | Форматирование даты в `YYYY-MM-DD` |
| `phone` | `phone` | Без изменений |
| `password_hash` | НЕ возвращается | Удаляется из ответа |
| `is_active` | НЕ возвращается | Удаляется из ответа |