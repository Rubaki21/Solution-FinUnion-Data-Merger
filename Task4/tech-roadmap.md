## Зоны

Используется адаптированный Technology Radar:

- `Adopt`: стандарт для production, поддерживается платформенной командой
- `Trial`: ограниченный production pilot с exit criteria; не применять повсеместно
- `Assess`: PoC/исследование без критичной production-зависимости
- `Hold`: не создавать новые решения; существующие должны иметь migration/retirement plan
- `Retire`: вывести после переключения consumers и выполнения archive gate

## Adopt

| технология/подход                                   | почему в зоне                                                          | применение и срок                                               |
| --------------------------------------------------- | ---------------------------------------------------------------------- | --------------------------------------------------------------- |
| `Party MDM / Customer Hub`                          | без global Party ID невозможно безопасно объединить профили            | MVP к М3; промышленный match/steward/HA к М6                    |
| `Product Catalog / Reference MDM`                   | устраняет конфликт кодов, тарифов и справочников                       | mappings к М3; target master к М6–М8                            |
| `API Gateway + Integration Services`                | управляемые синхронные команды, security/rate limit и strangler legacy | стандарт новых sync-интеграций с М2                             |
| `Kafka-compatible event platform + Schema Registry` | развязывает systems, поддерживает replay и versioned event contracts   | foundation М3–М5; новые доменные события с М5                   |
| `Transactional outbox / log-based CDC`              | публикация после commit без распределённой транзакции                  | стандарт для target OLTP; adapters для legacy по возможности    |
| `PostgreSQL DWH как interim`                        | быстрый результат к М3 на знакомой SQL-платформе                       | отдельный кластер М2–М3; не использовать как raw event archive  |
| `Enterprise DWH + semantic layer`                   | стабильные conformed models и сертифицированная отчётность             | М4–М7; primary reporting после cutover                          |
| `Airflow для batch/EOD orchestration`               | зависимости, retry, backfill, scheduling и run metadata                | с М2; HA/CI/CD к М5                                             |
| `dbt/SQL для DWH transformations`                   | version control, tests, documentation и понятные set-based models      | с MVP; общие packages/standards к М5                            |
| `Object Storage`                                    | архив, immutable raw, документы и дешёвая история                      | landing/archive с М2; zones/security к М4                       |
| `DataHub`                                           | ownership, glossary, classification, DQ/SLO и lineage                  | critical assets к М3; source-to-report coverage к М8            |
| `OpenLineage-совместимая публикация`                | связывает Airflow/jobs/datasets по стабильным run IDs                  | стандарт pipeline metadata с М4                                 |
| `IAM/KMS/Secrets/DLP/Audit`                         | обязательная защита ПДн, банковской тайны и карточных данных           | foundation до production data; continuous                       |
| `DQ + financial reconciliation as code`             | transport success не доказывает корректность банковских данных         | count/sum/hash, debit=credit и balance checks с первой загрузки |
| `IaC + CI/CD + contract tests`                      | воспроизводимость среды и управляемые изменения                        | foundation М1–М3; quality gate всех релизов                     |

### Почему PostgreSQL DWH - Adopt, но не конечный ответ для всего

Он является production-стандартом промежуточного scope и может остаться частью целевой платформы при подтверждённой производительности. Полная raw/event history переносится в Lakehouse; при нехватке ресурса сертифицированный DWH физически мигрирует на MPP без изменения логической модели

## Trial

| технология/подход                         | почему в зоне                                                                     | Exit criteria и движение                                                                                           |
| ----------------------------------------- | --------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `Apache Iceberg`                          | нужен для snapshots/time travel/schema evolution, но требует maintenance maturity | pilot карточной истории М3–М5; Adopt к М6 при успешных concurrent write/read, compaction, restore и security tests |
| `Trino`                                   | удобен для SQL по Iceberg, но требует workload/cost control                       | pilot М4–М6; Adopt для curated analytics к М7 после resource groups, ABAC и performance tests                      |
| `Flink/Kafka Streams`                     | нужен event-time/stateful processing, Airflow его не заменяет                     | пилот карточных/канальных событий; Adopt выбранного runtime после replay/checkpoint/upgrade tests                  |
| `Доменные data products / mesh practices` | ownership полезен, но полная федерализация преждевременна                         | 2 продукта М5–М7, 5 к М10; расширять при SLO/DQ/on-call maturity                                                   |
| `Policy as code для data access`          | снижает расхождение политик DWH/Lakehouse                                         | пилот identity/PCI zones; Adopt после negative access и audit tests                                                |
| `Автоматический column lineage`           | ручной lineage не масштабируется, но connectors могут быть неполны                | сверить с critical reports; Adopt только при доказанном coverage                                                   |

Trial - это production scope с ограниченным blast radius. Он имеет owner, support, security review и fallback; это не бесконтрольный эксперимент

## Assess

| технология/подход                                | причина оценки                                         | ограничение                                                                    |
| ------------------------------------------------ | ------------------------------------------------------ | ------------------------------------------------------------------------------ |
| `Графовая БД для Party/fraud relationships`      | полезна для ownership chains/fraud rings               | не мастер, не хранить лишние ПДн; PoC после стабилизации MDM                   |
| `Feature Store`                                  | может повысить воспроизводимость antifraud/ML features | нужен только при нескольких production ML use cases                            |
| `Data virtualization/federation`                 | быстро объединяет источники для discovery              | не использовать как обход миграции или live join к Core                        |
| `Change Data Capture для каждого legacy source`  | log-based CDC предпочтителен                           | оценить поддержку, нагрузку и лицензии; где нельзя — managed incremental batch |
| `Automated data observability/anomaly detection` | дополняет deterministic DQ                             | не заменяет финансовые controls и owner thresholds                             |
| `Confidential computing/advanced PETs`           | потенциально полезны для совместной аналитики          | нет необходимости включать в 12-месячный critical path без сценария            |

## Hold

| технология/подход                                       | почему Hold                                                  | замена                                              |
| ------------------------------------------------------- | ------------------------------------------------------------ | --------------------------------------------------- |
| `Новые point-to-point интеграции`                       | умножают связи, скрывают ownership и lineage                 | API Layer или Kafka contract                        |
| `Прямые BI-запросы к production OLTP`                   | создают нагрузку и обходят сертифицированные метрики         | DWH semantic layer                                  |
| `Ночные full extracts без watermark/manifest`           | пропуски/дубли, большая нагрузка, нет доказательства полноты | CDC или incremental + periodic reconcile            |
| `Распределённая dual write FU↔RB`                       | трудно обеспечить атомарность и rollback                     | один master + outbox/events/saga                    |
| `MDM как мастер транзакций/остатков`                    | нарушает доменные инварианты                                 | Core/Cards/Loans/Payments/GL                        |
| `Kafka как долгосрочное единственное хранилище`         | retention/replay не заменяют архив и аналитическую историю   | Iceberg/Object Storage                              |
| `Airflow для обработки каждого streaming event`         | scheduler не является stream runtime                         | Flink/Kafka Streams; Airflow — batch/control        |
| `Ручные Excel mappings и корректировки production mart` | нет версий, контроля и lineage                               | Reference MDM + versioned pipeline                  |
| `Новые proprietary raw formats без exit path`           | vendor lock-in и несовместимость engines                     | open file/table formats                             |
| `Full Data Mesh reorganization в первый год`            | нет общей семантики/платформенной зрелости                   | ограниченные domain products + federated governance |

## Retire

| компонент                                 | план                                                      | Gate                                             |
| ----------------------------------------- | --------------------------------------------------------- | ------------------------------------------------ |
| RB DWH / RB BI                            | read-only после reporting cutover, затем archive/shutdown | consumers switched, parallel close, restore test |
| Дублирующие FU/RB report jobs             | остановить после target schedule stability                | lineage и subscriptions перенесены               |
| RB point-to-point                         | strangler adapters, traffic observation, shutdown         | нулевой неутверждённый трафик                    |
| Локальные client/product crosswalks       | перенести в MDM и запретить изменение                     | mapping completeness и consumer adoption         |
| Мигрированные RB Core/Cards/Loans modules | write-stop -> read-only evidence -> archive/shutdown        | domain financial/business/legal gate             |

## Поставка по периодам

| период    | технологический результат                                                                                              |
| --------- | ---------------------------------------------------------------------------------------------------------------------- |
| `М1–М3`   | IaC/CI/CD; security baseline; MDM MVP; Integration adapters; PostgreSQL DWH; Airflow/dbt; DataHub MVP; object archive  |
| `М3–М5`   | Kafka/Schema Registry; CDC/outbox; Iceberg/Trino/streaming pilots; centralized observability; replay/DR tests          |
| `М5–М7`   | Iceberg и Trino переходят в Adopt после gates; Enterprise DWH/semantic layer; Product/Reference MDM; reporting cutover |
| `М6–М9`   | domain data products; policy/lineage automation; operational pilots; workload/cost tuning                              |
| `М8–М11`  | масштабирование migration waves, target masters, decommission adapters; capacity expansion                             |
| `М10–М12` | legacy shutdown/archive, BAU handover, license closure, radar review и roadmap следующего года                         |

## Критерии перевода Trial -> Adopt

- назначены Product/Technical Owner и on-call
- пройдены security, performance, resilience, upgrade и restore tests
- есть IaC, CI/CD, monitoring, cost limits и runbooks
- минимум один production use case выдержал SLO в согласованный период
- fallback проверен
- знания распределены между несколькими сотрудниками
- DataHub содержит contract, owner, classification и lineage

## Принцип ограничения стека

Для одной функции выбирается один основной поддерживаемый инструмент. Spark и stream runtime используются только там, где SQL/dbt недостаточно; Trino не дублирует BI semantic layer; DataHub не превращается в хранилище payload. Это ограничивает технологический риск во время слияния
