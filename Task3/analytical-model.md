# Аналитическая модель: «звезда» для транзакций

## Выбор модели

Для анализа транзакций выбрана **схема «звезда»**. Она проще «снежинки» для BI, даёт предсказуемые join'ы и хорошо
работает в PostgreSQL DWH и в Trino поверх Iceberg.

## Fact

| Таблица              | Зерно             | Основные меры                                       |
|----------------------|-------------------|-----------------------------------------------------|
| `fact_transactions`  | одна транзакция   | `amount`, `fee`, `count`, `balance_after`           |
| `fact_payments`      | один платёж       | `amount`, `fee`, `count`                            |
| `fact_loan_payments` | платёж по кредиту | `amount`, `principal`, `interest`, `overdue_amount` |

## Dimensions

| Таблица            | Описание   | Атрибуты                                                       |
|--------------------|------------|----------------------------------------------------------------|
| `dim_customer`     | Клиент     | `party_id`, тип, сегмент, регион, KYC-статус, SCD2             |
| `dim_account`      | Счёт       | `account_id`, тип, валюта, статус, отделение, SCD2             |
| `dim_card`         | Карта      | `card_id`, тип, платёжная система, статус, SCD2                |
| `dim_product`      | Продукт    | `product_id`, код, название, тип, категория                    |
| `dim_tariff`       | Тариф      | `tariff_id`, ставка, комиссии, период действия                 |
| `dim_agreement`    | Договор    | `agreement_id`, тип, статус, даты                              |
| `dim_loan`         | Кредит     | `loan_id`, тип, ставка, срок, статус                           |
| `dim_channel`      | Канал      | `channel_id`, мобильное приложение, веб, отделение, колл-центр |
| `dim_branch`       | Отделение  | `branch_id`, регион, тип                                       |
| `dim_date`         | Дата       | день, месяц, квартал, год                                      |
| `dim_counterparty` | Контрагент | `counterparty_id`, тип, категория                              |

## Связи

- `fact_transactions` → `dim_customer`, `dim_account`, `dim_card`, `dim_product`, `dim_channel`, `dim_branch`,
  `dim_date`, `dim_counterparty`.
- `fact_payments` → `dim_customer`, `dim_account`, `dim_product`, `dim_channel`, `dim_date`.
- `fact_loan_payments` → `dim_loan`, `dim_customer`, `dim_product`, `dim_date`.

## Почему «звезда» подходит для OLAP

1. **Простые join'ы** — BI-инструменты быстро строят агрегаты.
2. **Предсказуемая производительность** — денормализованные измерения уменьшают число join'ов.
3. **Понятная бизнесу модель** — аналитики легко строят отчёты.
4. **SCD2 в измерениях** — сохраняется история изменений клиента, счёта, карты.
5. **Совместимость с PostgreSQL и Trino** — модель одинаково хорошо работает в DWH и Lakehouse gold-слое.
6. **Масштабирование** — факты партиционируются по дате, измерения — небольшие.

## SCD

- `dim_customer`, `dim_account`, `dim_card`, `dim_product`, `dim_tariff` — SCD Type 2 (история изменений).
- `dim_date`, `dim_branch`, `dim_channel` — статические.

## Где используется

- `fact_transactions` — BI, антифрод, финансовый мониторинг.
- `fact_payments` — сверка, финансы, регуляторика.
- `fact_loan_payments` — кредитный портфель, риск-менеджмент.