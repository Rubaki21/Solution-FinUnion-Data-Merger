## План на 12 месяцев

Этапы частично перекрываются, но каждый production cutover зависит от прохождения gate предыдущих результатов

|   № | период  | этап                                       | содержание                                                                                                                        | результат / exit gate                                                                         |
| --: | ------- | ------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
|   0 | М1      | **Mobilization и scope**                   | governance, BIA, migration principles, приоритет продуктов, environments, change freeze policy, коммуникации                      | утверждены scope, budget, owners, RACI, критичность, success/abort criteria                   |
|   1 | М1–М2   | **Data discovery и dependency mapping**    | реестр систем/таблиц/API/files/jobs/reports/consumers; profiling; классификация; BIAN mapping; retention/legal hold               | каталог критичных данных, dependency graph, DQ baseline, decommission candidates              |
|   2 | М2–М3   | **Foundation и MVP**                       | Party crosswalk/MDM, reference mappings, Integration adapters, отдельный PostgreSQL DWH, Airflow, первые DataHub metadata/lineage | единый Party ID; четыре витрины; source-to-target controls; промежуточный результат 3 месяцев |
|   3 | М3–М6   | **Target platform build**                  | Kafka/Schema Registry, CDC/outbox, object storage, Iceberg zones, Trino pilot, security zones, observability, CI/CD/IaC           | platform NFR/security/performance tests; replay/DR; supported pipeline templates              |
|   4 | М4–М7   | **Canonicalization и analytics migration** | canonical contracts, Product/Reference MDM, conformed dimensions, target DWH/semantic layer, dual-run FU/RB reports               | отчёты сверены; Finance/business sign-off; reporting cutover ready                            |
|   5 | М6–М9   | **Domain pilots и data products**          | Customer 360, Payments/Transactions, Credit Portfolio pilots; shadow reads; trial migration; operational readiness                | products registered in DataHub; SLO/DQ achieved; pilot domain go/no-go                        |
|   6 | М8–М11  | **Operational migration waves**            | active agreements/accounts/cards/loans/cases by product; snapshot + CDC catch-up; target write cutover; hypercare                 | target master для выбранных scopes; financial/business reconciliation; fallback closed        |
|   7 | М10–М12 | **Consolidation и decommission**           | switch remaining consumers, stop legacy writes/jobs/P2P, archive/restore test, revoke access, close licenses, BAU handover        | RB DWH/BI off; migrated modules retired; residual legacy has owner and dated exit plan        |

## Детализация этапов

### Этап 0. Mobilization и scope

**Работы**

- создать Migration Steering Committee и Go/No-Go Board
- назначить Business/Data/Technical Owners и Migration Lead по доменам
- определить критичные процессы, окна, RPO/RTO и запретные периоды
- выбрать первые волны по бизнес-ценности и сложности
- сформировать defect, exception и decision logs
- согласовать, какие временные решения обязаны быть удалены к 12-му месяцу

**Gate**

- scope привязан к сущностям, системам и продуктам
- у каждого deliverable есть owner и критерий приёмки
- согласованы rollback authority и максимальная длительность cutover

### Этап 1. Data discovery

**Работы**

- автоматический scan и ручная инвентаризация FU/RB
- поиск скрытых point-to-point, cron/batch, Excel/manual extracts
- profiling ключей, дублей, null, кодов, временных полей и объёмов
- фиксация source-of-entry/source-of-record и BIAN domain
- построение lineage source -> report
- анализ legal retention, PCI/ПДн и архивов

**Gate**

- для каждого критичного dataset известны owner, source key, volume/growth, classification, consumers
- неопознанный интерфейс блокирует decommission затронутой системы
- DQ baseline принят owner, а не скрыт очисткой

### Этап 2. Foundation и MVP на 3 месяца

**Работы**

- загрузить FU/RB Party candidates и настроить match/merge/steward queue
- создать `global_party_id` и immutable crosswalk
- согласовать mappings продуктов, статусов, валют, сегментов и отделений
- построить CDC/incremental adapters с manifests
- развернуть PostgreSQL DWH schemas staging/integration/core/marts
- опубликовать витрины клиентов, продуктов/договоров, счетов/остатков и операций
- завести critical datasets/flows в DataHub

**Gate**

- auto-match rules протестированы на размеченной выборке
- ambiguous Party не merge автоматически
- DWH сверяется с source по count/sum/hash и balances
- аналитика не запрашивает production OLTP напрямую

### Этап 3. Target platform

**Работы**

- Kafka topics, Schema Registry, ACL и compatibility policy
- outbox/CDC connectors и DLQ/replay
- object storage zones, encryption, Iceberg catalogs/tables
- Airflow HA, deployment process, secrets и lineage
- Trino resource groups/policies
- centralized observability, cost controls и DQ service
- backup/restore, failover и performance tests

**Gate**

- synthetic peak + forecast load пройден
- duplicate/out-of-order/replay tests не нарушают результат
- security tests подтверждают masking, tokenization и audit
- отказ DataHub/BI/Lakehouse не блокирует клиентские транзакции

### Этап 4. Canonicalization и analytics migration

**Работы**

- утвердить canonical schemas и time semantics
- загрузить историю в Lakehouse, conformed data - в target DWH
- перенести metric definitions в semantic layer
- выполнить минимум согласованное число параллельных period closes
- обучить пользователей и перевести report subscriptions
- провести reporting cutover из [cutover-plan.md](cutover-plan.md)

**Gate**

- нет необъяснённых финансовых расхождений
- все critical reports имеют owner, certified flag и lineage
- RB DWH/BI готовы к read-only evidence window

### Этап 5. Domain pilots

Пилот выбирается по принципу «достаточно значим, но ограничен»: один регион/продукт/портфель без сложных исключений

**Работы**

- trial snapshot и CDC catch-up
- shadow target processing без влияния на клиента
- dual read/comparison и business scenario testing
- проверка customer service, operations, Finance, Risk и ИБ
- rollback rehearsal на production-like объёме
- поддержка и обучение frontline

**Gate**

- data product SLO/DQ и target capacity подтверждены
- все critical use cases пройдены
- rollback выполняется в допустимое окно
- Go/No-Go Board разрешает первую production wave

### Этап 6. Operational migration waves

Каждая волна проходит один шаблон:

1. **Select scope:** продукт/регион/портфель и исключения
2. **Clean/map:** исправление в source, MDM/reference mapping
3. **Initial load:** snapshot в staging/target
4. **Catch-up:** CDC до cutover watermark
5. **Validate:** техническая, бизнес- и финансовая сверка
6. **Switch:** route новых команд в target master
7. **Hypercare:** усиленный мониторинг и reconciliation
8. **Close fallback:** после стабильного периода legacy write блокируется окончательно

Рекомендуемая последовательность при подтверждённой сложности:

- CRM/open cases и customer servicing
- новые продажи продуктов сразу на target
- простой депозитный/счётный portfolio
- кредиты по одному типу
- карты после process/scheme readiness
- сложные корпоративные продукты и исключения - последними

Последовательность уточняется discovery; календарь не заменяет оценку рисков

### Этап 7. Consolidation и decommission

**Работы**

- закрыть остаточные consumers и временные adapters
- остановить legacy schedulers и входящие write endpoints
- перевести систему read-only
- архивировать data/schema/config/logs и проверить restore/read
- отозвать service accounts, network routes и privileged access
- обновить CMDB, DataHub, runbooks, DR и vendor contracts
- передать target в BAU с capacity/cost baseline

**Gate**

- нулевой неутверждённый трафик к legacy
- нет открытых Sev-1/Sev-2
- legal/records/Finance/ИБ/business/IT sign-off
- инфраструктура выключена или остаточный read-only scope формально исключён с датой выхода

## Управление волнами

| Артефакт | Содержание |
|---|---|
| Wave charter | scope, owners, source/target, объём, зависимости, окно |
| Mapping specification | source->target fields, codes, defaults, transformations, version |
| Reconciliation pack | counts, sums, hashes, relationships, balance, sample/business checks |
| Cutover runbook | шаг, команда, исполнитель, start/end, evidence, abort criterion |
| Rollback plan | restore point, reverse routing, treatment of target writes, decision authority |
| Hypercare dashboard | SLO, errors, lag, DQ, financial difference, complaints |
| Decommission checklist | consumers, archive, restore, IAM/network, contracts, sign-off |

## Go/No-Go governance

| Роль | Решение |
|---|---|
| Business Owner | клиентский процесс и приемлемость результата |
| Data Owner/Steward | mapping, качество и исключения |
| Chief Accountant/Finance | финансовая сверка и закрытый период |
| CISO/Privacy | доступ, ПДн/PCI, audit и incident readiness |
| Technical Owners | capacity, backup/restore, monitoring и rollback |
| Migration Lead | целостность runbook, evidence и статус зависимостей |

No-Go объявляет любой владелец для своего обязательного gate. Решение Go документируется с timestamp и версией reconciliation pack

## Сохранение доступности

- initial load выполняется из snapshot/read replica
- изменения после snapshot доставляются CDC
- короткий freeze применяется только к мигрируемому объекту/конфигурации, не ко всему банку
- новые продажи можно направить в target раньше back-book migration
- read traffic переключается через versioned endpoint/configuration
- очереди буферизуют асинхронные события
- при недоступности аналитики Core/Payments продолжают работу
