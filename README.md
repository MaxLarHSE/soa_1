# Marketplace HW (C4 + dockerized service)

## Описание
Проектирует архитектуру маркетплейса: продавцы размещают товары, пользователи получают персонализированную ленту и оформляют заказы, далее идут платежи и уведомления.  

В репозитории:
- C4 like Container диаграмма (запускается через докер)
- исходный код 1 сервиса + Dockerfile
- инструкция запуска и проверка `/health`

---

## Домены и ответственность
- **Gateway** - единая точка входа, маршрутизация запросов.
- **Catalog** - товары и управление каталогом продавца.
- **Feed** - выдача ленты товаров (персонализация упрощена).
- **Users** - покупатели/продавцы, регистрация/авторизация, профиль.
- **Orders** - оформление заказов и статусы.
- **Payments** - учёт платежей и статусов.
- **Notifications** - уведомления о статусах заказа/оплаты.

---

## Сервисы и API (распределение доменов по сервисам)

### API Gateway (`/api`)
- `GET /health`
- проксирует:
    - `/goods/**` → goods-service
    - `/feed/**` → feed-service
    - `/user/**` → user-service
    - `/order/**` → order-service
    - `/payment/**` → payment-service
    - `/notification/**` → notification-service

**DB:** нет.

---

### goods-service (`/goods`) - Catalog + Seller tools
- `GET /goods?filters...` - выдача товаров по фильтрам 
- `GET /goods/recomendation?id=...` -товары для персонализированной ленты 
- `/goods/saler`
    - `GET ?id_saler=...`
    - `POST` -добавить товар
    - `PUT /{id}` -изменить товар
    - `DELETE /{id}` -удалить товар

**DB (goods-db):** `products (seller_id, price, ...)`  
**Events:** публикует `ProductTouched` (создание/изменение/удаление товара).

---

### feed-service (`/feed`) - Feed
- `GET /feed?id=...` -выдача ленты товаров (по user_id)

**DB (feed-db):** id/ кеш предложений  
**Events:** слушает `ProductTouched` для обновления кэша.

---

### user-service (`/user`) - Users + Auth
- `POST /user/register` -создание пользователя
- `POST /user/login/{id}` -пароль в body
- `PUT /user/change` -изменить данные
- `DELETE /user/change` -удалить пользователя
- публикует `PersonCreated` 

**DB (user-db):** пользователи/профили + `password_hash`

---

### order-service (`/order`) - Orders
- `POST /order/buy`
    - sync: вызывает `POST /payment/buy` и слушает
    - при успехе создаёт заказ и публикует события `OrderCreated`/`OrderStatusChanged`
- `GET /order/list?id_user=...` -список заказов
- `GET /order/list?id_user=...&id=...` -детали заказа
- `DELETE /order/list?id_user=...&id=...` -удалить/отменить заказ, публикует событие

**DB (order-db):** `orders`  
**Events:** публикует `OrderCreated`, `OrderStatusChanged`  
**Events:** слушает `PaymentStatusChanged` и обновляет статус заказа.

---

### payment-service (`/payment`) - Payments
- `POST /payment/buy` -создать платёж (body: кто покупает, сумма и т.п.)
    - публикует `PaymentStatusChanged` 

**DB (payment-db):** `payments (status, order_id, ...)`  
**Events:** публикует `PaymentStatusChanged`

---

### notification-service (`/notification`) - Notifications
- `POST /notification/pingAll` -массовое уведомление (admin)
- `POST /notification/ping/{id}` -отправить/создать уведомление
- `PUT /notification/ping/{id}` -изменить
- `DELETE /notification/ping/{id}` -удалить

**DB (notification-db):** `notifications`  
**Events (consume):** слушает `PersonCreated`, `OrderCreated`, `OrderStatusChanged`, `PaymentStatusChanged`.

---
## Границы владения данными и события

### Владение данными (нет shared DB)

Каждый сервис владеет собственной базой данных и является единственным источником истины для своих данных. Другие сервисы не имеют прямого доступа к чужим БД и взаимодействуют только через API или события.

| Сервис | База данных | Владеет данными                                | Ответственность |
|---|---|------------------------------------------------|---|
| goods-service | goods-db | товары (`products`, `seller_id`, `price`)      | управление каталогом и товарами продавцов |
| feed-service | feed-db | персонализации (`id`, `cache`)                 | формирование персонализированной ленты |
| user-service | user-db | пользователи (`profiles`, `password_hash`)     | регистрация, авторизация, профиль пользователя |
| order-service | order-db | заказы (`orders`) | создание заказов и управление статусами |
| payment-service | payment-db | платежи (`payments`)     | учёт и статус платежей |
| notification-service | notification-db | уведомления (`notifications`)                  | отправка и хранение уведомлений |

Это гарантирует отсутствие shared database и чёткие границы владения данными.


## Взаимодействия сервисов (sync/async)

### Синхронно (HTTP)
- все что не асинхронно

### Асинхронно (events через Message Broker)
- goods-service → `ProductTouched` → feed-service
- order-service → `OrderCreated/OrderStatusChanged` → notification-service
- payment-service → `PaymentStatusChanged` → order-service, notification-service

**Важно:** базы данных не разделяются между сервисами.

---

## Почему выбран такой вариант
- **Минимальная связность:** сервисы не вызывают друг друга напрямую без необходимости.
- **Ясные границы данных:** каждый сервис владеет своей БД, нет shared DB.
- **Простая эволюция:** развивать компоненты системы независимо.

---

## Запуск сервиса в Docker (goods-service)

Сервис реализует health-check endpoint: `GET /health` → `200 OK`.

### Build
```bash
docker build -t goods-service ./services/goods
```
### RUN
```bash
docker run --rm -p 8080:8080 goods-service
```
### Check
```bash
curl -i http://localhost:8080/health
```
### Check 2... (BETTER)
```bash
curl.exe -i http://localhost:8080/health
```
### Build образа 
```bash
docker build -t likec4-viewer -f Dockerfile.likec4 .
```

### Запуск образа
```bash
docker run --rm -p 5173:5173 -p 24678:24678 -v ${PWD}:/workspace -w /workspace likec4-viewer
```




## Альтернативные варианты декомпозиции и trade-off’ы

### Вариант A - Монолит (1 сервис)
**Описание:** один backend-сервис реализует users/goods/feed/orders/payments/notifications, одна общая БД.  
**Плюсы:** максимально простой деплой и отладка на первых этапах разработки, минимум инфраструктуры.  
**Минусы:** размытые границы доменов и данных, общий релизный цикл, сложнее масштабировать и развивать персонализацию/уведомления отдельно.  
**Trade-off:** простота разработки <-> хуже масштабируемость и поддерживаемость.

### Вариант B - Service-Oriented Architecture (SOA) с общей DB-шиной
**Описание:** система состоит из отдельных сервисов (goods, feed, user, order, payment, notification), но все сервисы работают с общей базой данных (общая DB-шина). Сервисы логически разделены и могут запускаться отдельно, однако используют одно общее хранилище.  
**Плюсы:** 
- сервисы логически разделены по ответственности;  
- проще реализовать взаимодействие, так как все данные доступны в одной БД;  

**Минусы:** 
- меньшее разделение на зоны ответственности по сравнению с микросервисной архитектурой;
- сложнее масштабировать отдельные сервисы независимо.  

**Trade-off:** упрощение интеграции и инфраструктуры <-> потеря изоляции сервисов и масштабируемости.

### Вариант C - Выбранный (gateway + сервисы + broker + отдельные БД)
**Описание:** как в текущей архитектуре: API Gateway + goods/feed/user/order/payment/notification, асинхронные события через broker.  
**Плюсы:** минимальная связность (уведомления/лента по событиям), отказоустойчивость (сервис-подписчик может быть временно недоступен), понятные границы владения данными (нет shared DB).  
**Минусы:** больше инфраструктуры, сложнее писать.  
**Trade-off:** сложнее эксплуатация <-> лучше масштабируемость и устойчивость.

### Почему выбран вариант C
С точки зрения кейса это самый подходящий вариант (условия задачи - бизнес требования)  
Он напрямую покрывает требования кейса (каталог, лента, пользователи, заказы, платежи, уведомления) и соблюдает ограничение “нет shared DB”. Асинхронные события позволяют не связывать сервисы прямыми вызовами (особенно для уведомлений и обновления ленты), что делает архитектуру более устойчивой и расширяемой.
