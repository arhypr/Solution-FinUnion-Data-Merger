# Основные потоки данных

## Промежуточное состояние (+3 месяца)

### 1. Клиентские данные → MDM

| #  | Источник | Приёмник               | Сущности               | Действие                      | Режим               | Обоснование                                       |
|----|----------|------------------------|------------------------|-------------------------------|---------------------|---------------------------------------------------|
| F1 | FU CRM   | MDM                    | Клиент                 | Create / Update / Deduplicate | Async (CDC + Kafka) | Золотая запись клиента; дедупликация FU-клиентов  |
| F2 | RB CRM   | MDM                    | Клиент                 | Create / Update / Deduplicate | Async (CDC + Kafka) | Золотая запись клиента; дедупликация RB-клиентов  |
| F3 | FU Core  | MDM                    | Клиент (ссылки)        | Read / Validate               | Sync (REST)         | Валидация `client_id` при открытии счёта          |
| F4 | RB Core  | MDM                    | Клиент (ссылки)        | Read / Validate               | Sync (REST)         | Валидация `client_id` при открытии счёта          |
| F5 | MDM      | FU CRM / RB CRM        | Клиент (golden record) | Enrich / Update               | Async (event)       | Обогащение CRM golden `party_id`                  |
| F6 | MDM      | FU Core / RB Core      | Клиент (`party_id`)    | Enrich                        | Async (event)       | Связывание счетов и договоров с единым `party_id` |
| F7 | MDM      | FU CRM / RB CRM / Core | Клиент                 | Delete / Block                | Async               | Исполнение требований 152-ФЗ и блокировок         |

### 2. Справочники → MDM / RDM

| #   | Источник       | Приёмник                                                      | Сущности                   | Действие                   | Режим                   | Обоснование                                      |
|-----|----------------|---------------------------------------------------------------|----------------------------|----------------------------|-------------------------|--------------------------------------------------|
| F8  | Product Office | RDM                                                           | Продукт, Тариф, Справочник | Create / Update / Validate | Batch + workflow        | Управляемые версии продуктов и тарифов           |
| F9  | RDM            | MDM                                                           | Продукт, Тариф, Справочник | Create / Update            | Batch                   | Публикация справочников в MDM                    |
| F10 | MDM            | FU Core / RB Core / FU Cards / RB Cards / FU Loans / RB Loans | Продукт, Тариф, Справочник | Read / Enrich              | Async event + sync read | Единые коды продуктов и тарифов во всех системах |

### 3. Операционные данные → Integration Layer → DWH

| #   | Источник | Приёмник | Сущности                          | Действие                   | Режим                   | Обоснование                      |
|-----|----------|----------|-----------------------------------|----------------------------|-------------------------|----------------------------------|
| F11 | FU Core  | IL → DWH | Счёт, Договор, Транзакция, Платёж | CDC / Validate / Aggregate | Streaming → micro-batch | Единая отчётность по FU-счетам   |
| F12 | RB Core  | IL → DWH | Счёт, Договор, Транзакция, Платёж | CDC / Validate / Aggregate | Streaming → micro-batch | Единая отчётность по RB-счетам   |
| F13 | FU Cards | IL → DWH | Карта, Транзакция                 | CDC / Validate             | Streaming → micro-batch | Карточные операции FU            |
| F14 | RB Cards | IL → DWH | Карта, Транзакция                 | CDC / Validate             | Streaming → micro-batch | Карточные операции RB            |
| F15 | FU Loans | IL → DWH | Кредит, Платёж                    | CDC / Validate             | Streaming → micro-batch | Кредитный портфель FU            |
| F16 | RB Loans | IL → DWH | Кредит, Платёж                    | CDC / Validate             | Streaming → micro-batch | Кредитный портфель RB            |
| F17 | FU CRM   | IL → DWH | Обращение клиента                 | CDC / Enrich               | Streaming → micro-batch | Единая картина обращений         |
| F18 | RB CRM   | IL → DWH | Обращение клиента                 | CDC / Enrich               | Streaming → micro-batch | Единая картина обращений         |
| F19 | MDM      | IL → DWH | Клиент (golden)                   | CDC / Enrich               | Streaming               | Аналитика по единому клиенту     |
| F20 | RDM      | IL → DWH | Справочники                       | CDC                        | Streaming               | Согласованные разрезы отчётности |

### 4. Аналитика и отчётность

| #   | Источник | Приёмник  | Сущности                                             | Действие         | Режим              | Обоснование              |
|-----|----------|-----------|------------------------------------------------------|------------------|--------------------|--------------------------|
| F21 | DWH      | BI        | Клиент, Счёт, Продукт, Транзакция, Кредит, Обращение | Aggregate / Read | Sync (SQL/Parquet) | Витрины и дашборды       |
| F22 | DWH      | Регулятор | Регламентированные формы                             | Aggregate / Read | Batch              | Регуляторная отчётность  |
| F23 | IL       | DataHub   | Метаданные, lineage                                  | Read / Enrich    | Async              | Lineage и каталог данных |
| F24 | DWH      | DataHub   | Метаданные, lineage                                  | Read / Enrich    | Batch              | Lineage по витринам      |

## Целевое состояние (12 месяцев) — отличия

- Мастер-системы: единые FU Core, FU Cards, FU Loans, FU CRM, MDM, RDM.
- RB Core, RB Cards, RB Loans выводятся из эксплуатации.
- Потоки F12, F14, F16, F18 заменяются на единые FU-потоки.
- Добавляется Data Lakehouse для исторических и событийных данных.
- Регуляторная отчётность строится из DWH, аналитика — из Lakehouse/DWH.