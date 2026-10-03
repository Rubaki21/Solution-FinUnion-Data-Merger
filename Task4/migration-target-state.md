## Образ состояния на конец 12-го месяца

| вопрос                            | целевое решение                                                                                                                                                                                                                                                                                                    |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Какие системы остаются**        | Target Core Banking/Core Ledger, целевые CRM/Customer Case, Card Processing, Loan Platform и Payments Platform; Party MDM/Customer Hub; Product Catalog/Reference MDM; API/Integration Layer; Kafka + Schema Registry; DWH; Lakehouse на Object Storage + Iceberg; Trino; Airflow; DataHub; BI/Semantic Layer; ECM |
| **Какие системы выводятся**       | RB DWH и RB BI; дублирующие отчётные batch jobs; мигрированные point-to-point; локальные customer/product mappings вне MDM; RB CRM как мастер Party после переноса кейсов/каналов; RB Core/Cards/Loans — помодульно после migration gates                                                                          |
| **Какие данные мигрируются**      | golden Party/crosswalk; продукты/справочники; активные договоры, счета, карты и кредиты; открытые платежи/кейсы; необходимые operational history и документы; полная требуемая история в DWH/Lakehouse/архиве; metadata/lineage                                                                                    |
| **Какие потоки переключаются**    | onboarding и Party updates через MDM; доменные события через Kafka; синхронные команды через API Layer; CDC/EOD в DWH/Lakehouse; BI/регуляторные отчёты на единый DWH; metadata/lineage в DataHub                                                                                                                  |
| **Что остаётся только в архиве**  | закрытые договоры и операции вне operational retention, исходные batch-файлы, версии отчётов, документы/аудит, legacy snapshots и схемы — в защищённом immutable/read-only хранении                                                                                                                                |
| **Как подтверждается завершение** | consumer cutover, финансовая сверка, target SLO, security/DR test, business/Finance sign-off, закрытое legacy write, обновлённые CMDB/DataHub/runbooks и утверждённый акт вывода                                                                                                                                   |

## Судьба систем

| система/класс                                 | состояние через 12 месяцев                                       | действие                                                            | условие                                                                 |
| --------------------------------------------- | ---------------------------------------------------------------- | ------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| **FU/Target Core Banking и Core Ledger**      | остаётся/развивается как целевой мастер                          | принимает мигрированные account/ledger scopes по волнам             | trial migration, balance/entry reconciliation, rollback rehearsal       |
| **Целевая CRM / Customer Case**               | остаётся                                                         | становится единой системой обращений и operational customer context | перенесены открытые кейсы, история доступна, global Party ID обязателен |
| **Целевой Card Processing**                   | остаётся                                                         | принимает выбранные RB card portfolios                              | сертификация процессинга, token/PAN controls, scheme reconciliation     |
| **Целевая Loan Platform**                     | остаётся                                                         | принимает розничные/корпоративные портфели по типам продуктов       | график/долг/ledger tie-out, legal notices, servicing readiness          |
| **Payments Platform**                         | остаётся                                                         | принимает новые payment orders; legacy routes отключаются           | end-to-end idempotency, status mapping, partner certification           |
| **Party MDM / Customer Hub**                  | новый постоянный компонент                                       | мастер global Party ID, golden attributes и crosswalk               | match quality, steward queue, merge/split audit, HA                     |
| **Product Catalog / Reference MDM**           | новый постоянный компонент                                       | мастер опубликованных продуктов и канонических справочников         | versioned mappings и consumer adoption                                  |
| **API/Integration Layer + Kafka**             | новый/расширенный постоянный слой                                | заменяет новые point-to-point, разделяет sync/async                 | contracts, schema registry, DLQ/replay, SLO                             |
| **Enterprise DWH + Semantic Layer**           | остаётся как единый аналитический контур                         | сертифицированная управленческая/регуляторная отчётность            | dual-run reports, Finance sign-off, lineage                             |
| **Data Lakehouse (Object Storage + Iceberg)** | новый постоянный контур                                          | полная история, события, архивы и curated products                  | security zones, maintenance, replay tests                               |
| **Trino**                                     | остаётся как SQL engine Lakehouse                                | advanced analytics; без production federation к OLTP                | resource groups, access policies, cost limits                           |
| **Airflow + dbt/SQL/Spark**                   | остаётся                                                         | batch/EOD/backfill orchestration and transformations                | HA scheduler, code promotion, lineage, runbooks                         |
| **DataHub**                                   | остаётся                                                         | каталог, glossary, ownership, DQ/SLO и lineage                      | critical datasets покрыты owners/classification                         |
| **Unified BI**                                | остаётся                                                         | читает semantic layer/DWH                                           | RB/FU metrics reconciled, user migration complete                       |
| **RB DWH и RB BI**                            | выводятся                                                        | read-only evidence window -> archive -> shutdown                      | все отчёты/consumers переключены, период закрыт, restore test пройден   |
| **RB CRM**                                    | выводится как Party master; затем система — по готовности кейсов | сначала read-only customer source/archive, затем shutdown           | открытые кейсы/каналы перенесены, retention обеспечен                   |
| **RB Core/Cards/Loans**                       | выводятся помодульно, не одним switch                            | write-stop после каждой продуктовой волны, затем read-only/archive  | соответствующий доменный gate; без него — остаточный режим с exit date  |
| **RB point-to-point**                         | выводятся                                                        | strangler adapters -> shared Integration -> shutdown                  | consumer inventory показывает нулевой трафик и заменяющий flow          |

## Состав мигрируемых данных

| класс                           | в target OLTP                                                 | в DWH                                  | в Lakehouse/Object Archive                          | правило                                                  |
| ------------------------------- | ------------------------------------------------------------- | -------------------------------------- | --------------------------------------------------- | -------------------------------------------------------- |
| Party/ID/relationships          | golden operational subset через MDM                           | SCD2 Party/bridges                     | исходные версии и match audit в restricted zone     | не терять source ID; ambiguous match не объединять       |
| Products/references             | активные версии, нужные target engines                        | conformed dimensions                   | все исторические snapshots/mappings                 | effective dates и version обязательны                    |
| Agreements/accounts/cards/loans | активные и необходимые для обслуживания; открытые obligations | полная аналитическая история           | source snapshots/events и архив                     | operational scope определяется продуктовой волной        |
| Payments/transactions           | незавершённые и operationally required history                | reconciled fact history                | полная event-level история                          | completed history не обязательно загружать в target OLTP |
| Accounting entries/balances     | открытые периоды, current state и required ledger history     | сертифицированные entries/EOD balances | immutable extracts/manifests                        | debit=credit и balance roll-forward без tolerance        |
| Customer cases                  | открытые кейсы и необходимая service history                  | SLA/quality facts                      | полный текст/аудио по retention                     | чувствительный текст проходит DLP/redaction              |
| Documents                       | metadata + link в ECM                                         | только разрешённые metadata            | encrypted objects, versions, signatures, legal hold | hash, owner, retention, no orphan objects                |
| Technical metadata              | нужные operational mappings                                   | audit/control tables                   | pipeline logs по retention                          | DataHub содержит metadata, а не sensitive payload        |

## Переключаемые потоки

### Клиент и справочники

**Было:** FU/RB CRM и локальные справочники обмениваются через ESB/P2P
**Станет:** producers -> Integration/Kafka -> Party/Reference MDM -> golden/versioned events -> authorized consumers

### Операционные команды

**Было:** прямые интеграции каналов с конкретными FU/RB системами
**Станет:** channel -> API Gateway/Integration Service -> мастер домена; после commit -> outbox/CDC -> Kafka

### Аналитика

**Было:** независимые FU/RB batch -> FU/RB DWH -> FU/RB BI
**Станет:** domain events/CDC + EOD controls -> Lakehouse/DWH -> certified semantic layer -> Unified BI/Regulatory

### Metadata

**Было:** нет общего каталога/lineage. 
**Станет:** systems/topics/jobs/tables/reports -> DataHub с owner, classification, SLO, DQ и column lineage

## Критерии завершения перехода

### Данные

- 100% записей migration scope имеют source key, target key и migration status; это покрытие scope, а не обещание `идеального качества`
- Нет необъяснённых orphan records для критичных связей Party–Agreement–Account/Card/Loan
- Все активные продукты и справочники имеют утверждённый mapping/version
- Accounting batches сбалансированы; EOD balances сходятся с источником
- Неоднозначные Party находятся в steward queue и не объединены автоматически

### Потоки и системы

- новые записи выбранного scope создаются только в target master
- CDC backlog равен нулю на cutover watermark, далее lag находится в SLO
- все зарегистрированные consumers используют target endpoint/topic/dataset
- старый endpoint не имеет трафика в наблюдаемый период, кроме утверждённого audit/read-only
- backup/restore и DR runbook target проверены

### Отчётность

- параллельные FU/RB и target отчёты прошли согласованное число закрытий периода
- расхождения объяснены, утверждены и отражены в mapping/rules
- regulatory forms tie-out к Core Ledger и подписаны Finance
- report -> metric -> table -> pipeline -> source lineage доступен в DataHub

### Безопасность и эксплуатация

- IAM/ABAC, masking, encryption, audit и retention прошли security review
- production SLO выдержан в hypercare
- Sev-1/Sev-2 migration defects закрыты
- owners, on-call, runbooks, capacity и budget переданы в BAU
- лицензии/инфраструктура legacy закрыты только после legal hold и archive restore test

## Определение выведена из эксплуатации

Система считается выведенной, если одновременно:

1. закрыта возможность business write
2. отключены scheduler, interfaces и service accounts
3. consumer inventory и network monitoring подтверждают отсутствие использования
4. данные архивированы с checksum, схемой, ключами и retention
5. restore/read procedure протестирована
6. CMDB, DataHub и contracts отмечены retired
7. business owner, IT owner, ИБ, Records/Legal и Finance подписали акт
8. инфраструктура/лицензия остановлена

Read-only legacy без владельца и даты выхода не считается decommissioned
