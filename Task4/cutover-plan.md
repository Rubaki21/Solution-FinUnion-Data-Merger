## Выбранный критичный домен

Переключаются:

- FU DWH + FU BI
- RB DWH + RB BI
- локальные отчётные batch jobs и подписки

на:

- Enterprise DWH
- единый semantic layer
- Unified BI
- Regulatory Reporting из сертифицированной зоны DWH

Это критичный, но контролируемый cutover: операционные системы продолжают принимать платежи и обслуживать клиентов. Аналитический target не является мастером OLTP, поэтому fallback не требует отката клиентских транзакций

## Scope и ограничения

### В scope первой волны

- отчёты по клиентам, продуктам/договорам, счетам/EOD-остаткам и операциям
- утверждённые управленческие KPI
- выбранные регуляторные формы, прошедшие параллельное закрытие
- BI users, schedules, subscriptions и exports
- lineage и certification metadata

### Вне scope

- изменение Core/Cards/Loans masters
- миграция online balance
- отчёты без владельца/определения
- формы, не прошедшие Finance acceptance: они остаются на legacy с исключением и датой отдельного cutover

## Участники

| роль                        | ответственность                                   |
| --------------------------- | ------------------------------------------------- |
| Cutover Manager             | единый timeline, команды, evidence, go/no-go log  |
| DWH Technical Owner         | target load, backup/restore, performance          |
| FU/RB DWH Owners            | финальные выгрузки, legacy availability, fallback |
| Data Owners/Stewards        | mappings, DQ exceptions, business checks          |
| Chief Accountant/Finance    | financial reconciliation и regulatory sign-off    |
| BI Owner                    | users, semantic model, schedules, subscriptions   |
| ИБ/Privacy                  | access, masking, audit и export controls          |
| Service Desk/Communications | уведомления, support и escalation                 |

## Предварительные Go criteria

Не позднее T−5 рабочих дней должны быть подтверждены:

- минимум согласованное число успешных parallel closes
- все отчёты scope имеют owner, formula/version, source lineage и acceptance evidence
- target capacity/performance проверены на peak concurrency
- backup/restore target и legacy протестированы
- service accounts, SSO/RBAC/row-level security и export restrictions проверены
- final-load и rollback runbooks отрепетированы
- legacy pipelines могут продолжать работу в fallback window
- нет открытых Sev-1/Sev-2 и критичных `UNMAPPED`
- freeze/communication/change tickets утверждены

## План по времени

| время          | шаг                | действие                                                                                                   | Evidence / результат                       | Abort criterion                                  |
| -------------- | ------------------ | ---------------------------------------------------------------------------------------------------------- | ------------------------------------------ | ------------------------------------------------ |
| T−30…T−10 дней | Подготовка         | report inventory, owner acceptance, parallel run, user training, support plan                              | подписанная report matrix и comparison log | критичный отчёт без owner/lineage                |
| T−10…T−5       | Репетиция          | production-like final load, restore и fallback rehearsal                                                   | measured duration и обновлённый runbook    | шаг не помещается в окно, restore не работает    |
| T−5…T−1        | Change freeze      | запрет releases ETL/semantic/report layouts/mappings; emergency change только через board                  | зафиксированы Git/config/schema versions   | неутверждённое изменение                         |
| T−1            | Резервирование     | snapshots/PITR markers DWH, export configs, BI metadata, IAM, schedules                                    | checksum и restore point IDs               | backup/restore validation failed                 |
| T0, после EOD  | Финальная выгрузка | source deltas до согласованного watermark; EOD files/manifests из Core/Cards/Loans; MDM/reference snapshot | batch IDs, offsets, count/sum/hash         | неполный source close или неизвестный watermark  |
| T0+            | Загрузка           | landing/staging → canonical → DWH core; dimensions/facts/SCD; quarantine фиксируется                       | target batch complete, no hidden rejects   | pipeline failure, critical quarantine            |
| T0+            | Сверка             | техническая, бизнес- и финансовая reconciliation; report comparison                                        | подписанный reconciliation pack            | любое необъяснённое финансовое расхождение       |
| T0+            | Go/No-Go           | owners рассматривают evidence                                                                              | timestamped Go decision                    | обязательный owner не дал Go                     |
| T0+            | Переключение       | activate target datasource, BI routes, schedules/subscriptions; old BI read-only banner                    | target routes and smoke tests              | authentication, critical query или export failed |
| T+0…T+3 дня    | Hypercare          | усиленный monitoring, floor support, daily Finance/DQ sign-off                                             | incident/DQ log, SLO dashboard             | Sev-1 или повторяющееся критичное расхождение    |
| T+3…T+14       | Стабилизация       | target primary, legacy warm/read-only, close lower defects                                                 | стабильные closes и users                  | approved fallback trigger                        |
| После T+14     | Закрытие fallback  | stop legacy loads/schedules, archive evidence, revoke normal access                                        | decommission ticket phase 1                | незакрытые critical dependencies                 |

Конкретные часы и продолжительность определяются после rehearsal; календарная дата не должна подменять измеренный runtime

## 1. Подготовка

- создать полную матрицу `legacy report → target report/metric → owner → consumers`
- исключить неиспользуемые отчёты только после подтверждения consumer inventory
- зафиксировать semantic definitions: grain, filters, status, event/posting/business date, currency rule
- перенести users/groups, row/column security и subscriptions
- выполнить два независимых восстановления: target database и legacy report/config backup
- подготовить communication templates для Go, delay, fallback и recovery
- открыть bridge/channel, назначить scribe и запретить устные незафиксированные решения

## 2. Data freeze

Замораживаются:

- production releases ETL/dbt/Airflow для scope
- MDM/product/reference mappings, влияющие на отчёт
- semantic model, metric formulas, report layouts и schedules
- изменения IAM/groups, кроме аварийных
- закрытие/пересоздание source datasets

Не замораживаются:

- платежи, карты, кредиты и другие клиентские операции
- source OLTP writes
- обязательные fraud/security actions

Изменения OLTP после snapshot захватываются CDC/final delta. Emergency change документируется, повторно проходит impact analysis и может перенести cutover

## 3. Финальная выгрузка

Для каждого источника фиксируются:

- source system и business date
- snapshot ID/SCN/LSN/watermark
- минимальный/максимальный source key/time
- row count
- суммы debit/credit/amount по валюте и типу
- file size/hash и schema version
- extract start/end и responsible person

Файл без manifest не принимается. CDC поток закрывает интервал между initial snapshot и cutover watermark; overlap дедуплицируется по source key/version

## 4. Загрузка

1. Проверить checksum/schema и загрузить immutable landing
2. Bulk-load во временную staging partition
3. Типизировать и применить versioned mappings
4. Применить MDM crosswalk; unresolved сохранить отдельно
5. Загрузить SCD dimensions и факты идемпотентно
6. Выполнить DQ gates
7. Опубликовать новые partitions/views атомарно, но не открывать consumers до Go

Ручной `UPDATE` production mart для `быстрого исправления` запрещён. Исправление выполняется в source/mapping/pipeline и переигрывается

## 5. Сверка

| уровень            | проверка                                            | критерий Go                                                               |
| ------------------ | --------------------------------------------------- | ------------------------------------------------------------------------- |
| Транспорт          | files/events/batches, checksum, sequence, watermark | все ожидаемые объекты получены; нет gap/неучтённых rejects                |
| Количественный     | row counts по source/entity/date/status             | равенство после документированных exclusions                              |
| Финансовый         | debit=credit, обороты, комиссии, суммы по валюте    | точное равенство для бухгалтерских controls                               |
| Остатки            | EOD ledger/available/loan balances                  | точное source-to-target равенство по утверждённому grain                  |
| Ссылочный          | orphan facts, Party/Product/Account mappings        | нет критичных orphan/unmapped; некритичные исключения утверждены          |
| Отчётный           | line/metric-level legacy vs target                  | нет необъяснённых различий; approved methodological differences versioned |
| Безопасность       | users/roles/row-level results/export                | positive и negative access tests пройдены                                 |
| Производительность | critical dashboards/exports/concurrency             | измеренный SLO достигнут с согласованным запасом                          |

Для нефинансовой метрики допустимый порог задаёт Data Owner до cutover. Порог нельзя назначать после обнаружения расхождения

## 6. Переключение

Порядок:

1. отключить legacy report schedules/subscriptions, сохранив возможность ручного fallback
2. переключить BI connection/alias на target semantic layer
3. активировать target schedules и distribution lists
4. переключить Regulatory Reporting service на certified target dataset
5. очистить разрешённые BI caches
6. выполнить smoke tests под ролями Finance, manager, regional user и auditor
7. опубликовать Go message и ссылку на support channel
8. пометить legacy BI read-only и явно указать дату последней сертификации

DNS/alias/config меняется через версионируемую конфигурацию. Изменение не затрагивает OLTP routes

## 7. Фолбэк

### Триггеры

- необъяснённое финансовое расхождение
- отсутствует/искажён критичный отчёт или regulatory form
- массовый отказ авторизации либо утечка доступа
- target не выдерживает SLO и не восстанавливается в cutover window
- потерян lineage/evidence, необходимый для подтверждения формы
- Sev-1 с риском принятия неверного управленческого решения

### Процедура

1. Cutover Manager объявляет fallback и фиксирует timestamp/причину
2. Target BI schedules и exports останавливаются; пользователи получают banner
3. BI alias/subscriptions возвращаются на последнюю сертифицированную legacy-версию
4. Legacy loads догоняются до утверждённого watermark из независимых source extracts
5. Выполняется сокращённая финансовая сверка legacy
6. Target snapshot и logs сохраняются для расследования; данные не удаляются вручную
7. Users получают сообщение о восстановленной версии и её `as_of_time`
8. Новый cutover назначается только после RCA, исправления и полной rehearsal

Поскольку target DWH не является операционным мастером, откатывать Core-транзакции не требуется. Если regulatory submission уже отправлен, используется формальная процедура корректирующей формы; простое переключение BI не отменяет внешнюю подачу

## 8. Мониторинг после переключения

| категория     | метрики                                                                          |
| ------------- | -------------------------------------------------------------------------------- |
| Pipeline      | success rate, duration, lag, watermark, retries, DLQ/quarantine age              |
| Data quality  | completeness, duplicates, orphan/unmapped, SCD overlap, freshness                |
| Finance       | debit-credit difference, source-target amount difference, balance roll-forward   |
| BI            | query p95/p99, errors, cache freshness, report completion, subscription delivery |
| Security      | denied/privileged access, unusual export volume, DLP alerts, masking failures    |
| Adoption      | active users, legacy access attempts, target/legacy report usage                 |
| Support       | incident count/severity, user complaints, time to resolution                     |
| Cost/capacity | CPU/storage/concurrency, queueing, forecast headroom                             |

Hypercare review проводится минимум ежедневно; Finance sign-off — на каждом закрытии периода в fallback window

## Post-cutover

- закрыть incident/problem records и документировать accepted differences
- обновить DataHub certification, lineage и owners
- сохранить reconciliation pack, Go decision, manifests и audit log
- провести lessons learned перед следующей доменной волной
- отключить legacy только по decommission checklist, а не автоматически по истечении T+14
