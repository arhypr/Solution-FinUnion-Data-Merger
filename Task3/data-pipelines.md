# Пайплайны данных

## Сводная таблица

| #  | Пайплайн                                        | Источник                                                 | Приёмник                                              | Тип                           | Инструменты                           |
|----|-------------------------------------------------|----------------------------------------------------------|-------------------------------------------------------|-------------------------------|---------------------------------------|
| P1 | Interim DWH: клиенты, счета, продукты, операции | FU CRM, RB CRM, FU Core, RB Core                         | DWH PostgreSQL (staging → core → marts)               | Batch (ELT)                   | Airflow, dbt, PostgreSQL              |
| P2 | Клиентские мастер-данные                        | FU CRM, RB CRM, Core (ref)                               | MDM Customer Hub → CRM/Core/DWH                       | CDC + streaming + sync        | Debezium, Kafka, MDM, ESB             |
| P3 | Транзакции и платежи (CDC → Lakehouse → DWH)    | FU Core, RB Core, FU Cards, RB Cards, FU Loans, RB Loans | Lakehouse (bronze/silver/gold) → DWH marts            | CDC + streaming + micro-batch | Debezium, Kafka, Spark, Iceberg, dbt  |
| P4 | Справочники и тарифы                            | RDM                                                      | MDM → Core/Cards/Loans → DWH                          | Batch + event                 | Airflow, Kafka, dbt                   |
| P5 | История карточных операций (большой объём)      | FU Cards, RB Cards (архив)                               | Object Storage → Lakehouse (Iceberg) → агрегаты в DWH | Batch                         | Airflow, Spark, Iceberg, Trino        |
| P6 | Регуляторная отчётность                         | DWH                                                      | Регулятор                                             | Batch                         | Airflow, dbt, шаблоны форм            |
| P7 | Обращения клиентов                              | FU CRM, RB CRM                                           | DWH + Lakehouse (поиск)                               | CDC + streaming               | Debezium, Kafka, Spark, Search Engine |

## P1. Interim DWH на PostgreSQL (+3 месяца) — простой пайплайн

**Назначение:** минимально работоспособная отчётность по клиентам, счетам, продуктам и операциям.

**Шаги:**

1. **Extract:** Airflow забирает инкременты из FU CRM, RB CRM, FU Core, RB Core (по `updated_at`/CDC).
2. **Load:** сырые данные в `staging` PostgreSQL DWH.
3. **Transform (dbt):**
    - `stg_*` — очистка, приведение типов, дедупликация.
    - `core_*` — консолидация клиентов (через `party_id` из MDM), счетов, продуктов.
    - `mart_*` — витрины: `mart_customers`, `mart_accounts`, `mart_products`, `mart_operations`, `mart_appeals`.
4. **Test:** dbt-тесты (уникальность, not null, referential integrity).
5. **Serve:** BI читает `mart_*`.

**Почему простой:** batch, без streaming, без Lakehouse; достаточно для +3 месяцев и регуляторики.

## P2. Клиентские мастер-данные

**Назначение:** золотая запись клиента и единый `party_id`.

**Шаги:**

1. FU CRM и RB CRM пишут изменения в Kafka (или CDC через Debezium).
2. MDM потребляет события, выполняет матчинг (ФИО + ДР + ИНН + телефон + email), создаёт/обновляет golden record.
3. MDM публикует `party_id` в Kafka.
4. FU CRM, RB CRM, Core, DWH потребляют `party_id` и обогащают свои записи.
5. Delete/Block по 152-ФЗ — отдельное событие, MDM рассылает его потребителям.

**Контроль:** дедупликация, конфликты матчинга, DLQ, сверка количества клиентов.

## P3. Транзакции и платежи (CDC → Lakehouse → DWH)

**Назначение:** консолидированная история операций для BI, антифрода и финансов.

**Шаги:**

1. Debezium снимает CDC из FU Core, RB Core, FU Cards, RB Cards, FU Loans, RB Loans.
2. Kafka передаёт события в Spark Structured Streaming / Flink.
3. Данные пишутся в Lakehouse `bronze` (raw).
4. `silver` — очистка, нормализация кодов операций, связывание с `party_id`, `account_id`.
5. `gold` — агрегаты и витрины (`fact_transactions`).
6. dbt в DWH строит `mart_transactions` и `mart_payments` для BI и регуляторики.

**Контроль:** row count, сумма vs проводки, дубли по `(account_id, ts, amount)`, сверка с DWH.

## P4. Справочники и тарифы

**Назначение:** единые продукты и тарифы во всех системах.

**Шаги:**

1. Product Office вносит изменения в RDM (workflow, 4-eyes).
2. RDM публикует версии в MDM.
3. MDM рассылает обновления в Core, Cards, Loans через Kafka.
4. DWH получает копию справочников для отчётности.

**Контроль:** уникальность кода, версионность, пересечение периодов тарифов.

## P5. История карточных операций (большой объём)

**Назначение:** обработка архива карточных операций после слияния.

**Сценарий:**

- **Источник:** FU Cards, RB Cards (архивные выгрузки, исторические файлы).
- **Характер данных:** исторические, событийные, полуструктурированные (JSON/CSV/Parquet).
- **Целевое хранилище:** Object Storage → Lakehouse (Iceberg, bronze/silver/gold); агрегаты — в DWH.
- **Способ обработки:** Batch (Spark), оркестрация Airflow.
- **Потребители:** BI, антифрод, риск-менеджмент, регуляторная отчётность.

**Шаги:**

1. Архивные выгрузки загружаются в Object Storage (`bronze`).
2. Spark читает, валидирует схему, пишет в Iceberg `silver`.
3. Нормализация статусов карт и кодов операций через RDM.
4. Агрегаты (обороты, средние чеки, аномалии) — в `gold`.
5. dbt переносит агрегаты в DWH для BI и регуляторики.

## P6. Регуляторная отчётность

**Назначение:** формы регулятору в срок.

**Шаги:**

1. Airflow запускает расчёт форм из DWH.
2. dbt формирует таблицы форм.
3. Контроль контрольных сумм и сверка с источниками.
4. Выгрузка в формате регулятора, ЭП, доставка.

**Контроль:** контрольные суммы, сверка с DWH, журнал доставки.

## P7. Обращения клиентов

**Назначение:** единая картина обращений, поиск, качество сервиса.

**Шаги:**

1. FU CRM и RB CRM → CDC → Kafka.
2. Spark маскирует ПДн, обогащает `party_id`, классифицирует тему.
3. Данные в Lakehouse `silver` и DWH `mart_appeals`.
4. Индекс в Search Engine для контакт-центра.

**Контроль:** маскирование ПДн, schema, row count, DLQ.

## Сценарии больших данных (отдельно)

### Сценарий A. История карточных операций после слияния

- **Источник:** архивы FU Cards и RB Cards.
- **Характер:** исторические, событийные, полуструктурированные.
- **Целевое хранилище:** Object Storage → Lakehouse (Iceberg).
- **Способ обработки:** Batch (Spark), Airflow.
- **Потребители:** антифрод, BI, риск-менеджмент, регуляторика.

### Сценарий B. События клиентских каналов (мобильное приложение, веб, колл-центр)

- **Источник:** события каналов из FU/RB приложений и CRM.
- **Характер:** событийные, streaming, полуструктурированные.
- **Целевое хранилище:** Kafka → Lakehouse (Iceberg, bronze/silver).
- **Способ обработки:** Streaming (Flink/Spark), micro-batch.
- **Потребители:** маркетинг, BI, UX-аналитика, антифрод.