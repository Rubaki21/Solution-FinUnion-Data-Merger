
## Обозначения действий

| Код             | Действие                         | Смысл в потоках                                                                     |
| --------------- | -------------------------------- | ----------------------------------------------------------------------------------- |
| `C - Create`    | передача нового факта/записи     | Consumer создаёт локальную проекцию, но не становится мастером                      |
| `R - Read`      | чтение опубликованной версии     | Используется для профиля, аналитики или обогащения                                  |
| `U - Update`    | новая версия существующего факта | Для финансовых данных обычно append/correction, а не физический overwrite           |
| `D - Delete`    | удаление/закрытие                | Только metadata/retention/tombstone; проведённые финансовые факты не удаляются      |
| `E - Enrich`    | обогащение                       | Добавление global ID, канонического кода или атрибута без изменения исходного факта |
| `V - Validate`  | проверка                         | Схема, справочник, уникальность, ссылочная и финансовая целостность                 |
| `A - Aggregate` | агрегация                        | Расчёт витрин, метрик и отчётных форм в DWH                                         |

### Граница диаграммы

- Системы FU/RB продолжают обслуживать клиентов и остаются мастерами доменных финансовых данных
- Integration Layer - процесс доставки/валидации, не хранилище мастер-данных
- MDM Customer Hub - мастер идентичности Party и crosswalk
- DWH - аналитическое хранилище и источник сертифицированной отчётности
- DataHub/каталог получает metadata и lineage, а не полные ПДн/финансовые payload
- Регулятор - внешний потребитель контролируемой выгрузки из DWH

## Каталог основных потоков

| ID | Источник | Приёмник | Передаваемые сущности | Действие | Режим | Зачем нужен поток |
|---|---|---|---|---|---|---|
| `F01` | FU CRM | Integration Layer | Party, локальные ID, договор draft, обращение | C/U, первичная V | Async API + CDC | Передать изменения клиентов и кейсов без аналитических запросов к CRM |
| `F02` | RB CRM | Integration Layer | Party, локальные ID, договор draft, обращение | C/U, первичная V | Async API; CDC или инкремент | Включить RB-клиентов в единый профиль и сохранить происхождение |
| `F03` | FU Core | Integration Layer | договор, счёт, платёж, операция, проводка, остаток | C/R/U | CDC + EOD batch | Доставить финансовые факты и контрольный снимок из FU-мастера |
| `F04` | RB Core | Integration Layer | договор, счёт, платёж, операция, проводка, остаток | C/R/U | CDC/инкремент + EOD batch | Доставить RB-факты через управляемый адаптер вместо новых point-to-point связей |
| `F05` | FU Cards | Integration Layer | карта, статус карты, карточная операция | C/U | Streaming/CDC + reconciliation batch | Обеспечить актуальный профиль и полную карточную историю |
| `F06` | RB Cards | Integration Layer | карта, статус карты, карточная операция | C/U | Streaming либо micro-batch + reconciliation batch | Нормализовать статусы RB и контролировать полноту клиринга |
| `F07` | FU Loans | Integration Layer | кредит, график, задолженность, просрочка, роли Party | C/U | CDC + EOD snapshot | Передать кредитный портфель для профиля, риска и отчётности |
| `F08` | RB Loans | Integration Layer | кредит, график, задолженность, просрочка, роли Party | C/U | Инкремент + EOD snapshot | Консолидировать RB-портфель без смены его мастера за три месяца |
| `F09` | Integration Layer | MDM Customer Hub | кандидаты Party, source ID, контакты, связи, локальные справочники | C/U/E/V/Deduplicate | Async events/API; initial batch | Назначить global Party ID, выполнить match/merge и mapping кодов |
| `F10` | MDM Customer Hub | Integration Layer | golden Party, crosswalk, confidence/status, merge/split, канонические коды | R/E | Async events; sync lookup только для onboarding | Распространить эталонную идентичность и mappings |
| `F11a` | Integration Layer | FU CRM | global Party ID, разрешённые golden attributes, merge/split instruction | U/E | Async command/event | Привязать FU-профиль к глобальной Party без слепого overwrite |
| `F11b` | Integration Layer | RB CRM | global Party ID, разрешённые golden attributes, merge/split instruction | U/E | Async command/event | Привязать RB-профиль к глобальной Party и поддержать совместное обслуживание |
| `F12` | Integration Layer | DWH | все валидированные канонические измерения и факты | C/U/V | Streaming/CDC для инкрементов; batch для снимков | Историзировать данные, отделить OLAP от OLTP и выполнить сверку |
| `F13` | MDM Customer Hub | DWH | Party dimension, crosswalk, продуктовые/справочные mappings | R/E | Async change feed + daily full reconciliation | Связать факты FU/RB единым ключом и каноническими кодами |
| `F14` | DWH | Unified BI | сертифицированные витрины и semantic model | A/R | Scheduled refresh; direct query только к serving layer | Дать единые метрики без доступа BI к операционным системам |
| `F15` | DWH | Регуляторный контур/внешний регулятор | утверждённые формы и контрольные показатели | A/R | Регламентный batch/export | Обеспечить воспроизводимую отчётность из сертифицированного источника |
| `F16` | FU/RB operational systems | DataHub/каталог | схемы, термины, владельцы, классификация, статус dataset | C/U/D metadata | Daily metadata scan + change webhook | Сделать данные обнаружимыми и управлять schema change; payload не копируется |
| `F17` | Integration/MDM/DWH/BI | DataHub/каталог | run metadata, lineage, DQ score, SLA, связи report-to-source | C/U | Event/API after each run + daily reconcile | Получить сквозной lineage и наблюдаемость от отчёта до мастера |
| `F18` | DataHub/каталог | сотрудники и команды | определения, владельцы, lineage, DQ/SLA status | R | Sync UI/API | Позволить выбрать доверенный набор и найти ответственного |

## Два разных режима синхронизации Party

### Регистрация/изменение профиля

1. CRM фиксирует локальную запись и публикует событие с source ID
2. Integration Layer проверяет контракт, токенизирует/защищает чувствительные поля и направляет кандидата в MDM
3. MDM выполняет exact/probabilistic match:
   - однозначное совпадение - возвращает существующий `global_party_id`
   - новый Party - создаёт ID
   - неоднозначность - помещает в steward queue, не объединяя записи
4. Golden record и crosswalk публикуются в CRM и DWH

MDM не должен быть синхронной точкой отказа для уже существующего клиента. Синхронный lookup допускается при onboarding, но при недоступности запрос сохраняется с временным correlation ID и завершается асинхронно; финансовая операция использует ранее подтверждённую связь

### Merge/split

Merge/split - не обычный Update. Событие содержит старые и новые global ID, effective time, решение steward и correlation ID. Consumers применяют mapping идемпотентно, не переписывая исходные финансовые записи: DWH меняет аналитическую привязку через историзируемый bridge

## Поток финансовых данных

1. Профильный OLTP проводит операцию и публикует событие после commit
2. Integration Layer проверяет envelope, schema/version, обязательные ссылки и дубликат `event_id`
3. Валидное событие доставляется в DWH; невалидное - в карантин с сохранением payload и причины
4. Отдельный EOD-файл/снимок содержит manifest: количество, суммы по валютам, начальный/конечный остаток и hash
5. DWH сравнивает event-level загрузку с EOD control totals; период не сертифицируется при необъяснённом расхождении
6. BI/регулятор получает только опубликованный сертифицированный слой

Streaming обеспечивает свежесть, но batch reconciliation обеспечивает доказуемую полноту. Один режим не заменяет другой

## Обработка ошибок

- Недоступный consumer не блокирует commit мастера: сообщение остаётся в durable queue и повторяется с backoff
- Schema incompatibility, неизвестный справочник и нарушенная ссылка направляются в отдельные DLQ-категории
- После исправления данные переигрываются с исходным `event_id`; ручное копирование в целевую таблицу запрещено
- Lag, DLQ depth, success rate и reconciliation difference публикуются в мониторинг и каталог.
- Для каждого Sev-1/Sev-2 потока существует runbook с владельцем, RTO восстановления потока и процедурой backfill
