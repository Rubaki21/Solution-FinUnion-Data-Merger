
## Применение Data Mesh

В FinUnion Group используются `принципы Data Mesh`, но не полностью децентрализованная платформа:

- домен отвечает за смысл, контракт, качество и SLO продукта
- центральная Platform Team предоставляет Kafka, Iceberg, Trino, DWH, Airflow, DataHub, IAM и шаблоны
- Data Governance Council утверждает общие термины, классификацию и conformed dimensions
- Finance централизованно сертифицирует регуляторные показатели
- DataHub является реестром discoverability, ownership, contracts, DQ и lineage

Data product - не произвольная таблица и не копия всей доменной БД. Это поддерживаемый набор/API/event contract с назначенными consumers, SLO, документацией и политикой доступа

## Набор целевых data products

| Data product                    | BIAN / бизнес-домен                                                                          | основные данные                                                                                                                | владелец                                            | потребители                                                   |
| ------------------------------- | -------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------- | ------------------------------------------------------------- |
| `Customer 360`                | Party Data Management, Customer Profile                                                      | golden Party, crosswalk, сегмент, подтверждённые контакты/согласия, связи, holdings summary, качество/актуальность             | Customer Data Owner / CRM-домен; MDM Steward        | CRM, контакт-центр, BI, marketing с consent, Risk, onboarding |
| `Product & Customer Holdings` | Product Directory, Customer Agreement, Current/Savings Account, Issued Device Administration | продукты/версии/тарифы, договоры, роли, счета, карты, кредиты и servicing system                                               | Product Portfolio Owner; Agreement/Account Stewards | Customer 360, BI, channels, migration teams, Finance          |
| `Payments & Transactions`     | Payment Order, Payment Execution, Card Transaction                                           | payment order lifecycle, канонические операции, auth/clearing/reversal links, комиссии, каналы, контрагенты в разрешённом виде | Head of Payments; Payment/Transaction Steward       | Customer 360, BI, antifraud, disputes, Finance                |
| `Credit Portfolio & Risk`     | Consumer Loan, Corporate Loan, Credit Risk                                                   | договор, график, задолженность, просрочка, обеспечение/роли, risk grade/version, агрегаты портфеля                             | Head of Lending; CRO владеет risk attributes        | Risk, Finance, Regulatory, Collections, BI                    |
| `Financial Position`          | Financial Accounting                                                                         | проводки, план счетов, EOD-остатки, reconciliation status, закрытые периоды, сертифицированные финансовые агрегаты             | Chief Accountant / Finance Data Owner               | Regulatory Reporting, Finance, Audit, ALM, executive BI       |

## 1. Customer 360

### Контракт

- `Grain:` одна версия global Party + связанные summaries по `as_of_time`
- `Интерфейсы:` operational read API; `golden-party` events; DWH dimension; Curated Iceberg history
- `SLO:` подтверждённое MDM-изменение доступно operational consumers за согласованные минуты; аналитическая история — в пределах batch SLO
- `DQ:` unique global ID, crosswalk cardinality, completeness по типу Party, match confidence, contact freshness, merge/split acknowledgement
- `Security:` field-level policies; сильные идентификаторы токенизированы; consent/purpose enforcement; массовая выдача ограничена

Holdings summary содержит ссылки и агрегаты, а не становится мастером счетов/карт/кредитов. Online balance читается из разрешённого operational API/cache с timestamp

## 2. Product & Customer Holdings

### Контракт

- `Grain:` версия продукта; версия договора/holding; роль Party в договоре
- `Интерфейсы:` Product Catalog API/events; Curated holdings tables; DWH conformed dimensions
- `SLO:` новая версия продукта публикуется до effective date; operational holding events - near real-time; EOD snapshot - по расписанию
- `DQ:` mapping FU/RB, непересекающиеся effective periods, договор ссылается на действующую версию продукта, задан `servicing_system`
- `Security:` договоры/holdings - банковская тайна; продуктовый каталог может иметь public/internal части

Продукт объединяет данные нескольких operational domains только как read-oriented portfolio. Статус карты меняет Cards, кредита - Loans, счёта - Core

## 3. Payments & Transactions

### Контракт

- `Grain:` одно payment order; одна canonical transaction leg; отдельно сохраняется связь с accounting entries
- `Интерфейсы:` Kafka events, Curated Iceberg tables, DWH transaction/payment facts
- `SLO:` события - near real-time; clearing и финансовая сертификация - после scheduled reconciliation
- `DQ:` idempotency, status sequence, event/posting/business time, auth-clearing-reversal linkage, duplicate rate, count/sum/hash
- `Security:` tokenized account/card identifiers, encrypted topics/zones, purpose-based counterparty access

Поле `certification_status` различает предварительный поток и финансово сверенную версию

## 4. Credit Portfolio & Risk

### Контракт

- `Grain:` кредит/договор + EOD-состояние + версия risk assessment
- `Интерфейсы:` lifecycle events; Standardized/Curated Iceberg; DWH credit snapshots
- `SLO:` lifecycle changes - near real-time; задолженность/risk snapshot - daily before reporting window
- `DQ:` schedule consistency, borrower/guarantor roles, debt-to-ledger reconciliation, delinquency rules, model/rule version
- `Security:` высокий класс банковской тайны/ПДн; разделение lending и independent risk access; audit model usage

Risk grade принадлежит Risk domain, договор и задолженность - Loan domain. Product contract объединяет их по versioned keys, не размывая ownership

## 5. Financial Position

### Контракт

- `Grain:` accounting entry и account/balance-type/business-date snapshot
- `Интерфейсы:` certified DWH facts/marts; controlled exports; агрегаты в Curated для разрешённых scenarios
- `SLO:` EOD готов к утверждённому Finance deadline; закрытый период доступен как immutable version
- `DQ:` debit=credit, valid chart of accounts, balance roll-forward, source-to-target totals, closed-period control
- `Security:` maker-checker, segregation of duties, immutable audit, ограниченный export

Только этот продукт получает статус authoritative analytical source для финансовой/регуляторной позиции после sign-off. Операционный мастер проводок и online balance остаётся Core Ledger

## Матрица ответственности

| роль               | домен                       | Central Platform              | Data Office / Governance      | ИБ / Finance                            |
| ------------------ | --------------------------- | ----------------------------- | ----------------------------- | --------------------------------------- |
| Бизнес-определение | отвечает                    | консультируется               | согласует общие термины       | Finance утверждает регулируемые понятия |
| Source contract    | отвечает с producer team    | предоставляет шаблон/registry | контролирует наличие metadata | ИБ согласует классификацию              |
| Pipeline runtime   | on-call за доменную логику  | on-call за платформу          | наблюдает SLO/DQ              | получает critical alerts                |
| DQ rules           | отвечает                    | исполняет/измеряет            | steward ведёт issue workflow  | Finance утверждает финансовые gates     |
| Доступ             | указывает purpose/consumers | реализует policy              | ведёт классификацию           | ИБ утверждает high-risk access          |
| Lineage            | описывает business links    | собирает technical lineage    | контролирует полноту          | Audit использует доказательства         |

## Этапы внедрения

1. `Пилот:` Customer 360 и Financial Position - максимально централизованно из-за MDM и отчётности
2. `Расширение:` Payments & Transactions и Credit Portfolio - доменные команды принимают ownership/SLO
3. `Портфель:` Product & Customer Holdings объединяет опубликованные доменные контракты
4. `Оценка зрелости:` покрытие owners, SLO, DQ, lineage, contract adoption и incident response
5. `Дальнейшая федерализация:` только домены, прошедшие maturity gate, получают больше автономии

Это позволяет использовать преимущества Data Mesh, не создавая пять несовместимых платформ в первый год после слияния
