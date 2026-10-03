## Решение по горизонтам

| горизонт       | DWH                                                                                             | Lakehouse                                                                                  | gричина                                                     |
| -------------- | ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ | ----------------------------------------------------------- |
| **3 месяца**   | PostgreSQL DWH: staging, integration/core, marts, audit/control                                 | Объектная landing/raw-зона для исходных файлов; без обязательной сложной lakehouse-логики  | Минимальный риск и быстрый выпуск четырёх витрин            |
| **12 месяцев** | Enterprise DWH для conformed dimensions, сертифицированных фактов, BI и регуляторной отчётности | Object Storage + Apache Iceberg для raw/standardized/curated history/events; Trino для SQL | Разделить стабильную отчётность и большие/изменчивые данные |

Принцип: данные размещаются не по правилу «всё сначала в Lakehouse» или `всё в DWH`, а по требованиям к гранулярности, изменчивости, объёму, сертификации и потребителям

## PostgreSQL DWH для FinUnion + RetailBank

### Логическая структура на три месяца

| Схема              | содержимое                                                                  | правило доступа                            |
| ------------------ | --------------------------------------------------------------------------- | ------------------------------------------ |
| `stg_fu`, `stg_rb` | инкременты/снимки источников в исходной гранулярности                       | только pipeline service accounts и support |
| `int`              | типизированные канонические модели, source keys, mappings, quarantine links | data engineering и DQ                      |
| `core`             | исторические conformed dimensions и reconciled facts                        | controlled read; изменения только pipeline |
| `mart_customer`    | Customer 360 analytical view                                                | CRM/BI по row/column policy                |
| `mart_accounts`    | счета и EOD-остатки                                                         | Finance/BI                                 |
| `mart_products`    | продукты, договоры и holdings                                               | Product/BI                                 |
| `mart_operations`  | операции и согласованные обороты                                            | Finance/BI/Risk по разрешениям             |
| `ctl`              | watermarks, manifests, count/sum/hash, batch status, certification          | platform, audit, owners                    |

### Ограничители нагрузки

- отдельный кластер от production OLTP
- партиционирование крупных фактов по `business_date`, индексы по surrogate/source/global keys
- bulk load в staging, set-based transformations и atomic publish
- read replica/serving copy для тяжёлого BI при необходимости
- connection/workload limits и запрет raw ad hoc queries
- retention детальных staging-данных после подтверждённого архива в object storage

PostgreSQL остаётся допустимым целевым DWH только после нагрузочного теста с прогнозным годовым объёмом, backfill и одновременными BI-запросами. Если запас по SLA недостаточен, физический движок DWH меняется на MPP, а логические модели/контракты сохраняются

## Data Lakehouse

### Зоны

| зона                      | формат                                | назначение                                                           | изменяемость                                |
| ------------------------- | ------------------------------------- | -------------------------------------------------------------------- | ------------------------------------------- |
| **Landing**               | исходные файлы/объекты + manifest     | кратковременный приём, malware/checksum validation                   | объект не перезаписывается                  |
| **Raw / Bronze**          | Iceberg; payload + metadata источника | полная replayable история CDC/events/archive                         | append; исправление новой записью           |
| **Standardized / Silver** | Iceberg с каноническими типами        | dedup, normalize, tokenize, source/global keys, schema harmonization | MERGE по version rules, snapshots сохранены |
| **Curated / Gold**        | Iceberg data products                 | готовые доменные наборы, features, большие агрегаты                  | публикуется атомарным snapshot после DQ     |

Обозначение Gold в Lakehouse не означает, что набор автоматически сертифицирован для регуляторной отчётности. Certification status, owner и purpose фиксируются отдельно

## Роль Apache Iceberg

Iceberg выбран как открытый table format поверх объектного хранилища:

- атомарная публикация набора файлов через snapshot
- time travel и rollback ошибочной публикации
- schema evolution без ручного чтения каждой исторической схемы
- partition evolution без изменения бизнес-контракта
- hidden partitioning и metadata pruning
- row-level delete/update/MERGE для управляемых исправлений
- совместное чтение несколькими engines

Iceberg не решает автоматически:

- дедупликацию и бизнес-качество
- выбор правильного partition/layout
- доступ на уровне бизнес-цели
- compaction и удаление старых snapshots/orphan files
- согласование с Core Ledger

Поэтому для каждой таблицы нужны maintenance jobs: compaction, manifest rewrite, statistics collection, snapshot expiration и orphan-file cleanup с retention/legal hold

## Роль Trino

Trino используется как распределённый SQL query engine:

- интерактивное чтение Standardized/Curated Iceberg
- объединение разрешённых Lakehouse-наборов между доменами
- исследовательская аналитика и подготовка выборок
- materialization результата в Curated или DWH через управляемый pipeline
- единый SQL-интерфейс для data analysts без выдачи доступа к raw object paths

Ограничения Trino:

- не является хранилищем или мастером
- federation не должна превращаться в production join к Core/CRM
- тяжёлые повторяемые joins следует материализовать
- нужны resource groups, query limits, cost monitoring и row/column policies
- регуляторный отчёт не должен зависеть от ad hoc federated query

## Какие данные остаются в DWH

| данные                                                                 | почему DWH                                              |
| ---------------------------------------------------------------------- | ------------------------------------------------------- |
| Conformed dimensions Party/Product/Agreement/Account/Calendar/Currency | единая семантика, SCD2, стабильные joins                |
| Факты бухгалтерских проводок                                           | строгая гранулярность, сверка с GL и аудит              |
| EOD-остатки и кредитная задолженность                                  | закрытие дня и воспроизводимость периода                |
| Согласованные платежи/операции                                         | стабильные бизнес-метрики и BI                          |
| Регуляторные и финансовые витрины                                      | certification, maker-checker, immutable report snapshot |
| Управленческие KPI/semantic layer                                      | единые формулы, ownership и эффективная BI-нагрузка     |

DWH может хранить необходимую деталь, но не обязан хранить каждую техническую версию события, raw JSON или многократно переигрываемый архив

## Какие данные попадают в Lakehouse

| данные                                             | зона                                        | почему Lakehouse                               |
| -------------------------------------------------- | ------------------------------------------- | ---------------------------------------------- |
| Полный CDC/outbox и доменные события               | Raw -> Standardized                          | replay, audit trail, schema evolution          |
| Многолетняя история карточных авторизаций/клиринга | Raw/Standardized/Curated                    | большой объём и потоковая обработка            |
| Clickstream мобильного/веб-канала                  | Raw/Standardized                            | высокая скорость, полуструктурированность      |
| Записи/транскрипты контакт-центра                  | Object + governed Iceberg metadata/features | файлы и производные NLP                        |
| Архивные FU/RB batch-выгрузки                      | Landing/Raw                                 | сохранение оригинала, постепенная нормализация |
| Логи антифрода, device/network events              | Standardized/Curated                        | оконные признаки и graph/ML preparation        |
| Training datasets и features                       | Curated                                     | воспроизводимые snapshots для моделей          |
| Детальные исторические версии до агрегации         | Standardized                                | backfill и исследовательская аналитика         |

## Данные в обоих контурах

Копирование допустимо, если роли различны:

| набор      | Lakehouse                                                        | DWH                                 | контроль                                                     |
| ---------- | ---------------------------------------------------------------- | ----------------------------------- | ------------------------------------------------------------ |
| Транзакции | полная event-level история и технические версии                  | канонический сертифицированный факт | общий `transaction_id`, source key, count/sum reconciliation |
| Party      | история входных записей и match features с ограниченным доступом | golden dimension/bridge             | MDM version и global ID                                      |
| Продукт    | локальные source snapshots                                       | conformed dimension/version         | Product Catalog version                                      |
| Остатки    | raw intraday + EOD source snapshots                              | сертифицированный EOD fact          | manifest и balance roll-forward                              |

Наличие копии не создаёт второго мастера. System of Record остаётся операционным; MDM/DWH/Lakehouse имеют явно заданные производные роли.

## Ограничения и риски выбранного подхода

| ограничение               | риск                                         | компенсация                                                                  |
| ------------------------- | -------------------------------------------- | ---------------------------------------------------------------------------- |
| Два аналитических контура | дублирование и спор, где `правда`            | dataset certification, purpose, owner и source-of-truth metadata в DataHub   |
| Больше технологий         | дефицит компетенций и on-call complexity     | platform team, IaC, шаблоны pipeline, ограниченный supported stack           |
| Iceberg maintenance       | small files/metadata growth ухудшают запросы | compaction, snapshot policy, observability, workload tests                   |
| Federation через Trino    | дорогие unpredictable joins                  | resource groups, limits, materialized products, запрет OLTP connectors       |
| PostgreSQL interim debt   | временная модель становится постоянной       | exit criteria, migration backlog, совместимые logical models                 |
| Разные security engines   | несогласованный доступ DWH/Lakehouse         | central IAM/policies, automated entitlement tests, quarterly recertification |
| Late events и backfill    | изменение закрытых метрик                    | bitemporal model, controlled reopen, adjustment facts, report version        |
| Vendor/tool sprawl        | рост стоимости                               | open formats, FinOps, capacity/cost SLO, ежегодный architecture review       |
