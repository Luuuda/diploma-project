# Thesis Proposal

| Поле | Значення |
| --- | --- |
| Студент | Андрющенко Людмила Анатоліївна |
| Науковий керівник | Добровольський Юрій Георгійович |
| Спеціальність | Інженерія програмного забезпечення |
| Рівень | Бакалавр |

## 1. Робоча назва проєкту

**UA:** Розробка масштабованої подієво-орієнтованої екосистеми для мультинаціонального маркетплейсу

**EN:** Development of a Scalable Event-Driven Ecosystem for a Multinational Marketplace (EventMarket)

## 2. Актуальність та проблематика

Мультинаціональні маркетплейси обробляють різке зростання навантаження під час пікових подій (розпродажі, сезонні акції), коли кількість замовлень збільшується в десятки разів упродовж короткого часу. Монолітні та синхронні архітектури за таких умов мають такі обмеження:

- **Каскадні відмови.** Синхронні виклики між сервісами (замовлення, склад, оплата) призводять до поширення збою одного компонента на всю систему.
- **Порушення консистентності даних.** Запис у базу даних і публікація події в брокер не є атомарною операцією. Це призводить до втрати подій або «фантомних» повідомлень.
- **Дублювання обробки.** Механізм доставки «at-least-once» призводить до повторної обробки повідомлень, що спричиняє подвійні списання та повторне резервування товару.
- **Перепродаж товару (overselling).** Конкурентні запити на резервування обмеженого залишку без атомарних операцій порушують цілісність складських даних.
- **Локалізація.** Робота в кількох регіонах потребує підтримки різних валют, податкових правил і мов без дублювання бізнес-логіки.

Складність задачі полягає в забезпеченні кінцевої консистентності (Eventual Consistency) розподілених даних, гарантій доставки та ідемпотентності обробки за умов пікових навантажень і часткових відмов інфраструктури.

---

## 3. Сформована ідея продукту

**Тип системи:** розподілена подієво-орієнтована мікросервісна платформа електронної комерції.

**Архітектурний стиль:** Event-Driven Architecture на базі Apache Kafka; Choreography-based Saga для розподілених транзакцій замовлення; Transactional Outbox на стороні відправника; ідемпотентні консюмери; ізоляція шарів кожного сервісу за Clean Architecture.

**Склад системи:**

| Сервіс | Відповідальність |
| --- | --- |
| API Gateway | Єдина точка входу, автентифікація, Rate Limiting |
| Catalog Service | Каталог товарів, ціни за регіонами |
| Order Service | Життєвий цикл замовлення, ініціація Saga |
| Inventory Service | Резервування та звільнення залишків |
| Payment Service | Авторизація та повернення платежів (емуляція платіжного провайдера) |
| Notification Service | Розсилка сповіщень про статус замовлення |

Сервіси взаємодіють асинхронно через Kafka; синхронні виклики (gRPC) застосовуються лише для запитів на читання.

---

## 4. Функціональні вимоги (Core Features)

1. **Оркестрація замовлення через Saga.** Choreography-based Saga (`OrderCreated → InventoryReserved → PaymentAuthorized → OrderConfirmed`) з компенсуючими діями (`InventoryReleased`, `PaymentRefunded`, `OrderCancelled`) при збої на будь-якому кроці.
2. **Transactional Outbox.** Атомарний запис бізнес-даних і події в одній транзакції PostgreSQL із подальшою публікацією в Kafka окремим relay-процесом (гарантія «at-least-once»).
3. **Ідемпотентні консюмери.** Дедуплікація повідомлень за унікальним ідентифікатором події (таблиця оброблених подій або Redis), що забезпечує ефект «exactly-once» на рівні бізнес-логіки.
4. **Атомарне резервування залишків.** Захист від перепродажу за допомогою атомарних операцій Redis (Lua-скрипти) з подальшою асинхронною синхронізацією з PostgreSQL.
5. **Мультирегіональність і мультивалютність.** Регіональні ціни, конвертація валют, локалізація повідомлень; партиціонування топіків Kafka за ключем замовлення для збереження порядку подій.
6. **Механізми відмовостійкості.** Retry з експоненційною затримкою, Dead Letter Topic для необроблюваних повідомлень, Circuit Breaker для викликів платіжного провайдера, Rate Limiting (Token Bucket) на шлюзі.
7. **Спостережуваність і навантажувальна верифікація.** Розподілена трасація (OpenTelemetry) через межі брокера, метрики Prometheus (consumer lag, latency, throughput), дашборди Grafana, сценарії пікового тестування k6.

---

## 5. Попередній інструментарій та технологічний стек

| Категорія | Технології |
| --- | --- |
| Мови програмування | Go 1.22, SQL, Lua, TypeScript, YAML, Shell |
| Архітектурні патерни | Event-Driven Architecture, Saga (Choreography), Transactional Outbox, Idempotent Consumer, CQRS (читання каталогу окремо від запису), Circuit Breaker, Retry with Backoff, Dead Letter Queue, Clean Architecture, API Gateway, C4 Model, ADR |
| Фреймворки та бібліотеки | Gin або Fiber (HTTP), gRPC, segmentio/kafka-go або Sarama, pgx, go-redis, Protocol Buffers, Testcontainers for Go, React (демонстраційний клієнт) |
| Бази даних та брокери | Apache Kafka (KRaft), Schema Registry, PostgreSQL (за принципом «база даних на сервіс»), Redis |
| Тестування та якість | Go testing, testify, Testcontainers, golangci-lint, SonarQube, k6 |
| DevOps та DevSecOps | Docker, Docker Compose, Kubernetes (Helm), GitHub Actions, Trivy, Gitleaks, OpenTofu |
| Observability | OpenTelemetry, Prometheus, Grafana, Jaeger, Kafka Exporter |
| Документування | Markdown (Docs-as-Code), OpenAPI/Swagger, AsyncAPI, PlantUML/Mermaid |

**Обґрунтування вибору:** Go забезпечує високу продуктивність і низьке споживання ресурсів для мережевих сервісів з великою кількістю конкурентних з'єднань; Apache Kafka гарантує збереження порядку подій у межах партиції, реплікацію й повторне відтворення (replay); Redis надає атомарні операції з мікросекундними затримками для резервування залишків; PostgreSQL забезпечує ACID-транзакції, необхідні для патерну Outbox; Kubernetes і Helm дозволяють горизонтально масштабувати консюмерів; OpenTelemetry забезпечує наскрізну трасацію через асинхронні межі; Trivy, Gitleaks і SonarQube формують контур DevSecOps.

---

## 6. Очікувані результати

- Архітектурна документація (C4-діаграми, ADR, специфікація подій AsyncAPI).
- Працюючий прототип платформи з не менше ніж п'ятьма сервісами, розгорнутий у хмарі.
- Конвеєр CI/CD із лінтингом, тестуванням, скануванням вразливостей і Quality Gate.
- Результати навантажувального тестування (k6): пропускна здатність, затримки, поведінка при збої окремих сервісів, доведення відсутності перепродажу й дублювання платежів.

---

# English Version

## 1. Working Title

Development of a Scalable Event-Driven Ecosystem for a Multinational Marketplace (EventMarket).

## 2. Relevance and Problem Statement

Multinational marketplaces face sharp load spikes during peak events (sales, seasonal promotions), when order volume increases tenfold or more in a short period. Monolithic and synchronous architectures exhibit the following limitations:

- **Cascading failures:** synchronous calls between services (orders, inventory, payments) propagate a single component failure across the system.
- **Data inconsistency:** a database write and a broker publish are not atomic, resulting in lost events or phantom messages.
- **Duplicate processing:** at-least-once delivery causes repeated message handling, leading to double charges and duplicate stock reservations.
- **Overselling:** concurrent reservation requests against limited stock without atomic operations violate inventory integrity.
- **Localization:** operating in multiple regions requires support for different currencies, tax rules and languages without duplicating business logic.

The core difficulty is ensuring eventual consistency of distributed data, delivery guarantees and idempotent processing under peak load and partial infrastructure failures.

## 3. Product Concept

A distributed event-driven microservice e-commerce platform. Architecture: Event-Driven Architecture on Apache Kafka, Choreography-based Saga for distributed order transactions, Transactional Outbox on the producer side, idempotent consumers, and Clean Architecture within each service. Services: API Gateway, Catalog, Order, Inventory, Payment, Notification. Inter-service communication is asynchronous via Kafka; synchronous gRPC is used for read queries only.

## 4. Core Features

1. Order Saga (`OrderCreated → InventoryReserved → PaymentAuthorized → OrderConfirmed`) with compensating actions on failure.
2. Transactional Outbox: atomic write of business data and event in one PostgreSQL transaction, published to Kafka by a relay process.
3. Idempotent consumers with event-ID deduplication (processed-events table or Redis).
4. Atomic stock reservation using Redis Lua scripts to prevent overselling.
5. Multi-region and multi-currency support; Kafka topic partitioning by order key to preserve event ordering.
6. Resilience: exponential backoff retries, Dead Letter Topic, Circuit Breaker, Token Bucket rate limiting.
7. Observability and load verification: OpenTelemetry tracing across the broker, Prometheus metrics (consumer lag, latency, throughput), Grafana dashboards, k6 peak-load scenarios.

## 5. Technology Stack

| Category | Technologies |
| --- | --- |
| Languages | Go 1.22, SQL, Lua, TypeScript, YAML, Shell |
| Architectural patterns | Event-Driven Architecture, Saga (Choreography), Transactional Outbox, Idempotent Consumer, CQRS, Circuit Breaker, Retry with Backoff, Dead Letter Queue, Clean Architecture, API Gateway, C4 Model, ADR |
| Frameworks and libraries | Gin or Fiber, gRPC, kafka-go or Sarama, pgx, go-redis, Protocol Buffers, Testcontainers for Go, React |
| Databases and brokers | Apache Kafka (KRaft), Schema Registry, PostgreSQL (database per service), Redis |
| Testing and quality | Go testing, testify, Testcontainers, golangci-lint, SonarQube, k6 |
| DevOps and DevSecOps | Docker, Docker Compose, Kubernetes (Helm), GitHub Actions, Trivy, Gitleaks, OpenTofu |
| Observability | OpenTelemetry, Prometheus, Grafana, Jaeger, Kafka Exporter |
| Documentation | Markdown (Docs-as-Code), OpenAPI/Swagger, AsyncAPI, PlantUML/Mermaid |

## 6. Expected Results

- Architecture documentation (C4 diagrams, ADRs, AsyncAPI event specification).
- Working prototype with at least five services deployed in the cloud.
- CI/CD pipeline with linting, testing, vulnerability scanning and Quality Gate.
- k6 load test results: throughput, latency, behavior under partial service failures, and evidence of no overselling or duplicate payments.