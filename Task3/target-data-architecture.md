
## Контекст To-Be через 12 месяцев

Архитектура разделена на четыре контура:

1. `Операционный`: принимает команды клиента и хранит авторитетное состояние банковских продуктов
2. `Интеграционный`: обслуживает синхронные API и асинхронные доменные события
3. `Аналитический`: хранит историю, выполняет трансформации и публикует сертифицированные данные
4. `Управляющий`: каталог, lineage, качество, безопасность и наблюдаемость

### 1. Operational Systems

| компонент                         | мастер-данные                                  | назначение                                          |
| --------------------------------- | ---------------------------------------------- | --------------------------------------------------- |
| Target Core Banking / Core Ledger | счета, проводки, ledger balance, core-договоры | ACID-учёт и финансовая целостность                  |
| CRM / Customer Case Management    | обращения, SLA, operational customer context   | обслуживание клиента и контакт-центр                |
| Card Processing                   | карты, авторизации, карточные статусы          | жизненный цикл карты и карточные операции           |
| Loan Platform                     | кредиты, графики, задолженность, просрочка     | розничное и корпоративное кредитование              |
| Payments Platform                 | payment order и execution status               | идемпотентное исполнение платежей                   |
| ECM + object storage              | оригиналы документов, версии, hash, retention  | юридически значимые и неструктурированные материалы |

Системы соответствуют границам BIAN: Current/Savings Account, Financial Accounting, Customer Case Management, Issued Device Administration/Card Authorization, Consumer/Corporate Loan и Payment Order/Execution

Операционные системы:
- выполняют бизнес-команды и локальные ACID-транзакции
- публикуют событие только после commit через transactional outbox или log-based CDC
- предоставляют API для команд, но не дают аналитическим consumers прямой write-доступ
- не зависят синхронно от DWH, Lakehouse, BI или DataHub

### 2. MDM

`Party MDM / Customer Hub` владеет `global_party_id`, crosswalk FU/RB, golden attributes, общими Party relationships и аудитом match/merge/split

`Product Catalog / Reference MDM` владеет опубликованными версиями продуктов, каноническими кодами, mapping локальных справочников и периодами действия

MDM не хранит авторитетные платежи, операции, проводки или текущие остатки. Изменения поступают как предложения от producer; survivorship выполняется на уровне атрибутов, неоднозначные совпадения разбирает Data Steward

### 3. Integration Layer

Слой имеет две части:
- `синхронная:` API Gateway, integration services, service identity, rate limit, валидация команд, маршрутизация в мастер-систему
- `асинхронная:` Kafka-совместимая платформа, Schema Registry, CDC/outbox connectors, durable topics, DLQ и replay

Kafka хранит поток фактов и развязывает producers/consumers, но не заменяет Core или DWH. Ключ партиционирования выбирается по aggregate/business key (`party_id`, `account_id`, `payment_id`) для сохранения локального порядка. Consumers идемпотентны. Для синхронного сценария используется API, когда клиенту нужен немедленный ответ: проверка профиля, создание команды платежа, блокировка карты. Для истории, аналитики, уведомлений и распространения мастер-данных применяется асинхронный поток

### 4. Data Lakehouse

Lakehouse размещается на объектном хранилище:
- `Raw:` неизменяемые source events, CDC и архивные файлы с техническими metadata
- `Standardized:` типизированные, дедуплицированные и нормализованные данные с source/global keys
- `Curated:` доменные data products и большие аналитические наборы

Apache Iceberg обеспечивает табличные snapshots, ACID-commit, schema/partition evolution и time travel. Trino предоставляет интерактивный SQL к Standardized/Curated и может читать разрешённые наборы нескольких каталогов

### 5. DWH

Enterprise DWH содержит:
- conformed dimensions Party, Product, Agreement, Account, Calendar, Currency и Reference Data
- сертифицированные факты операций, проводок, платежей, кредитов и EOD-остатков
- историзацию SCD2 и bitemporal attributes, где нужна воспроизводимость
- витрины регуляторной, финансовой и управленческой отчётности
- semantic layer с владельцем, версией формулы и effective date метрики

DWH загружается из валидированных потоков/Lakehouse и напрямую из MDM для золотых измерений. Закрытый период получает immutable report snapshot и certification status

### 6. Airflow и processing engines

Airflow:
- оркестрирует batch, EOD, backfill, сверки и публикацию витрин
- хранит зависимости и run metadata
- запускает SQL/dbt, Spark jobs и quality gates
- публикует job lineage в DataHub

Airflow не обрабатывает каждое событие Kafka. Streaming выполняет отдельный runtime (например, Kafka Streams/Flink/Spark Structured Streaming), а Airflow контролирует его deployment/checkpoint health и batch reconciliation

### 7. BI и аналитические consumers

- BI читает DWH/semantic layer; row/column-level security применяется до выдачи данных.
- Risk/antifraud/data science читает разрешённые Curated Iceberg tables через Trino либо специализированные serving features
- Регуляторная отчётность формируется из сертифицированного DWH и сверяется с Core Ledger
- Customer 360 - управляемая read model: golden Party из MDM, holdings из операционных APIs/events и аналитические агрегаты. Она не становится мастером

### 8. DataHub

DataHub содержит:
- business glossary и BIAN-маппинг
- dataset/topic/API/job/report registry
- Data Owner, Data Steward и Technical Owner
- классификацию полей, retention и allowed purposes
- schema versions, DQ score и SLO
- technical/column lineage source -> pipeline -> table -> metric -> report

Payload финансовых данных и ПДн в каталог не копируется. Sample data выключен для чувствительных наборов либо маскируется.

## Основные пути данных

### Операционная команда

`Channel -> API Gateway -> domain service -> OLTP commit -> outbox/CDC -> Kafka`

Синхронный ответ зависит только от необходимых операционных компонентов. Публикация события повторяется независимо после commit.

### Golden Party

`CRM/KYC event -> Kafka -> MDM match/merge -> golden event -> CRM/Core/DWH/Lakehouse`

Consumer получает только разрешённые атрибуты. Merge/split историзируется; исходные финансовые факты сохраняют source ID

### Финансовая аналитика

`Core/Cards/Loans -> CDC/events + EOD controls -> Raw/Standardized -> reconciliation -> DWH facts -> semantic layer -> BI/regulator`

Streaming даёт свежесть; EOD manifest и ledger reconciliation подтверждают полноту

### Большая событийная аналитика

`Channels/Cards -> Kafka -> stream processing -> Iceberg -> Trino -> antifraud/risk/data science`

В DWH попадают только согласованные факты и агрегаты, необходимые для стабильной отчётности

## Надёжность и контроль качества

| область             | механизм                                                                        |
| ------------------- | ------------------------------------------------------------------------------- |
| Доставка            | durable topics, replication, retry/backoff, DLQ, replay                         |
| Идемпотентность     | `event_id`, source version, upsert/merge rules, dedup window                    |
| Полнота             | watermark/offset, manifest, count/sum/hash, periodic snapshot diff              |
| Финансовая точность | debit=credit, balance roll-forward, source-to-DWH reconciliation                |
| Восстановление      | versioned object storage, Iceberg snapshots, DWH PITR/backups, tested restore   |
| Наблюдаемость       | lag, freshness, DQ, reconciliation, cost, lineage completeness и SLO dashboards |
| Изменение схемы     | Schema Registry, compatibility policy, consumer contract tests                  |
