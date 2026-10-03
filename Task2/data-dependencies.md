## Критические цепочки зависимостей

### 1. Единый профиль

`CRM source ID -> Integration -> MDM match/crosswalk -> global_party_id -> DWH bridge -> Customer 360/BI`

Если crosswalk не готов или ошибочен, факты нельзя безопасно объединять. При этом исходные записи не блокируются: они попадают в DWH с source key и статусом `unresolved`, а после решения MDM привязываются историзируемым bridge без перезаписи финансового факта

### 2. Финансовая отчётность

`Core/Cards/Loans facts -> canonical mapping -> DWH reconciliation -> certified period -> BI/regulatory export`

Отчёт не публикуется как сертифицированный, пока не пройдены count/sum/hash, debit-credit и balance roll-forward. Своевременность не имеет приоритета над необъяснённым финансовым расхождением

### 3. Продуктовая аналитика

`Local product code -> versioned FU/RB mapping -> global_product_id -> Agreement -> Account/Card/Loan facts`

Неполный mapping не должен молча попадать в категорию `прочее`: используется `UNMAPPED` с владельцем, причиной и сроком исправления

### 4. Сквозной lineage

`source record/event -> flow run -> DWH table/model -> semantic metric -> report/export`

Lineage зависит от стабильных dataset/job/report ID и correlation/run ID. Каталог не должен пытаться восстановить происхождение только по совпадению имён таблиц

## Реестр проблемных зависимостей

|   № | проблема / зависимость                                             | последствие                                                                   | ранний индикатор                                                               | мера управления                                                                                                       | владелец                                 |
| --: | ------------------------------------------------------------------ | ----------------------------------------------------------------------------- | ------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------- | ---------------------------------------- |
|   1 | Разные и переиспользуемые FU/RB идентификаторы клиента             | Неверный 360°-профиль, раскрытие продуктов другому лицу, двойной учёт клиента | Один source ID связан с несколькими global ID; рост merge/split                | MDM crosswalk, составной ключ `(source_system, source_id)`, confidence, steward review, аудит merge/split             | Data Owner Party                         |
|   2 | Ложноположительный auto-merge Party по ФИО/телефону                | Наиболее опасная ошибка: смешение ПДн, счетов и обязательств разных лиц       | Рост жалоб, общие телефоны, низкий confidence, частые split                    | Запрет merge по одному слабому атрибуту, пороги по типу Party, negative rules, dual control для high-risk merge       | MDM Product Owner / Party Steward        |
|   3 | Неполный или неоднозначный mapping продуктов, тарифов и статусов   | Несопоставимая отчётность, неверные комиссии/сегменты, невозможность миграции | `UNMAPPED` > порога, one-to-many без правила, активный договор без версии      | Versioned Product/Reference Hub, effective dates, owner approval, блокировка сертификации при критичном mapping gap   | Product Data Owner                       |
|   4 | Разные определения операции, даты и статуса в FU/RB                | Двойной счёт оборота, расхождение KPI и регуляторных форм                     | Одна операция попадает в несколько фактов; отличие event/posting/business date | Канонический словарь BIAN, отдельные Payment/Transaction/Entry, semantic layer и тесты гранулярности                  | Payment Owner + Chief Accountant         |
|   5 | Потеря, повтор или нарушение порядка CDC-событий                   | Неверный статус, дубли платежей и неполная история                            | Gap sequence/LSN, duplicate event rate, отрицательный lag, stale version       | Durable log, outbox, idempotency, partition by business key, sequence/version check, periodic full reconciliation     | Integration Technical Owner              |
|   6 | Streaming не совпадает с EOD-снимком/главной книгой                | BI показывает оперативные данные, которые не подтверждаются учётом            | Count/sum/hash diff, debit ≠ credit, balance roll-forward diff                 | Двойной контур: события + контрольный EOD; quarantine; certified flag только после сверки                             | Accounting Data Owner                    |
|   7 | Несогласованные часовые пояса, business date и закрытие дня        | Операции попадают не в тот период, нарушается SLA отчёта                      | Скачок late events у полуночи, разные EOD cut-off                              | UTC event time + локальный offset + business date от мастера; единый календарь и cut-off table                        | Accounting Steward                       |
|   8 | Аналитическая выгрузка нагружает production OLTP                   | Рост latency/отказы клиентских операций                                       | CPU/IO Core растёт во время ETL, lock waits, batch overruns                    | Log-based CDC/read replica, rate limits, extract windows, запрет ad hoc BI к OLTP                                     | Core Technical Owner                     |
|   9 | Скрытые RB point-to-point интеграции обходят Integration Layer     | Неконтролируемые копии, конфликтующие обновления, неполный lineage            | Изменение в RB есть у потребителя, но нет flow ID/catalog entry                | Инвентаризация сетевых/планировочных потоков, freeze новых P2P, strangler adapters, decommission plan                 | Enterprise Architect / Integration Owner |
|  10 | Integration Layer или MDM становится синхронной точкой отказа      | Невозможность onboarding/обслуживания при сбое общей платформы                | Рост sync timeout, очереди команд, fallback use                                | Async-first, локальный cache подтверждённых ID, circuit breaker, durable queue; sync lookup только там, где необходим | Platform Owner                           |
|  11 | Изменение схемы producer ломает consumers                          | Остановка MDM/DWH или молчаливое смещение полей                               | Schema reject, null spike, unknown enum                                        | Schema registry, compatibility rules, consumer contract tests, deprecation window, canary                             | Producer Technical Owner                 |
|  12 | Поздние события и backfill меняют закрытый отчётный период         | Невоспроизводимая отчётность и ручные корректировки                           | Late-arrival rate, новые записи после certification timestamp                  | Bitemporal history, controlled reopen, adjustment batch, versioned report snapshot и approval                         | Chief Accountant / DWH Owner             |
|  13 | Обратная синхронизация создаёт цикл CRM<->MDM                      | Бесконечные обновления, timestamp churn, потеря приоритета источника          | Один correlation ID многократно возвращается; update storm                     | Attribute ownership, event origin, expected version, echo suppression, запрет blind overwrite                         | Party Data Owner                         |
|  14 | Источник не может выдать надёжный watermark или delete/tombstone   | Пропуски, повторная полная загрузка, `воскресшие` записи                      | Row count дрейфует; записи закрыты в источнике, но активны в DWH               | Change table/outbox; составной high-water mark; overlap window; periodic snapshot diff                                | Source System Owner                      |
|  15 | ПДн/PAN попадают в события, логи, DLQ или каталог                  | Массовая утечка и расширение защищаемого контура                              | DLP hit, full PAN pattern, sensitive sample in catalog                         | Data minimization, tokenization at source, log redaction, encrypted quarantine, no samples by default, retention      | CISO + Data Owner                        |
|  16 | Privacy deletion конфликтует с банковскими сроками хранения        | Незаконное удаление учётных данных либо незаконное сохранение лишних ПДн      | Unresolved deletion requests, разные retention rules                           | Legal basis matrix, legal hold, field-level anonymization, central deletion workflow и downstream acknowledgement     | Privacy Officer / Records Manager        |
|  17 | Каталог/lineage отстаёт от реальных потоков                        | Нельзя доказать происхождение отчёта и найти владельца инцидента              | Orphan dataset/job, stale scan, report without source                          | Stable IDs, автоматическая публикация lineage per run, quality gate для production flow                               | Data Governance Lead                     |
|  18 | Технически успешная загрузка содержит семантически неверные данные | `Зелёный` pipeline создаёт неверный отчёт                                     | Резкий drift метрик при нулевых transport errors                               | Business DQ checks, thresholds by segment/currency, reconciliation and owner sign-off                                 | Relevant Data Owner                      |

## Приоритет рисков на первые три месяца

### `P0`: до совместной отчётности

- ошибочный merge/split Party
- потеря/дублирование финансовых событий
- дисбаланс проводок и остатков
- утечка ПДн/карточных данных
- отсутствие воспроизводимого source-to-report lineage

### `P1`: до масштабирования обмена

- неполный mapping продуктов и справочников
- schema breaking changes
- нагрузка извлечения на OLTP
- синхронная зависимость от MDM/Integration
- поздние события в закрытом периоде

### `P2`: управлять планом 4–12 месяцев

- скрытые point-to-point связи
- legacy-источники без CDC/tombstone
- неполная автоматизация каталога
- технический долг временных batch-адаптеров

## Quality gates

### Поток допускается в production, если

- назначены Data Owner, Data Steward и Technical Owner
- контракт зарегистрирован и прошёл compatibility/security review
- определены source key, global key, time semantics и idempotency
- есть DQ-порог, DLQ, replay и reconciliation
- мониторинг связан с runbook и дежурной командой
- lineage публикуется в DataHub

### Набор допускается в certified DWH/BI, если

- закрыт критический `UNMAPPED`
- пройдены count/sum/hash и финансовые сверки
- нет необъяснённых orphan facts
- указан business date, freshness и версия mapping
- Data Owner подписал результат либо действует заранее утверждённый автоматический gate

## Остаточный риск промежуточного состояния

За три месяца сохраняются два набора OLTP-мастеров и часть batch-интеграций. Это осознанный риск, а не целевая архитектура. Он принимается только при наличии:

- явного `servicing_system` для каждой финансовой записи
- канонического ID/mapping и запрета конкурирующей записи
- ежедневной полной финансовой сверки
- наблюдаемого адаптера вместо нового point-to-point
- даты и владельца вывода каждого временного потока из эксплуатации
