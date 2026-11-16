# Специфікація вимог до програмного забезпечення (SRS)

**Система:** EventMarket — масштабована подієво-орієнтована екосистема для мультинаціонального маркетплейсу
**Стандарт структури:** ISO/IEC/IEEE 29148:2018 (Software Requirements Specification)
**Версія:** 1.0 (після ШІ-аудиту) · **Дата:** 16.11.2026 · **Статус:** Approved
**Автор:** Андрющенко Л. А. · **Джерело вимог:** [Розділ 1, підрозділ 1.5](1_analysis.md) (FR-1…FR-8, NFR-P1…NFR-L1)

Навігація: [Розділ 2. Проєктування](2_design.md) · [ADR-001](adr/0001-choice-of-technology.md) · [README](README.md)

---

## 1. Вступ

### 1.1 Призначення

Документ є технічним контрактом між вимогами кваліфікаційної роботи та архітектурою системи EventMarket. Документ фіксує межі системи, атомарні функціональні вимоги, вимірювані нефункціональні вимоги та методи їх верифікації. Кожна вимога має унікальний ідентифікатор, на який посилаються діаграми C4 у [Розділі 2](2_design.md) та архітектурні рішення в каталозі [adr/](adr/).

### 1.2 Модальні дієслова

- **повинна / повинен** — обов'язкова вимога (пріоритет Must), невиконання означає невідповідність прототипу SRS;
- **має** — вимога з пріоритетом Should, виконання якої перевіряється, але допускає обґрунтоване відхилення в межах прототипу.

### 1.3 Обсяг (Scope)

**У межах системи:** прийняття замовлення, резервування залишків, авторизація та повернення платежів (через емульований платіжний провайдер), розподілена транзакція замовлення за патерном Choreography-based Saga, регіональні ціни та валюти, локалізовані сповіщення, автентифікація, Rate Limiting, спостережуваність.

**Поза межами системи:** реальна інтеграція з платіжною системою та повний обсяг PCI DSS, доставка й логістика, рекомендаційні системи та AI/ML, мобільні застосунки, активна мультирегіональна реплікація кластерів (active-active). Мультирегіональність реалізується на логічному рівні (регіон замовлення, валюта, податкова ставка, локаль) в одному кластері Kubernetes.

---

## 2. Контекст системи (System Context)

### 2.1 Межі системи

EventMarket приймає замовлення покупців через єдину точку входу (API Gateway), узгоджує стан замовлення, залишків і платежів через асинхронний обмін подіями в Apache Kafka та повідомляє покупця про результат. Синхронний шлях запиту закінчується прийняттям замовлення (відповідь `202 Accepted`). Резервування, авторизація платежу та сповіщення виконуються асинхронно.

### 2.2 Актори та зовнішні системи

| ID | Актор / зовнішня система | Тип | Взаємодія з EventMarket | Протокол |
|---|---|---|---|---|
| ACT-01 | Покупець (Customer) | Person | Перегляд каталогу, створення замовлення, перевірка статусу, отримання сповіщень | REST/HTTPS, JSON |
| ACT-02 | Менеджер каталогу (Catalog Manager) | Person | Ведення товарів і регіональних цін | REST/HTTPS, JSON |
| ACT-03 | Оператор складу (Warehouse Operator) | Person | Поповнення та коригування залишків | REST/HTTPS, JSON |
| ACT-04 | Інженер експлуатації (SRE) | Person | Моніторинг метрик, трас і Dead Letter Topic | HTTPS (Grafana, Jaeger) |
| EXT-01 | Платіжний провайдер (емуляція) | System_Ext | Авторизація та повернення платежів | REST/HTTPS, JSON |
| EXT-02 | Джерело курсів валют (API НБУ) | System_Ext | Отримання офіційних курсів UAH/EUR/USD | REST/HTTPS, JSON |
| EXT-03 | Поштовий сервіс (SMTP-провайдер; Mailpit у тестовому середовищі) | System_Ext | Доставка email-сповіщень | SMTP + STARTTLS |

### 2.3 Обмеження (Constraints)

| ID | Обмеження |
|---|---|
| CON-01 | Мова реалізації сервісів — Go, підтримувана гілка релізів (не нижче 1.26 станом на листопад 2026 року). |
| CON-02 | Брокер подій — Apache Kafka (режим KRaft); обґрунтування: [ADR-001](adr/0001-choice-of-technology.md). |
| CON-03 | Сховища: PostgreSQL за принципом «база даних на сервіс»; Redis для атомарного резервування та Rate Limiting. |
| CON-04 | Розгортання: Docker, Kubernetes (Helm), IaC на OpenTofu, CI/CD на GitHub Actions. |
| CON-05 | Ресурсний бюджет тестового кластера: не більше 8 vCPU та 16 GiB RAM (див. [SRS-NFR-INFRA-02](#srs-nfr-infra-02)). |
| CON-06 | Платіжні реквізити не зберігаються в сховищах системи; зберігаються лише токени провайдера (PCI DSS scope reduction). |

### 2.4 Припущення (Assumptions)

- Ідентичність користувачів веде Identity Provider (Keycloak) за протоколом OpenID Connect.
- Курси валют оновлюються не частіше одного разу на добу; затримка курсу до 24 год прийнятна для прототипу.
- Пікове навантаження моделюється інструментом k6; реальний трафік не використовується.

---

## 3. Функціональні вимоги (Functional Requirements)

Колонка «Джерело» трасує вимогу до Core Features Розділу 1; позначка «Аудит» означає вимогу, додану за результатами ШІ-аудиту (див. AI Insights).

### 3.1 API Gateway

| ID | Вимога | Пріоритет | Джерело |
|---|---|---|---|
| <a id="srs-fun-01"></a>SRS-FUN-01 | API Gateway повинен перевіряти підпис (RS256), видавця (`iss`), аудиторію (`aud`) і термін дії (`exp`) токена JWT для кожного запиту до захищених ресурсів та відхиляти недійсний токен з кодом `401`. | Must | FR-1 |
| <a id="srs-fun-02"></a>SRS-FUN-02 | API Gateway повинен маршрутизувати запити з префіксом `/v1/` до цільового сервісу за таблицею маршрутів. | Must | FR-1 |
| <a id="srs-fun-03"></a>SRS-FUN-03 | API Gateway повинен обмежувати частоту запитів за алгоритмом Token Bucket окремо для кожного суб'єкта токена (`sub`) з параметрами за замовчуванням 100 запитів/хв і ємністю bucket 20 та повертати код `429` із заголовком `Retry-After` після вичерпання ліміту. | Must | FR-1 |

### 3.2 Catalog Service

| ID | Вимога | Пріоритет | Джерело |
|---|---|---|---|
| <a id="srs-fun-04"></a>SRS-FUN-04 | Catalog Service повинен повертати картку товару з ціною в валюті регіону запиту (UAH, EUR, USD; ISO 4217) та описом мовою локалі запиту (uk, en). | Must | FR-2 |
| <a id="srs-fun-05"></a>SRS-FUN-05 | Catalog Service повинен оновлювати курси валют із зовнішнього джерела (EXT-02) не рідше одного разу на 24 год та зберігати дату курсу разом із перерахованою ціною. | Must | FR-2, Аудит |
| <a id="srs-fun-06"></a>SRS-FUN-06 | Catalog Service повинен надавати внутрішнім сервісам метод читання актуальної ціни позиції за `sku` та регіоном через gRPC. | Must | FR-2 |
| <a id="srs-fun-07"></a>SRS-FUN-07 | Catalog Service повинен дозволяти користувачу з роллю `catalog_manager` створювати й змінювати товари та регіональні ціни. | Must | Аудит (актор ACT-02) |

### 3.3 Order Service

| ID | Вимога | Пріоритет | Джерело |
|---|---|---|---|
| <a id="srs-fun-08"></a>SRS-FUN-08 | Order Service повинен створювати замовлення зі статусом `PENDING` та повертати код `202 Accepted` з ідентифікатором замовлення (`order_id`). | Must | FR-3 |
| <a id="srs-fun-09"></a>SRS-FUN-09 | Order Service повинен повертати відповідь першої обробки без створення нового замовлення на повторний запит із тим самим заголовком `Idempotency-Key` протягом 24 год. | Must | Аудит |
| <a id="srs-fun-10"></a>SRS-FUN-10 | Order Service повинен повертати статус замовлення за запитом `GET /v1/orders/{id}` лише власнику замовлення та відповідати кодом `404` іншим користувачам. | Must | Аудит |
| <a id="srs-fun-11"></a>SRS-FUN-11 | Order Service повинен допускати лише переходи статусу `PENDING → CONFIRMED` та `PENDING → CANCELLED` і відхиляти інші переходи. | Must | FR-3 |

### 3.4 Transactional Outbox

| ID | Вимога | Пріоритет | Джерело |
|---|---|---|---|
| <a id="srs-fun-12"></a>SRS-FUN-12 | Кожен сервіс-видавець подій повинен записувати зміну бізнес-даних і відповідну подію в таблицю `outbox` в одній транзакції PostgreSQL. | Must | FR-4 |
| <a id="srs-fun-13"></a>SRS-FUN-13 | Relay-процес повинен публікувати події з таблиці `outbox` у Kafka з ключем партиціонування `order_id` та позначати подію опублікованою лише після підтвердження брокером (`acks=all`). | Must | FR-4 |

### 3.5 Inventory Service

| ID | Вимога | Пріоритет | Джерело |
|---|---|---|---|
| <a id="srs-fun-14"></a>SRS-FUN-14 | Inventory Service повинен перевіряти та зменшувати доступний залишок позиції одним атомарним Lua-скриптом Redis. | Must | FR-5 |
| <a id="srs-fun-15"></a>SRS-FUN-15 | Inventory Service повинен звільняти резерв за подією компенсації або після спливання TTL резерву (15 хв без події `PaymentAuthorized`). | Must | FR-5, Аудит |
| <a id="srs-fun-16"></a>SRS-FUN-16 | Inventory Service повинен асинхронно записувати стан резервувань у PostgreSQL як джерело відновлення Redis. | Must | FR-5 |
| <a id="srs-fun-17"></a>SRS-FUN-17 | Inventory Service повинен дозволяти користувачу з роллю `warehouse_operator` поповнювати й коригувати залишки. | Must | Аудит (актор ACT-03) |

### 3.6 Payment Service

| ID | Вимога | Пріоритет | Джерело |
|---|---|---|---|
| <a id="srs-fun-18"></a>SRS-FUN-18 | Payment Service повинен авторизувати платіж через платіжного провайдера (EXT-01) з ключем ідемпотентності, що дорівнює `order_id`. | Must | FR-6 |
| <a id="srs-fun-19"></a>SRS-FUN-19 | Payment Service повинен виконувати повернення авторизованого платежу за подією компенсації. | Must | FR-6 |
| <a id="srs-fun-20"></a>SRS-FUN-20 | Payment Service повинен блокувати виклики платіжного провайдера через Circuit Breaker за параметрами [SRS-NFR-REL-04](#srs-nfr-rel-04). | Must | FR-6 |

### 3.7 Choreography-based Saga та надійність обробки подій

| ID | Вимога | Пріоритет | Джерело |
|---|---|---|---|
| <a id="srs-fun-21"></a>SRS-FUN-21 | Order Service повинен встановлювати статус `CONFIRMED` та публікувати подію `OrderConfirmed` після отримання події `PaymentAuthorized`. | Must | FR-7 |
| <a id="srs-fun-22"></a>SRS-FUN-22 | Система повинна виконувати компенсуючі дії (`InventoryReleased`, `PaymentRefunded`, `OrderCancelled`) після отримання події `InventoryRejected` або `PaymentFailed`. | Must | FR-7 |
| <a id="srs-fun-23"></a>SRS-FUN-23 | Кожен консюмер повинен пропускати повідомлення з ідентифікатором `event_id`, який уже зафіксовано в таблиці оброблених подій (`processed_events`). | Must | FR-7 |
| <a id="srs-fun-24"></a>SRS-FUN-24 | Кожен консюмер повинен виконувати повторні спроби обробки за параметрами [SRS-NFR-REL-03](#srs-nfr-rel-03) та передавати повідомлення в Dead Letter Topic після вичерпання спроб. | Must | FR-7 |

### 3.8 Notification Service

| ID | Вимога | Пріоритет | Джерело |
|---|---|---|---|
| <a id="srs-fun-25"></a>SRS-FUN-25 | Notification Service повинен надсилати покупцю email-сповіщення про зміну статусу замовлення мовою локалі покупця (uk, en). | Must | FR-8 |

---

## 4. Нефункціональні вимоги (NFRs Matrix)

Значення є критеріями приймання прототипу. Колонка «Джерело» посилається на матрицю NFR Розділу 1 (табл. 1.7).

| ID | Категорія | Вимога (вимірюваний індикатор) | Джерело |
|---|---|---|---|
| <a id="srs-nfr-perf-01"></a>SRS-NFR-PERF-01 | Продуктивність | Система повинна обробляти 500 RPS на `POST /v1/orders` протягом 10 хв із часткою відповідей 5xx не більше 0,1 %. | NFR-P1 |
| <a id="srs-nfr-perf-02"></a>SRS-NFR-PERF-02 | Продуктивність | Latency прийняття замовлення повинна становити p95 ≤ 200 мс та p99 ≤ 500 мс при 500 RPS. | NFR-P2 |
| <a id="srs-nfr-perf-03"></a>SRS-NFR-PERF-03 | Продуктивність | Latency читання каталогу повинна становити p95 ≤ 100 мс при 500 RPS. | NFR-P3 |
| <a id="srs-nfr-perf-04"></a>SRS-NFR-PERF-04 | Продуктивність | Час від події `OrderCreated` до кінцевого статусу замовлення повинен становити p95 ≤ 3 с при 500 RPS. | NFR-P4 |
| <a id="srs-nfr-perf-05"></a>SRS-NFR-PERF-05 | Продуктивність | Затримка між записом події в `outbox` і її публікацією в Kafka повинна становити p95 ≤ 500 мс. | NFR-P5 |
| <a id="srs-nfr-scal-01"></a>SRS-NFR-SCAL-01 | Масштабованість | Система повинна витримувати 1000 RPS протягом 5 хв; consumer lag повинен повернутися до значення < 1 000 повідомлень не пізніше ніж через 60 с після завершення піку. | NFR-S1 |
| <a id="srs-nfr-scal-02"></a>SRS-NFR-SCAL-02 | Масштабованість | Подвоєння кількості реплік консюмера повинно збільшувати пропускну здатність не менше ніж у 1,6 раза; топіки Saga повинні мати не менше 12 партицій. | NFR-S2 |
| <a id="srs-nfr-cons-01"></a>SRS-NFR-CONS-01 | Консистентність | 10 000 конкурентних запитів на резервування 1 000 одиниць залишку повинні давати не більше 1 000 успішних резервувань (перепродаж = 0). | NFR-C1 |
| <a id="srs-nfr-cons-02"></a>SRS-NFR-CONS-02 | Консистентність | Кількість дублікатів платежів повинна дорівнювати 0 за ін'єкції не менше 5 % повторних повідомлень. | NFR-C2 |
| <a id="srs-nfr-cons-03"></a>SRS-NFR-CONS-03 | Консистентність | Кількість втрачених подій повинна дорівнювати 0 за аварійної зупинки Order Service між фіксацією транзакції та публікацією події. | NFR-C3 |
| <a id="srs-nfr-cons-04"></a>SRS-NFR-CONS-04 | Консистентність | Метрика розбіжності `inventory_drift_items` (кількість позицій зі значенням залишку в Redis, відмінним від PostgreSQL) повинна повертатися до 0 не пізніше ніж через 60 с після відновлення Redis. | NFR-C4 |
| <a id="srs-nfr-sec-01"></a>SRS-NFR-SEC-01 | Безпека | Автентифікація повинна виконуватися за OAuth 2.0 / OpenID Connect (Authorization Code + PKCE); токени JWT повинні мати підпис RS256 і TTL access-токена не більше 15 хв. | NFR-SEC1 |
| <a id="srs-nfr-sec-02"></a>SRS-NFR-SEC-02 | Безпека | Зовнішній трафік повинен використовувати TLS не нижче 1.2; секрети повинні зберігатися поза репозиторієм (Kubernetes Secrets); Gitleaks повинен виявляти 0 секретів. | NFR-SEC2 |
| <a id="srs-nfr-sec-03"></a>SRS-NFR-SEC-03 | Безпека | Система повинна проходити автоматизовані перевірки OWASP API Security Top 10 (2023): API1 BOLA (доступ до чужого замовлення → `404`), API4 (Rate Limiting → `429`), валідація вхідних даних (некоректне тіло → `400`); сканування OWASP ZAP (baseline) повинно давати 0 знахідок рівня High. | NFR-SEC3 |
| <a id="srs-nfr-sec-04"></a>SRS-NFR-SEC-04 | Безпека | Trivy повинен виявляти 0 вразливостей рівня CRITICAL та 0 рівня HIGH із доступним виправленням в образах контейнерів і IaC. | NFR-SEC4 |
| <a id="srs-nfr-sec-05"></a>SRS-NFR-SEC-05 | Безпека | Журнали та події повинні містити 0 полів PII у відкритому вигляді (email, телефон, адреса маскуються); система повинна зберігати лише токени платіжного провайдера. | NFR-SEC5 |
| <a id="srs-nfr-sec-06"></a>SRS-NFR-SEC-06 | Безпека | Система має анонімізувати персональні дані покупця в сервісах Order і Notification не пізніше ніж через 30 днів після запиту на видалення (GDPR, ст. 17). | Аудит |
| <a id="srs-nfr-rel-01"></a>SRS-NFR-REL-01 | Надійність | За примусової зупинки одного pod будь-якого stateless-сервісу частка помилок `POST /v1/orders` повинна бути не більше 1 % протягом 60 с. Цільовий SLO доступності 99,9 % (28 днів) застосовується для експлуатації та не верифікується в межах прототипу. | NFR-R1 |
| <a id="srs-nfr-rel-02"></a>SRS-NFR-REL-02 | Надійність | Кластер Kafka повинен складатися з 3 брокерів (KRaft) з `replication.factor=3`, `min.insync.replicas=2`; продюсери повинні використовувати `acks=all` та `enable.idempotence=true`. | NFR-R2 |
| <a id="srs-nfr-rel-03"></a>SRS-NFR-REL-03 | Надійність | Повторні спроби обробки повинні виконуватися до 5 разів із затримкою 100 мс × 2ⁿ та jitter ±20 %, після чого повідомлення повинно передаватися в Dead Letter Topic `<topic>.dlt`. | NFR-R3 |
| <a id="srs-nfr-rel-04"></a>SRS-NFR-REL-04 | Надійність | Circuit Breaker повинен переходити у стан Open за частки помилок ≥ 50 % у ковзному вікні 20 запитів і в стан Half-Open через 30 с. | NFR-R4 |
| <a id="srs-nfr-rel-05"></a>SRS-NFR-REL-05 | Надійність | Для PostgreSQL повинні забезпечуватися RPO ≤ 5 хв (архівування WAL) та RTO ≤ 30 хв. | NFR-R5 |
| <a id="srs-nfr-rel-06"></a>SRS-NFR-REL-06 | Надійність | Кожен сервіс повинен надавати liveness- і readiness-проби та завершувати обробку поточних повідомлень (graceful shutdown) не довше ніж за 30 с після сигналу SIGTERM. | Аудит |
| <a id="srs-nfr-infra-01"></a>SRS-NFR-INFRA-01 | Інфраструктура | Сервіси повинні розгортатися як контейнери Docker у Kubernetes (Helm) з не менше ніж 2 репліками stateless-сервісів; інфраструктура повинна описуватися OpenTofu; CI/CD (GitHub Actions) повинен містити Quality Gate. | NFR-I1 |
| <a id="srs-nfr-infra-02"></a>SRS-NFR-INFRA-02 | Інфраструктура | Кожен pod прикладного сервісу повинен мати задані `requests`/`limits` (не більше 500m CPU і 256 MiB RAM); сумарне споживання тестового кластера повинно не перевищувати 8 vCPU і 16 GiB RAM. | Аудит |
| <a id="srs-nfr-obs-01"></a>SRS-NFR-OBS-01 | Спостережуваність | 100 % HTTP/gRPC-запитів і повідомлень Kafka повинні містити `trace_id` (OpenTelemetry, W3C Trace Context); метрики consumer lag, Latency і throughput повинні відображатися в Grafana. | NFR-O1 |
| <a id="srs-nfr-obs-02"></a>SRS-NFR-OBS-02 | Спостережуваність | Prometheus повинен формувати алерт за consumer lag > 5 000 повідомлень довше 2 хв та за будь-якого приросту кількості повідомлень у Dead Letter Topic. | Аудит |
| <a id="srs-nfr-qual-01"></a>SRS-NFR-QUAL-01 | Якість | Покриття модульними тестами шарів domain та application повинно бути ≥ 70 %; golangci-lint повинен завершуватися без помилок; SonarQube Quality Gate повинен мати статус Passed. | NFR-Q1 |
| <a id="srs-nfr-qual-02"></a>SRS-NFR-QUAL-02 | Якість | Схеми подій (Protocol Buffers) повинні реєструватися в Schema Registry з режимом сумісності `BACKWARD`; несумісна зміна схеми повинна блокувати pipeline CI. | Аудит |
| <a id="srs-nfr-loc-01"></a>SRS-NFR-LOC-01 | Локалізація | Система повинна підтримувати не менше 3 валют (UAH, EUR, USD; ISO 4217) і 2 мов (uk, en). | NFR-L1 |

---

## 5. Критерії верифікації (Verification Criteria)

Методи верифікації за ISO/IEC/IEEE 29148: **T** — Test (тестування), **A** — Analysis (аналіз), **I** — Inspection (інспекція), **D** — Demonstration (демонстрація).

| Вимога | Метод | Інструмент | Критерій приймання |
|---|---|---|---|
| [SRS-FUN-01](#srs-fun-01), [SRS-FUN-03](#srs-fun-03) | T | Go test + testify; Postman/Newman (колекція API) | Токен із простроченим `exp` або чужим `iss` → `401`; 21-й запит серії без паузи (ємність bucket 20) → `429` з `Retry-After` |
| [SRS-FUN-08](#srs-fun-08), [SRS-FUN-09](#srs-fun-09) | T | Testcontainers (PostgreSQL, Kafka), Go test | Два запити з однаковим `Idempotency-Key` → одне замовлення в БД, ідентичні тіла відповідей |
| [SRS-FUN-10](#srs-fun-10), [SRS-NFR-SEC-03](#srs-nfr-sec-03) | T | Postman/Newman; OWASP ZAP baseline у CI | Запит чужого `order_id` → `404`; ZAP: 0 знахідок High |
| [SRS-FUN-12](#srs-fun-12), [SRS-NFR-CONS-03](#srs-nfr-cons-03) | T | Testcontainers; chaos-сценарій (Chaos Mesh, PodChaos) | Зупинка процесу після `COMMIT` до публікації: після перезапуску подія присутня в Kafka; кількість подій у топіку = кількості записів `outbox` |
| [SRS-FUN-14](#srs-fun-14), [SRS-NFR-CONS-01](#srs-nfr-cons-01) | T | Go test `-race` + Testcontainers (Redis); k6 (10 000 VU-ітерацій) | Кількість успішних резервувань ≤ 1 000; залишок ≥ 0 |
| [SRS-FUN-15](#srs-fun-15) | T | Go test з керованим годинником (fake clock) | Резерв без `PaymentAuthorized` звільняється через 15 хв ± 1 хв |
| [SRS-FUN-21](#srs-fun-21), [SRS-FUN-22](#srs-fun-22) | T | Інтеграційний тест Saga (Testcontainers: Kafka, PostgreSQL, Redis) | Сценарій `PaymentFailed` → статус `CANCELLED`, залишок відновлено, подія `InventoryReleased` присутня |
| [SRS-FUN-23](#srs-fun-23), [SRS-NFR-CONS-02](#srs-nfr-cons-02) | T | k6 + xk6-kafka (ін'єкція 5 % дублікатів) | Кількість записів `payments` зі статусом `AUTHORIZED` = кількості унікальних `order_id` |
| [SRS-FUN-24](#srs-fun-24), [SRS-NFR-REL-03](#srs-nfr-rel-03) | T | Інтеграційний тест із «отруєним» повідомленням | 5 спроб зі зростаючою затримкою; повідомлення в `<topic>.dlt`; партиція не блокується |
| [SRS-NFR-PERF-01](#srs-nfr-perf-01), [SRS-NFR-PERF-02](#srs-nfr-perf-02) | T | k6 (сценарій constant-arrival-rate, 500 RPS, 10 хв), Prometheus | k6 thresholds: `http_req_duration{p(95)}<200`, `p(99)<500`, `http_req_failed<0.001` |
| [SRS-NFR-PERF-04](#srs-nfr-perf-04) | A | Jaeger (трасування Saga), Prometheus histogram | p95 тривалості трасування `OrderCreated → OrderConfirmed/OrderCancelled` ≤ 3 с |
| [SRS-NFR-SCAL-01](#srs-nfr-scal-01) | T | k6 (Spike Testing: 500 → 1000 → 500 RPS), Kafka Exporter | `kafka_consumergroup_lag` < 1 000 через ≤ 60 с після піку |
| [SRS-NFR-SCAL-02](#srs-nfr-scal-02) | T | k6, Kubernetes HPA, Kafka Exporter | Throughput(2N реплік) / Throughput(N реплік) ≥ 1,6 |
| [SRS-NFR-CONS-04](#srs-nfr-cons-04) | T | Інтеграційний тест (перезапуск Redis), Prometheus | `inventory_drift_items` = 0 через ≤ 60 с |
| [SRS-NFR-SEC-01](#srs-nfr-sec-01) | I, T | Конфігурація Keycloak (realm export), Go test | Алгоритм RS256, `accessTokenLifespan` ≤ 900 с |
| [SRS-NFR-SEC-02](#srs-nfr-sec-02), [SRS-NFR-SEC-04](#srs-nfr-sec-04) | T | Gitleaks, Trivy (image + config) у GitHub Actions | 0 секретів; 0 CRITICAL; 0 HIGH із доступним виправленням; інакше pipeline зупиняється |
| [SRS-NFR-SEC-05](#srs-nfr-sec-05) | I, T | Модульні тести маскувальника журналу; аудит вибірки логів у Loki/stdout | 0 входжень шаблонів email/телефону у вибірці журналів k6-прогону |
| [SRS-NFR-REL-01](#srs-nfr-rel-01) | T | Chaos Mesh (PodChaos `pod-kill`) під навантаженням k6 | `http_req_failed` ≤ 1 % у вікні 60 с |
| [SRS-NFR-REL-02](#srs-nfr-rel-02) | I | `kafka-topics.sh --describe`, Helm values | RF=3, `min.insync.replicas=2` для всіх топіків Saga |
| [SRS-NFR-REL-04](#srs-nfr-rel-04) | T | Go test з емулятором провайдера (WireMock), що повертає 5xx | Стан Open після 10 помилок із 20; Half-Open через 30 с |
| [SRS-NFR-REL-05](#srs-nfr-rel-05) | D | Тест відновлення з WAL-архіву (pgBackRest) | Відновлення ≤ 30 хв; втрата даних ≤ 5 хв |
| [SRS-NFR-INFRA-02](#srs-nfr-infra-02) | I | `kubectl describe`, Helm lint, Prometheus (kube-state-metrics) | Усі pod мають `requests`/`limits`; пікове споживання ≤ 8 vCPU / 16 GiB |
| [SRS-NFR-OBS-01](#srs-nfr-obs-01) | A | Jaeger, OpenTelemetry Collector | Частка спанів без батьківського контексту на межі Kafka = 0 |
| [SRS-NFR-QUAL-01](#srs-nfr-qual-01), [SRS-NFR-QUAL-02](#srs-nfr-qual-02) | T | `go test -cover`, golangci-lint, SonarQube, Schema Registry compatibility check | Покриття ≥ 70 %; Quality Gate Passed; несумісна схема → CI failed |

---

## 6. Глосарій (Glossary)

| Термін | Визначення |
|---|---|
| **Eventual Consistency** | Модель узгодженості, за якої репліки та сервіси після завершення обробки всіх подій переходять в узгоджений стан, проте в проміжку можуть повертати різні значення. |
| **Choreography-based Saga** | Спосіб виконання розподіленої транзакції як послідовності локальних транзакцій сервісів, що реагують на події один одного без центрального координатора; збій кроку компенсується зворотними діями. |
| **Transactional Outbox** | Патерн, за якого подія записується в таблицю `outbox` у тій самій транзакції БД, що й бізнес-дані, а окремий relay-процес публікує її в брокер; усуває неатомарність подвійного запису. |
| **Idempotent Consumer** | Консюмер, який фіксує ідентифікатор обробленої події (`event_id`) і пропускає повторні доставки, що забезпечує ефект exactly-once на рівні бізнес-логіки за семантики at-least-once. |
| **Dead Letter Topic (DLT)** | Окремий топік Kafka для повідомлень, обробка яких не вдалася після вичерпання повторних спроб; запобігає блокуванню партиції. |
| **p95 Latency** | 95-й перцентиль часу відповіді: значення, яке не перевищують 95 % запитів у вибірці вимірювання. |
| **JWT (JSON Web Token)** | Підписаний токен (RFC 7519), що містить твердження (claims) про суб'єкта; використовується для stateless-автентифікації запитів. |

---

## 7. Матриця трасування (Requirements → Architecture)

| Вимоги | Контейнер C4 (Рівень 2) | Компонент C4 (Рівень 3) | Рішення |
|---|---|---|---|
| SRS-FUN-01…03 | [API Gateway](2_design.md#container-api-gateway) | — | — |
| SRS-FUN-04…07 | [Catalog Service](2_design.md#container-catalog-service) | — | — |
| SRS-FUN-08…11, 21 | [Order Service](2_design.md#container-order-service) | [Order Service: компоненти](2_design.md#c4-level3) | [ADR-001](adr/0001-choice-of-technology.md) |
| SRS-FUN-12, 13 | [Order / Inventory / Payment Service](2_design.md#container-order-service) | [Outbox Relay Worker](2_design.md#comp-outbox-relay) | [ADR-001](adr/0001-choice-of-technology.md) |
| SRS-FUN-14…17 | [Inventory Service](2_design.md#container-inventory-service) | — | — |
| SRS-FUN-18…20 | [Payment Service](2_design.md#container-payment-service) | — | — |
| SRS-FUN-22…24 | [Apache Kafka](2_design.md#container-kafka) та консюмери всіх сервісів | [Saga Event Consumer](2_design.md#comp-saga-consumer) | [ADR-001](adr/0001-choice-of-technology.md) |
| SRS-FUN-25 | [Notification Service](2_design.md#container-notification-service) | — | — |
| SRS-NFR-SEC-01 | [Identity Provider](2_design.md#container-identity-provider) | [Auth Interceptor](2_design.md#comp-auth-interceptor) | — |
| SRS-NFR-OBS-01, 02 | [Спостережуваність](2_design.md#c4-level2-infra) | [Telemetry Interceptor](2_design.md#comp-telemetry) | — |
