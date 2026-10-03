## Область и выбор схемы

Выбрана **схема `звезда` для анализа финансовых операций клиента**. Она подходит для частых OLAP-запросов:

- оборот и количество операций по времени, клиентскому сегменту, продукту и каналу
- комиссии и конверсия
- активность клиента и продукта
- карточные/платёжные операции по типу и контрагенту
- сравнение FinUnion/RetailBank во время миграции

Звезда денормализует бизнес-атрибуты в измерениях, уменьшает число joins и даёт стабильную semantic layer для BI. Нормализованные source-модели остаются в Standardized Lakehouse/DWH integration layer

## Гранулярность факта

`fact_transaction` содержит **одну каноническую ногу завершённой/учтённой бизнес-операции по одному клиентскому счёту в одной валюте**

Это определение обязательно:

- перевод между двумя клиентскими счетами создаёт две ноги с общим `enterprise_transaction_id`
- комиссия хранится отдельной ногой либо явно заданной мерой согласно контракту
- авторизация без клиринга остаётся в Lakehouse operational view и попадает в факт только с определённым аналитическим статусом
- сторно - новая строка с `is_reversal=true` и ссылкой на исходную операцию
- технические версии одного события не создают новые строки после дедупликации
- `transaction_count=1` используется только после применения scope-фильтра

Без явной гранулярности суммирование внутренних переводов даст двойной оборот. `dim_transaction_type.financial_scope` задаёт правила: customer debit/credit, internal leg, fee, reversal, non-financial

## Fact table

| таблица            | Grain                                    | ключевые меры                                                                                    | Degenerate/business IDs                                                  |
| ------------------ | ---------------------------------------- | ------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------ |
| `fact_transaction` | одна transaction leg на клиентском счёте | `original_amount`, `account_amount`, `base_amount`, `fee_amount`, `fx_rate`, `transaction_count` | `enterprise_transaction_id`, source transaction ID, reversal/original ID |

### Аддитивность

| мера                | аддитивность                                             | ограничение                                           |
| ------------------- | -------------------------------------------------------- | ----------------------------------------------------- |
| `account_amount`    | аддитивна в одной валюте счёта и после direction/scope   | нельзя складывать разные валюты                       |
| `base_amount`       | аддитивна по утверждённому base-currency rate/version    | хранится версия курса/правила                         |
| `original_amount`   | неаддитивна между валютами                               | анализируется с `dim_currency`                        |
| `fee_amount`        | аддитивна, если комиссия не представлена отдельной ногой | контракт исключает двойной учёт                       |
| `transaction_count` | аддитивна после dedup и scope filter                     | внутренние ноги могут завысить число бизнес-операций  |
| `fx_rate`           | неаддитивна                                              | используются weighted metrics или конкретная операция |

Distinct business operations считаются по `enterprise_transaction_id`, а не по строкам факта

## Dimensions

| измерение              | тип                            | основные атрибуты                                                         | источник                          |
| ---------------------- | ------------------------------ | ------------------------------------------------------------------------- | --------------------------------- |
| `dim_party`            | SCD2                           | global Party ID, тип, сегмент, резидентность, KYC band без избыточных ПДн | Party MDM                         |
| `dim_account`          | SCD2                           | global/source account ID, тип, валюта, статус, servicing system           | Core Banking                      |
| `dim_agreement`        | SCD2                           | тип и статус договора, дата/версия условий                                | Customer Agreement/domain OLTP    |
| `dim_product`          | SCD2                           | global product ID, family, type, tariff version                           | Product Catalog                   |
| `dim_counterparty`     | SCD2/limited                   | токен/глобальный ID, тип, страна, отрасль                                 | MDM/payment data                  |
| `dim_date`             | статическое                    | календарная и banking business date, месяц, квартал, рабочий день         | Reference MDM/calendar            |
| `dim_time`             | статическое                    | час, минута, часть дня                                                    | DWH                               |
| `dim_channel`          | SCD1/версионируемый справочник | канал, группа, assisted/self-service                                      | Reference MDM                     |
| `dim_transaction_type` | версионируемый справочник      | canonical type, group, financial scope                                    | Reference MDM/Finance             |
| `dim_currency`         | версионируемый справочник      | ISO code, minor units                                                     | внешний authority -> Reference MDM |
| `dim_merchant`         | SCD2/limited                   | merchant token, MCC, страна                                               | Cards/Reference                   |
| `dim_source_system`    | SCD2                           | source, bank origin, serving domain                                       | DataHub/technical reference       |

`dim_date` используется как role-playing dimension для event date и posting date. В физической модели это один справочник с двумя FK

## Ключи и историзация

- Surrogate keys (`*_sk`) изолируют факт от смены natural/source keys
- Global ID сохраняется в dimensions; локальный source key и source system обязательны для lineage
- SCD2-версия выбирается по business/event time, а не по времени загрузки
- Для поздно пришедшего факта выполняется temporal lookup к версии измерения, действовавшей в момент операции
- Merge Party не переписывает source transaction: bridge/crosswalk связывает старые source ID с global Party и сохраняет историю решения

## Unknown и late-arriving dimensions

Для каждого измерения существуют специальные строки:

- `0 — Unknown/not supplied`
- `-1 — Unresolved mapping`
- `-2 — Not applicable`
- `-3 — Restricted`

Это предотвращает потерю фактов из-за inner join. После разрешения mapping обновляется FK управляемым pipeline, а исходный ключ остаётся в lineage/audit. Доля `Unknown/Unresolved` является DQ-метрикой и имеет порог сертификации

## Согласование с финансовым учётом

`fact_transaction` предназначен для бизнес-аналитики и не заменяет бухгалтерский журнал. Для финансовой/регуляторной области используются отдельные совместимые факты:

| факт                    | Grain                                       | назначение                                       |
| ----------------------- | ------------------------------------------- | ------------------------------------------------ |
| `fact_accounting_entry` | одна debit/credit entry                     | tie-out с Core Ledger и регуляторная отчётность  |
| `fact_daily_balance`    | счёт × тип остатка × business date × валюта | EOD-позиция и balance roll-forward               |
| `fact_payment`          | одно платёжное распоряжение                 | lifecycle, execution time, reject/return metrics |

Они образуют fact constellation и используют conformed dimensions Party, Account, Agreement, Product, Date, Currency и Source System. Связь `transaction -> accounting entries` контролируется, но суммы разных grains не соединяются прямым join без заранее определённой bridge/aggregation

## Физическая оптимизация

### PostgreSQL DWH

- range partition `fact_transaction` по `business_date`/месяцу
- локальные индексы по часто используемым FK и unique source leg key
- bulk load в staging и atomic partition/table publish
- aggregate tables/materialized views только для измеренных BI-паттернов
- workload limits/read replica для тяжёлого BI

### MPP DWH

- partition по date
- distribution/cluster key выбирается нагрузочным тестом, обычно account/party либо transaction key
- небольшие dimensions реплицируются, крупные co-locate по частому join
- физические aggregate projections не меняют логическую звезду

## Semantic layer

Semantic layer определяет, а не оставляет на усмотрение каждого отчёта:

- `оборот`, `успешная операция`, `активный клиент`, `комиссионный доход`
- какие status/scope/reversal включаются
- используемую дату: event, posting или business date
- правила валютной конвертации и версию курса
- row/column security и purpose
- owner, formula version, effective date и certification status

## Почему звезда подходит для OLAP

- факты фильтруются по узким surrogate keys и партициям даты
- BI выполняет предсказуемые joins fact-to-dimension
- SCD2 воспроизводит атрибуты на момент операции
- conformed dimensions обеспечивают одинаковую трактовку FU/RB
- модель понятна аналитикам и semantic layer
- новые источники подключаются через canonical mappings без изменения grain факта

Ограничение: звезда не подходит для raw/replay, сложных графовых связей и хранения всех технических событий. Эти данные остаются в Lakehouse/графовой проекции
