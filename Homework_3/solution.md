## A1. Выбор архитектурного стиля
### Контекст
Существующее решение: монолит с тремя клиентами покупатель(мобилка, веб), продавец (веб).
Существующие компонтенты:
![alt text](image.png)

Необходимо встроить:

![alt text](image-1.png)

### Рассмотренные варианты
#### 1 кейс
- Доработать существующий монолит
- Выделить модуль Order & Fulfillment в отдельные сервисы

#### 2 кейс
- использовать event-driven подход
- использовать синхронные вызовы

### Решения и обоснования
#### 1 кейс
**Выделить модуль Order & Fulfillment в отдельные сервисы**
1. Ни один из новых сервисов не влияет на UI покупателя. Внедряется аналитическое хранилище, которое скорее всего повлечет за собой доработки кабинета продавца (UI веб сервиса продавца и BFF для него). Если встроить аналитическое хранилище в монолит, то оно будет замедлять сервисы для покупателей. Тем более, что по требованиям по обновлению данных у нас есть 5 минут для синхронизации, что позволит, например, отправлять события с данными для аналитики и сохранять их в отдельную денормализованную БД;
2. Появляется интеграция с внешними сервисами доставки. Эти сервисы могут меняться, могут (в теории) меняться контракты этих сервисов и тп. При необходимости доработать и обновить на продакшене небольшой сервис по интеграции с внешними доставками проще, чем целый монолит;
3. За эти сервисы отвечают разные команды.

#### 2 кейс
**использовать event-driven подход**
1. Сервисы уведомлений и доставки могут "в своем темпе" разбирать события по заказам;
2. Нет требований на немедленную актуализацию статуса доставки по заказу/отправки сообщения, соответственно нет смысла "заставлять" монолит ожидать синхронную обратную связь от сервисов;
Исключение: я бы сделала синхронным взаимодействие с InventoryService т.к. нам важно получать актуальные данные о наличии перед оформлением заказа и согласованность при резервировании.

### Последствия
#### 1 кейс
Из плюсов:
- Упростится масштабирование;
- Независимый деплой.

Из минусов:
- Усложнится инфраструктура (и поддержка этой инфраструктуры);
- Усложнится мониторинг и логирование.

#### 2 кейс
Из плюсов: 
- Повышение отказоустойчивости системы: при падении потребителей (в нашем случае DeliveryService и NotificationService) сообщения не потеряются, а просто будут отправлены позже, при восстановлении работоспособности;
- Уменьшение связаности;
- С точки зрения продюсера (нашего монолита): нет необходимости обращаться к разным сервисам напрямую, нет необходимости ждать ответа от сервисов.

Из минусов:
- Усложняется архитектура системы (как минимум, появятся очереди сообщений);
- Нужно позаботиться об идемпотентности, об мониторинге и логировании.

## A2. Три клиентских канала и BFF
BFF применяется только для мобильного приложения для ускорения загрузки и экономии траффика т.к. там часто встречаются интерфейсы с ограниченной функциональностью. 

Схема:
![alt text](schema.png)

```plantuml
@startuml
!NEW_C4_STYLE = 1
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

title МаркетХаб - диаграмма контейнеров (монолит)

Person(buyer, "Покупатель", "Просматривает каталог, оформляет заказы")
Person(seller, "Продавец", "Размещает товары, управляет ценами и остатками")

System_Ext(bank, "Платежный шлюз", "Обработка платежей")
System_Ext(delivery_cdek, "Службы доставки CDEK", "Доставка заказов покупателю")
System_Ext(delivery_boxberry, "Службы доставки Boxberry ", "Доставка заказов покупателю")
System_Ext(crm, "CRM", "Хранение профилей клиентов и истории обращений")
System_Ext(email_service, "Email-сервис", "Отправка email-писем")
System_Ext(push_service, "Push-сервис", "Отправка push-уведомлений")

Container_Boundary(marketplace, "МаркетХаб") {
    Container(web_app, "Веб-приложение_покупатель", "React, TypeScript", "Обеспечивает UI для покупателей и админов")
    Container(web_app_seller, "Веб-приложение_продавец", "React, TypeScript", "Обеспечивает UI для продавцов")
    Container(mobile_app, "Мобильное приложение_покупатель", "React Native, TypeScript", "Предоставляет интерфейс для покупателей")
    Container(bff_for_mobile_app, "BFF для мобильного приложения покупателя", "Node.js, TypeScript", "Агрегирует данные для мобильного приложения покупателя")
    Container(api_gateway, "API Gateway", "Kong / Nginx", "Маршрутизация запросов, аутентификация, rate limiting")
    
    Container(monolith, "МаркетХаб (монолит)", "Spring Boot, Java", "Каталог, заказы, платежи, пользователи, уведомления")

    Container(DeliveryService, "Сервис оркестрации доставки", "Spring Boot, Java", "Оркестрация заявок на доставку")
    Container(InventoryService, "Сервис резервирования товара", "Spring Boot, Java", "Резервирование товара")
    Container(NotificationService, "Сервис уведомлений", "Spring Boot, Java", "Рассылка Email, push, SMS уведомлений покупателям и продавцам")
    Container(AnalyticalService, "Сервис аналитики", "Spring Boot, Java", "Построение аналитических отчетов для продавцов")
    
    ContainerDb(postgres_db, "База данных", "PostgreSQL", "Товары, заказы, клиенты и прочие данные")
    ContainerDb(postgres_db_analytical, "База данных для аналитики", "PostgreSQL", "Товары, заказы, клиенты и прочие данные")
    ContainerDb(cache_catalog, "Кэш каталога", "Redis", "Кэширование популярных товаров, категорий и результатов поиска")

    ContainerQueue(message_queue, "Очередь сообщений", "RabbitMQ", "Хранит события заказов для асинхронной обработки")
}

Container_Ext(rate_limiter_cdek, "Rate Limiter CDEK", "Envoy / Nginx", "Ограничение частоты запросов к API CDEK (собственные лимиты)")
Container_Ext(rate_limiter_boxberry, "Rate Limiter Boxberry", "Envoy / Nginx", "Ограничение частоты запросов к API Boxberry (собственные лимиты)")

' Связи пользователей с UI
Rel(buyer, web_app, "Использует", "HTTPS")
Rel(buyer, mobile_app, "Использует", "HTTPS")
Rel(seller, web_app_seller, "Управляет товарами", "HTTPS")

' UI обращается к API Gateway
Rel(web_app, api_gateway, "Отправляет запросы", "HTTPS/JSON")
Rel(web_app_seller, api_gateway, "Отправляет запросы", "HTTPS/JSON")
Rel(mobile_app, api_gateway, "Отправляет запросы", "HTTPS/JSON")

' API Gateway маршрутизирует запросы
Rel(api_gateway, monolith, "Маршрутизирует запросы", "gRPC")
Rel(api_gateway, bff_for_mobile_app, "Маршрутизирует запросы", "gRPC")
Rel(monolith, InventoryService, "Маршрутизирует запросы", "gRPC")

' BFF обращается к монолиту
Rel(bff_for_mobile_app, monolith, "Отправляет запросы", "gRPC")

' Публицация событий
Rel(monolith, message_queue, "Маршрутизирует запросы", "gRPC")
Rel(monolith, message_queue, "Маршрутизирует запросы", "gRPC")
Rel(monolith, message_queue, "Маршрутизирует запросы", "gRPC")

' Публицация событий
Rel(message_queue, AnalyticalService, "Маршрутизирует запросы", "AMQP")
Rel(message_queue, DeliveryService, "Маршрутизирует запросы", "AMQP")
Rel(message_queue, NotificationService, "Маршрутизирует запросы", "AMQP")



' Сервисы работают с БД и кэшем
Rel(monolith, postgres_db, "Читает/пишет данные", "JDBC")
Rel(InventoryService, postgres_db, "Читает/пишет данные", "JDBC")
Rel(AnalyticalService, postgres_db_analytical, "Читает/пишет данные", "JDBC")
Rel(monolith, cache_catalog, "Кэширует каталог", "Redis")

' Внешние системы
Rel(monolith, bank, "Выполняет платежи", "REST API")
Rel(DeliveryService, rate_limiter_cdek, "Отправляет запросы", "HTTPS")
Rel(DeliveryService, rate_limiter_boxberry, "Отправляет запросы", "HTTPS")
Rel(rate_limiter_cdek, delivery_cdek, "Передаёт заказы на доставку", "gRPC")
Rel(rate_limiter_boxberry, delivery_boxberry, "Передаёт заказы на доставку", "gRPC")
Rel(monolith, crm, "Синхронизирует профили клиентов", "REST API")
Rel(NotificationService, email_service, "Отправляет email", "SMTP/REST")
Rel(NotificationService, push_service, "Отправляет push", "FCM/APNs")

@enduml
```

## A3. Rate Limiter: где и зачем
Первый Rate Limiter расположен в API Gateway на входящие запросы к сервисам МаркетХаба. При превышении лимита клиенту возвращается 429 Too Many Requests (Rejection mode). Лимит определяется, как 100 RPS с одного аккаунта (или IP адреса для не аунтифицированных пользователей). Нужен для того, чтобы защититься от намереных/ненамеренных атак и не перегружать систему. 
Rate Limiter в API Gateway отклоняет все запросы, которые превышают установленный лимит (вне зависимости от вида запроса).

Второй/Третий Rate Limiter указаны, как Rate Limiter внешней системы (API служб доставок), предполагается, что у них тоже Rejection mode. Но для покупателя это не видно, потому что при отклонении запроса DeliveryService пробует retry. И в случае, если и все попытки retry неуспешны, то это событие идет в ручной разбор (службой сопровождения и тп). Нужен для того, чтобы не превышать лимиты из требований и ограничений (FR-4, C-2), и не блокировать ключ.
![alt text](sequence-диаграмма.png)

```plantuml
@startuml
title Sequence: Оформление заказа и доставка (CDEK → Boxberry, retry до 5 раз)

actor "Покупатель" as Buyer
participant "Веб-приложение\nпокупателя" as WebApp
participant "API Gateway" as Gateway
participant "МаркетХаб\n(монолит)" as Monolith
queue "Очередь\nсообщений" as Queue
participant "Сервис оркестрации\nдоставки" as Delivery
participant "Rate Limiter\nCDEK" as RLC
participant "API CDEK" as CDEK
participant "Rate Limiter\nBoxberry" as RLB
participant "API Boxberry" as Boxberry

== Оформление заказа ==

Buyer -> WebApp : Оформить заказ
activate WebApp
WebApp -> Gateway : POST /orders (корзина, адрес)
activate Gateway
Gateway -> Gateway : Аутентификация,\nrate limiting
Gateway -> Monolith : POST /orders
activate Monolith

Monolith -> Monolith : Создать заказ\n(статус CREATED)
Monolith -> Monolith : Сохранить заказ\nв БД
Monolith --> Gateway : 201 Created + orderId
deactivate Monolith
Gateway --> WebApp : 201 Created + orderId
deactivate Gateway
WebApp --> Buyer : Заказ оформлен
deactivate WebApp

== Передача заказа в оркестрацию доставки ==

Monolith -> Queue : Публикация события\nOrderCreated
activate Queue
Queue -> Delivery : OrderCreated
deactivate Queue
activate Delivery

== Попытка доставки: CDEK → при неудаче Boxberry ==

group Попытка доставки (первая + до 5 retry)
    note right of Delivery : Интервалы между retry:\n1 мин → 5 мин → 30 мин → 5 ч → 12 ч

    == Шаг 1: CDEK (основная служба) ==

    Delivery -> RLC : POST /delivery/cdek\n(заявка на доставку)
    activate RLC
    RLC -> RLC : Проверить счётчик

    alt CDEK доступен и лимит не превышен
        RLC -> CDEK : Передать заявку
        activate CDEK
        CDEK --> RLC : 200 OK + трек-номер
        deactivate CDEK
        RLC --> Delivery : 200 OK + трек-номер
        deactivate RLC
        note right of Delivery : Доставка оформлена\nчерез CDEK
    else Лимит превышен или CDEK недоступен
        RLC --> Delivery : 429 Too Many Requests\n/ 503 Service Unavailable
        deactivate RLC
        note right of Delivery : CDEK недоступен —\nпробуем Boxberry

        == Шаг 2: Boxberry (резервная служба) ==

        Delivery -> RLB : POST /delivery/boxberry\n(заявка на доставку)
        activate RLB
        RLB -> RLB : Проверить счётчик

        alt Boxberry доступен и лимит не превышен
            RLB -> Boxberry : Передать заявку
            activate Boxberry
            Boxberry --> RLB : 200 OK + трек-номер
            deactivate Boxberry
            RLB --> Delivery : 200 OK + трек-номер
            deactivate RLB
            note right of Delivery : Доставка оформлена\nчерез Boxberry
        else Boxberry недоступен или лимит превышен
            RLB --> Delivery : 429 / 503
            deactivate RLB
            note right of Delivery : Обе службы недоступны.\nЖдём интервал и\nповторяем попытку\n(CDEK → Boxberry)
        end
    end
end

== Все попытки исчерпаны ==

note right of Delivery : 5 retry исчерпаны,\nобе службы недоступны.\nЗаказ помечается\nдля ручного разбора.
Delivery -> Delivery : Пометить заказ\nкак требующий\nручного разбора

Delivery -> Queue : Публикация события\nDeliveryCreated
activate Queue
deactivate Queue

== Синхронный опрос статуса доставки ==

loop Периодический опрос статуса\n(пока статус не «доставлено»)
    Delivery -> RLC : GET /delivery/cdek/{trackId}\n(статус перевозки)
    activate RLC
    RLC -> RLC : Проверить счётчик

    alt Лимит не превышен и CDEK доступен
        RLC -> CDEK : GET /status/{trackId}
        activate CDEK
        CDEK --> RLC : 200 OK + статус\n(принят / в пути / доставлен)
        deactivate CDEK
        RLC --> Delivery : 200 OK + статус
        deactivate RLC
        note right of Delivery : Статус получен\nсинхронно
    else Лимит превышен или CDEK недоступен
        RLC --> Delivery : 429 / 503
        deactivate RLC
        note right of Delivery : Повторим опрос\nпозже
    end

    opt Если доставка оформлена через Boxberry
        Delivery -> RLB : GET /delivery/boxberry/{trackId}\n(статус перевозки)
        activate RLB
        RLB -> Boxberry : GET /status/{trackId}
        activate Boxberry
        Boxberry --> RLB : 200 OK + статус
        deactivate Boxberry
        RLB --> Delivery : 200 OK + статус
        deactivate RLB
    end

    Delivery -> Queue : Публикация события\nDeliveryStatusChanged
    activate Queue
    deactivate Queue
end

deactivate Delivery

== Монолит как потребитель событий доставки ==

activate Queue
Queue -> Monolith : DeliveryStatusChanged
deactivate Queue
activate Monolith
Monolith -> Monolith : Обновить статус\nдоставки в заказе
Monolith -> Monolith : Сохранить\nв БД
deactivate Monolith

@enduml
```

## B4. Диагностика потенциальных антипаттернов
### Кейс А. BFF для мобильного приложения покупателя содержит логику расчёта стоимости доставки: проверяет вес товаров, применяет скидки программы лояльности, выбирает оптимальную службу доставки.
Это антипаттерн Smart UI. 
В случае маркетхаба: за стомость доставки отвечают внешние сервисы (CDEK, Boxberry), UI должен напрямую ходить к ним, что вызовет проблемы с лимитами, retry. 
Логика расчета стоимости доставки, расчет скидок и выбор оптимальной службы доставки должны быть вынесены в разные сервисы.
### Кейс Б. OrderStatusService напрямую читает таблицу payments из базы данных PaymentService, чтобы показать покупателю статус оплаты рядом со статусом заказа.
Нарушение инкапсуляции данных, размывание границ.
В контексте моей версии маркетхаба - хуже не станет, у меня и так это монолит с единой БД))
В случае разных БД и сервисов: каждый раз при добавлении новых статусов, их изменении и тп придется дорабатывать сервис, который казалось бы никак не был связан с оплатой (OrderStatusService).
По-хорошему это должны быть разные сервисы, и чтение одним сервисом БД другого сервиса не допускается. Для получения статуса оплаты необходимо обращаться к PaymentService путём интеграции через API. 
### Кейс В. OrderOrchestratorService в одном методе: создаёт заказ, резервирует товар на складе, запрашивает тариф доставки, списывает оплату, запускает начисление баллов лояльности, отправляет email-уведомление и пишет событие в аналитику.
God Service
Порождает много зависимостей, высокую связанность.
Скорее всего, это будет SPOF. Так же будет сложность с масштабированием (не получится масштабировать отдельно резервирование товаров на складе или списание оплаты). Изменение в одном бизнес-домене вполне смогут ломать логику другого домена. 
Это должны быть разные сервисы.

## C5. Декомпозиция на модули/сервисы

Мне кажется я уже разделила Order & Fulfillment на составляющие т.к. нарисовала модуль как микросервисы.

### DeliveryService (внутренний)
    Взаимодейтвует со "старыми" сервисами через очередь сообщений. Не знает о них ничего, и не должен.
    Зона ответственности: оркестрация процесса создания доставки во внешних сервисах.

### InventoryService
    Взаимодействует с монолитом через gRPC (синхронное взаимодействие).
    Зона ответственности: резервирование товаров 
    Знает какой товар и количество и по-какому заказу необходимо зарезервировать или снять резерв. 
    Не должен знать ни о статусе оплаты/доставки, ничего их клиентской информации или атрибутах товара. 

###  NotificationService
    Взаимодейтвует со "старыми" сервисами через очередь сообщений. Не знает о них ничего, и не должен.
    Зона ответственности: отправка уведомлений

###  СДЭК API (внешняя)/Boxberry API (внешняя)
    Синхронное взаимодействие с DeliveryService
    Зона ответственности: прием заявок на доставку.
    Должен знать идентификатор/любой другой уникальный атрибут заказа. Атрибуты доставляемых товаров (категория/вес/габариты/особые условия). Атрибуты клиента (куда доставить/кому).

### Аналитическое хранилище
    Взаимодействует с основным сервисом через очередь сообщений. 
    Зона ответственности: сбор и хранение данных по заказам и построение аналитических отчетов для продавцов.

## C6. Контракты между слоями
![alt text](schema_C6.png)

```plantuml
@startuml
' ============================================================
' ДИРЕКТИВЫ ОФОРМЛЕНИЯ
' ============================================================
' autonumber — автоматическая нумерация сообщений (1, 2, 3...)
' Удобно, когда в тексте ответа ссылаешься на «шаг 5».
autonumber

' sequenceMessageAlign center — выравнивание текста на стрелках по центру.
' По умолчанию текст прижат к одному краю, при длинных подписях это некрасиво.
skinparam sequenceMessageAlign center

' responseMessageBelowArrow — ответные стрелки (--) рисуются под стрелкой запроса,
' а не поверх неё. Улучшает читаемость при асинхронных ответах.
skinparam responseMessageBelowArrow true

' ============================================================
' УЧАСТНИКИ
' ============================================================
' actor — человечек (внешний пользователь).
' participant — прямоугольник (система/контейнер).
' queue — прямоугольник со «стеком» (очередь сообщений).
'
' Синтаксис: <тип> "Отображаемое имя" as Псевдоним
'   • Отображаемое имя — то, что видно на диаграмме.
'     \n внутри строки — перенос строки.
'   • Псевдоним — то, что используется в стрелках (Buyer, Mobile, ...).
'     Без псевдонима PlantUML сам сгенерирует имя, и стрелки станут нечитаемыми.
'
' Порядок объявления = порядок колонок слева направо.
' Поэтому сначала пользователь, потом UI, потом инфраструктура.

actor "Покупатель" as Buyer
participant "Мобильное приложение\nпокупателя" as Mobile
participant "BFF мобильного\nприложения" as BFF
participant "API Gateway" as Gateway
participant "МаркетХаб\n(монолит)" as Marketplace
participant "Сервис резервирования\nтовара" as Inventory
queue "Очередь сообщений\n(RabbitMQ)" as Queue
participant "Сервис оркестрации\nдоставки" as Delivery
participant "Сервис уведомлений" as Notifications
participant "Сервис аналитики" as Analytics
participant "Rate Limiter CDEK" as RateLimiterCdek
participant "Rate Limiter Boxberry" as RateLimiterBoxberry
participant "Платёжный шлюз" as Bank
participant "Email-сервис" as Email
participant "Push-сервис" as Push

' ============================================================
' РАЗДЕЛИТЕЛИ ЭТАПОВ
' ============================================================
' == Текст == — горизонтальный разделитель с заголовком.
' Разбивает поток на смысловые блоки: оформление, оплата, доставка.

== 1. Оформление заказа ==

' ============================================================
' СИНХРОННЫЕ СООБЩЕНИЯ
' ============================================================
' Синтаксис: A -> B : текст
'   • -> — синхронный вызов (сплошная стрелка с закрашенным наконечником).
'   • текст — подпись под стрелкой. \n — перенос строки.
'   • Можно добавить протокол: "HTTPS/JSON", "gRPC" — как часть подписи.

Buyer -> Mobile : нажатие «Оформить заказ»
Mobile -> Gateway : POST /api/v1/orders\nHTTPS/JSON
Gateway -> BFF : маршрутизация\ngRPC
BFF -> Marketplace : createOrder(cart, userId, address)\ngRPC

== 2. Проверка наличия и резервирование ==

' ============================================================
' ОТВЕТНЫЕ СООБЩЕНИЯ
' ============================================================
' Синтаксис: A --> B : текст
'   • --> — ответ (пунктирная стрелка).

Marketplace -> Inventory : checkAvailability(sku, qty)\ngRPC
Inventory --> Marketplace : available = true/false\ngRPC

' ============================================================
' АЛЬТЕРНАТИВНЫЕ ПОТОКИ (alt / else / end)
' ============================================================
' alt Условие ... else Условие ... end — блок «если/иначе».
'   • alt — начало блока, текст после alt — условие.
'   • else — альтернативная ветка.
'   • end — закрытие блока.
' Можно вкладывать друг в друга.

alt Товар закончился
    Marketplace --> BFF : 409 OUT_OF_STOCK
    BFF --> Mobile : 409
    Mobile --> Buyer : «Товар закончился»

    ' ========================================================
    ' ЗАМЕТКИ (note)
    ' ========================================================
    ' note right of X : текст — заметка справа от участника X.
    ' Другие варианты: note left of X, note over X, note over X, Y.

    note right of Buyer : Альтернативный путь A

    ' ========================================================
    ' РАННИЙ ВЫХОД (return)
    ' ========================================================
    ' return — прерывает текущий блок и возвращает управление наружу.
    ' Здесь: если товара нет — дальше поток не идёт, выходим из сценария.

    return
end

Marketplace -> Inventory : reserveStock(orderId, items, ttl)\ngRPC
Inventory --> Marketplace : reservationId, expiresAt\ngRPC

== 3. Инициация оплаты ==

' ============================================================
' ЗАМЕТКА НАД УЧАСТНИКОМ
' ============================================================
' note over X — заметка поверх участника X (широкая, «плавающая»).
' Используется, когда нужно пояснить, что происходит внутри контейнера,
' не рисуя лишних стрелок.

note over Marketplace
  Платёжный модуль — внутри монолита.
  Отдельного PaymentService в C4 нет.
end note

Marketplace -> Bank : REST API /payments
Bank --> Marketplace : paymentId, status = PENDING

alt Оплата отклонена
    ' Направление стрелки Bank -> Marketplace,
    ' потому что это асинхронный callback от платёжного шлюза.
    Bank -> Marketplace : callback FAILED
    Marketplace -> Inventory : releaseReservation(reservationId)\ngRPC
    Marketplace --> BFF : 402 PAYMENT_DECLINED
    BFF --> Mobile : 402
    Mobile --> Buyer : «Оплата отклонена»
    note right of Buyer : Альтернативный путь B
    return
end

Bank -> Marketplace : callback SUCCESS

== 4. Фиксация заказа и начисление баллов ==

note over Marketplace
  Внутри монолита:
  — сохранение заказа со статусом PAID;
  — начисление баллов лояльности
    (модуль Loyalty внутри монолита).
  На межконтейнерном уровне это одна
  атомарная операция над состоянием заказа.
end note

== 5. Публикация события «заказ оплачен» ==

Marketplace -> Queue : publish OrderPaidEvent\nAMQP

== 6. Асинхронная обработка ==

' ============================================================
' ПАРАЛЛЕЛЬНЫЕ ПОТОКИ (par / else / end)
' ============================================================
' par Действие ... else Действие ... end — блок параллельных веток.
'   • par — начало, текст после par — первая ветка.
'   • else — следующая параллельная ветка.
'   • end — закрытие.

par Доставка
    Queue -> Delivery : OrderPaidEvent\nAMQP
    Delivery -> RateLimiterCdek : HTTPS
    RateLimiterCdek -> Delivery : пропуск
    Delivery -> RateLimiterBoxberry : HTTPS
    Delivery -> Queue : publish DeliveryCreatedEvent\nAMQP
else Уведомления
    Queue -> Notifications : OrderPaidEvent\nAMQP
    Notifications -> Email : SMTP/REST
    Notifications -> Push : FCM/APNs
else Аналитика
    Queue -> Analytics : OrderPaidEvent\nAMQP
end

== 7. Ответ покупателю ==

Marketplace --> BFF : 201 Created\n{orderId, status, eta, points}\ngRPC
BFF --> Gateway : 201
Gateway --> Mobile : 201
Mobile --> Buyer : Экран «Заказ оформлен»\n+ номер, ETA, баллы

@enduml
```

### Контракты переходов

| # | Переход | Тип | Успех | Ошибка |
|---|---------|-----|-------|--------|
| 1 | Покупатель → Мобильное приложение | UI-событие | Нажатие обработано | — |
| 2 | Мобильное приложение → API Gateway | Синхронный (HTTPS/JSON) | 2xx | 4xx/5xx, таймаут |
| 3 | API Gateway → BFF | Синхронный (gRPC) | OK | DEADLINE_EXCEEDED, UNAVAILABLE |
| 4 | BFF → Монолит (createOrder) | Синхронный (gRPC) | OrderResponse | INVALID_ARGUMENT, UNAVAILABLE |
| 5 | Монолит → Сервис резервирования (checkAvailability) | Синхронный (gRPC) | available=true/false | NOT_FOUND, UNAVAILABLE |
| 6 | Монолит → Сервис резервирования (reserveStock) | Синхронный (gRPC), идемпотентный | reservationId, expiresAt | INSUFFICIENT_STOCK, DEADLINE_EXCEEDED |
| 7 | Монолит → Платёжный шлюз | Синхронный (REST) + async callback | paymentId, PENDING | PAYMENT_GATEWAY_UNAVAILABLE |
| 8 | Платёжный шлюз → Монолит (callback) | Асинхронный (webhook) | SUCCESS | FAILED, TIMEOUT → компенсация |
| 9 | Монолит → Сервис резервирования (releaseReservation) | Синхронный (gRPC), идемпотентный | released | NOT_FOUND (уже истекла — OK) |
| 10 | **Внутри монолита: фиксация заказа + начисление баллов** | Локальная транзакция | Оба действия выполнены | Откат транзакции → 500 |
| 11 | Монолит → Очередь сообщений (publish OrderPaidEvent) | Асинхронный (AMQP), at-least-once | publisher confirm | nack → outbox + retry |
| 12 | Очередь → Сервис доставки | Асинхронный (AMQP), идемпотентный consumer | Заявка создана | Ошибка → DLQ + retry |
| 13 | Сервис доставки → Rate Limiter → CDEK/Boxberry | Синхронный (HTTPS/gRPC) | 2xx + trackingId | 429 → backoff; 5xx → retry; 4xx → DLQ |
| 14 | Очередь → Сервис уведомлений | Асинхронный (AMQP) | Email + push отправлены | Ошибка → DLQ + retry |
| 15 | Очередь → Сервис аналитики | Асинхронный (AMQP) | Событие обработано | Ошибка → DLQ + retry |
| 16 | Монолит → BFF → Gateway → Мобильное приложение → Покупатель | Синхронный ответ | 201 + orderId, ETA, points | 409/402/500 |


---

### Точки Layer Bleed 

| # | Антипаттерн | 
|---|-------------|
| **LB-1** | `BFF → Inventory` напрямую; BFF не резервирует товар — это делает монолит |
| **LB-2** | `Marketplace → delivery_cdek` напрямую; Доставка идёт через `DeliveryService` + `rate_limiter_cdek` |
| **LB-3** | `Marketplace → email_service` напрямую; Уведомления — только через `OrderPaidEvent → Queue → Notifications` |
| **LB-4** | `Delivery → Marketplace` (обратный синхронный вызов); DeliveryService не дёргает монолит — всё через события |
| **LB-5** | `Gateway` содержит логику заказа; Gateway — только маршрутизация/auth/rate limiting |
| **LB-8** | `Mobile → Marketplace` минуя Gateway/BFF; Нарушение маршрутизации и аутентификации |

## C7. Компромиссы и обоснование

# Архитектурные компромиссы модуля Order & Fulfillment

С учётом C4-диаграммы МаркетХаба: заказы, платежи и лояльность — **внутри монолита** (`monolith`), а `InventoryService`, `DeliveryService`, `NotificationService`, `AnalyticalService` — **отдельные контейнеры**, взаимодействующие через gRPC и RabbitMQ. Компромиссы ниже привязаны именно к этой архитектуре.

| Компромисс | Выбрали | Отказались от | Причина | Остаточный риск |
|---|---|---|---|---|
| **Резервирование товара: sync vs async** | **Синхронный gRPC-вызов** `monolith → InventoryService.reserveStock` с TTL и компенсацией `releaseReservation` | Полностью асинхронное резервирование через событие (order created → inventory reserved) | Покупатель должен сразу узнать, что товара нет | Если `InventoryService` недоступен — заказ не оформить |
| **Границы монолита: что оставить внутри, что вынести** | **Заказы + платежи + лояльность — внутри монолита**; вынесены только Inventory, Delivery, Notification, Analytics | Вынос OrderService и PaymentService в отдельные контейнеры (микросервисный «идеал») | Исходила из того, что задача стоит именно добавить новую функциональность, а не перестраивать всю систему | Монолит остаётся слабым местом с точки зрения отказоустойчивости и масштабирования системы |
| **Коммуникация «монолит → внешние потребители»: события vs синхронные вызовы** | **Асинхронная публикация `OrderPaidEvent` в RabbitMQ**; Delivery, Notifications, Analytics — независимые подписчики | Синхронные вызовы `monolith → DeliveryService`, `monolith → NotificationService`, `monolith → AnalyticalService` | Уведомления, доставка и аналитика не должны блокировать ответ покупателю и не должны влиять на доступность оформления заказа. Событие — единственный контракт, не требующий знания о потребителях | At-least-once доставка → возможны дубликаты событи |

---
