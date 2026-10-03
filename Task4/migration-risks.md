## Шкала

- **Вероятность:** высокая / средняя / низкая - экспертная оценка до получения статистики rehearsal
- **Влияние:** критическое / высокое / среднее / низкое
- **Критическое влияние:** риск потери/искажения средств, нарушения обязательной отчётности, массового раскрытия данных или остановки ключевого сервиса

Рейтинг пересматривается после discovery, каждой репетиции и product wave. `Низкая вероятность` не отменяет contingency для критического влияния

## Реестр рисков

| ID  | риск                                              | влияние                                                          | вероятность | ранний индикатор                                                 | митигация                                                                                | Contingency / фолбэк                                                                  | владелец                      |
| --- | ------------------------------------------------- | ---------------------------------------------------------------- | ----------- | ---------------------------------------------------------------- | ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- | ----------------------------- |
| R01 | Ложный merge клиентов FU/RB                       | Критическое: смешение ПДн, счетов и коммуникаций                 | Средняя     | высокий split rate, общие телефоны, низкий confidence            | strong identifiers, negative rules, размеченная выборка, steward review, dual control    | остановить auto-merge, выполнить split, переиздать crosswalk и downstream bridges     | Party Data Owner              |
| R02 | Дубли/неразрешённые Party не обработаны к волне   | Высокое: неполный профиль и orphan holdings                      | Высокая     | рост steward queue age, `unresolved` выше gate                   | приоритизация active scope, временный source-scoped ID, capacity stewards                | исключить записи из cutover scope без потери обслуживания; перенести отдельной волной | MDM Product Owner             |
| R03 | Неверный mapping продукта/тарифа/статуса          | Критическое для расчётов; высокое для аналитики                  | Высокая     | `UNMAPPED`, one-to-many, расхождение начислений                  | versioned mapping, Product Owner approval, effective dates, scenario tests               | остановить affected product wave; обслуживать её в legacy                             | Product Data Owner            |
| R04 | Потеря или дублирование CDC при snapshot/catch-up | Критическое: пропущенные/повторные финансовые факты              | Средняя     | gap LSN/offset, duplicate event, count/sum drift                 | durable offsets, overlap + idempotency, source snapshot manifest, replay rehearsal       | вернуть routing на legacy, восстановить offset, переиграть из raw/outbox              | Integration Owner             |
| R05 | Нарушение порядка событий                         | Высокое: неверный статус карты/платежа/договора                  | Средняя     | stale version, недопустимый transition                           | partition by aggregate key, source version, optimistic checks, quarantine                | rebuild projection из ordered source history; legacy остаётся master                  | Domain Technical Owner        |
| R06 | Финансовая сверка не сходится                     | Критическое: неверные остатки/отчётность                         | Средняя     | debit≠credit, EOD balance diff, unexplained amount variance      | dual controls, trial runs, immutable manifests, Finance sign-off                         | No-Go/fallback; target не становится certified/master до нулевого unexplained diff    | Chief Accountant              |
| R07 | Late events меняют закрытый период                | Высокое: невоспроизводимый отчёт                                 | Высокая     | late-arrival spike, backfill after certification                 | bitemporal model, close calendar, adjustment facts, controlled reopen                    | сохранить предыдущую report version; выпустить утверждённую корректировку             | Finance Data Owner            |
| R08 | Извлечение перегружает RB/FU OLTP                 | Критическое: деградация клиентского сервиса                      | Средняя     | CPU/IO/locks растут во время extract                             | CDC/read replica, throttling, off-peak snapshot, performance test                        | немедленно остановить extract; продолжить с последнего watermark в новое окно         | Source System Owner           |
| R09 | Cutover превышает окно                            | Высокое: отчёт/операция недоступны вовремя                       | Средняя     | rehearsal duration близка к окну, растёт delta backlog           | измеренная rehearsal, parallelism, pre-stage, freeze scope, abort checkpoint             | abort до routing; вернуть legacy schedule/endpoint                                    | Cutover Manager               |
| R10 | Fallback технически существует только на бумаге   | Критическое при неудачном cutover                                | Средняя     | restore/route-back не репетировались, неизвестен owner           | production-like rollback rehearsal, evidence, automated config versioning                | No-Go до успешной rehearsal                                                           | Migration Lead                |
| R11 | Dual write расходится                             | Критическое для финансовых masters                               | Средняя     | mismatch target/legacy versions, retry asymmetry                 | избегать dual write; один master + events; короткое окно и reconciliation                | остановить target write, legacy остаётся master, reconcile target copies              | Domain Owner                  |
| R12 | Скрытая point-to-point зависимость остаётся       | Высокое: consumer перестаёт работать после shutdown              | Высокая     | трафик к legacy без catalog flow, неизвестные service accounts   | network/job/report inventory, logs, freeze new P2P, consumer sign-off                    | вернуть legacy endpoint read-only, зарегистрировать и мигрировать consumer            | Enterprise Integration Owner  |
| R13 | Breaking schema change во время миграции          | Высокое: остановка pipelines или silent corruption               | Средняя     | schema registry reject, null spike, unknown enum                 | change freeze, compatibility rules, contract tests, versioned topics                     | pin previous version, route incompatible messages to DLQ, postpone wave               | Producer Owner                |
| R14 | ПДн/PAN попадают в staging, logs или DLQ          | Критическое: утечка и расширение PCI/ПДн-контура                 | Средняя     | DLP hit, full PAN pattern, unencrypted export                    | tokenize at source, field allow-list, encrypted quarantine, log redaction, access review | isolate flow, revoke credentials, preserve evidence, execute incident response        | CISO                          |
| R15 | Retention/legal hold не учтён при decommission    | Высокое: невозможность доказать операцию или незаконное удаление | Средняя     | dataset без retention owner, archive restore не пройден          | Records/Legal review, WORM archive, schema/config/key escrow, restore test               | остановить shutdown; сохранить legacy read-only до закрытия требований                | Records Manager               |
| R16 | Target capacity/SLO недостаточны                  | Высокое: latency, backlog, failed reports                        | Средняя     | saturation, queueing, compaction lag, query p99                  | forecast + peak/backfill tests, autoscale/quotas, workload isolation                     | ограничить ad hoc, scale target, оставить часть scope на legacy                       | Platform Owner                |
| R17 | Недостаток специалистов/перегрузка Data Stewards  | Высокое: задержка waves и плохие решения                         | Высокая     | queue age, overtime, owner approvals delayed                     | scope prioritization, migration factory, training, reserve capacity, automation          | уменьшить wave scope/перенести дату; не снижать quality gates                         | Program Sponsor               |
| R18 | Разные определения KPI обнаружены поздно          | Высокое: недоверие к единой отчётности                           | Высокая     | legacy reports оба «правильные», semantic dispute                | glossary, metric owner, parallel close, versioned methodology/effective date             | временно публиковать обе версии с маркировкой; не сертифицировать спорную метрику     | Business Metric Owner         |
| R19 | DataHub/lineage неполон к аудиту                  | Высокое: нельзя доказать происхождение цифры                     | Средняя     | orphan report/table/job, stale connector                         | automated lineage per run, quality gate, stable dataset IDs                              | сохранить manual evidence pack; блокировать decommission источника                    | Data Governance Lead          |
| R20 | Iceberg/Trino/Airflow эксплуатационно незрелы     | Среднее/высокое: нестабильные pipelines и стоимость              | Средняя     | small files, metadata growth, scheduler backlog, runaway queries | pilot, supported stack, compaction, resource groups, HA/runbooks                         | использовать PostgreSQL DWH/object archive для критичного scope до readiness          | Data Platform Owner           |
| R21 | Frontline не готов к новой CRM/маршрутизации      | Высокое: ошибки обслуживания и жалобы                            | Средняя     | training failures, рост handle time/errors                       | role-based training, sandbox, floor support, phased region rollout                       | route affected region back, увеличить hypercare/support                               | Customer Service Owner        |
| R22 | Vendor/лицензия/поставка задерживает платформу    | Среднее/высокое: сдвиг roadmap                                   | Средняя     | procurement milestone missed                                     | early procurement, open formats, alternative deployment, dependency buffer               | оставить interim DWH/adapters дольше с контролируемым exit plan                       | Program Sponsor / Procurement |

## Риски по этапам

| этап              | главные риски           | обязательный контроль                                                        |
| ----------------- | ----------------------- | ---------------------------------------------------------------------------- |
| Discovery         | R12, R15, R18           | consumer inventory, retention matrix, metric ownership                       |
| MDM/MVP           | R01, R02, R03, R08      | match benchmark, steward capacity, mappings, extract throttling              |
| Platform          | R04, R05, R13, R16, R20 | resilience/performance tests, schema compatibility, runbooks                 |
| Analytics cutover | R06, R07, R09, R18, R19 | parallel close, exact reconciliation, rollback rehearsal, lineage            |
| Operational waves | R03–R11, R14, R21       | scoped freeze, one master, trial migration, security and frontline readiness |
| Decommission      | R12, R15, R19           | traffic observation, archive restore, formal sign-off                        |

## Резервы и contingency

План должен иметь:

- временной резерв между сложными product waves
- compute/storage capacity для initial load, replay и parallel run
- отдельную steward queue capacity
- сохранение legacy licenses/infrastructure до закрытия fallback window
- независимый immutable raw/archive
- коммуникационный и incident-response резерв

Резерв нельзя использовать для пропуска reconciliation или security gate

## Управление risk register

1. Риск имеет owner, due date и evidence митигации
2. Перед каждой волной выполняется reassessment probability/impact
3. Critical residual risk принимает только уполномоченный business/Finance/CISO owner, а не проектная команда
4. Trigger автоматически создаёт incident/change stop, если так указано в runbook
5. Реализовавшийся риск становится issue с отдельным recovery plan; он не закрывается изменением формулировки риска
