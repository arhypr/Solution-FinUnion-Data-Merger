# Доменные data-продукты

## Подход

Data Mesh применяется в формате **Mesh-lite**: data products имеют владельцев, контракты и SLA, но платформа и
governance централизованы (Data Office). Полный mesh для всей организации не проектируется.

## Реестр data-продуктов

| Data product                    | Домен                      | Основные данные                                                 | Владелец                    | Потребители                         | BIAN                                              |
|---------------------------------|----------------------------|-----------------------------------------------------------------|-----------------------------|-------------------------------------|---------------------------------------------------|
| **Customer 360**                | Party / Customer Profile   | Клиент, сегмент, контакты, согласия, KYC-статус, `party_id`     | Data Office / CRM-домен     | CRM, BI, маркетинг, риск-менеджмент | Party Data Management                             |
| **Accounts & Balances 360**     | Current Account / Savings  | Счета, остатки, статусы, отделения, валюта                      | Core Banking Ops            | BI, финансы, риск, CRM              | Current Account / Savings Account                 |
| **Payments & Transactions 360** | Payment Execution / Ledger | Транзакции, платежи, статусы, каналы, контрагенты               | Payment Ops / Data Platform | BI, антифрод, финансы, регуляторика | Payment Execution / Financial Transaction Capture |
| **Credit Portfolio**            | Consumer / Corporate Loan  | Кредиты, остатки, просрочка, скоринг, графики платежей          | Credit Risk                 | BI, риск-менеджмент, регуляторика   | Consumer Loan / Corporate Loan                    |
| **Cards & Fraud Signals**       | Card Management            | Карты, статусы, карточные операции, аномалии, антифрод-признаки | Card Ops / Fraud            | Антифрод, BI, CRM                   | Card Management                                   |
| **Service Quality 360**         | Customer Case Management   | Обращения, темы, SLA, каналы, удовлетворённость                 | Contact Center Ops          | BI, качество сервиса, CRM           | Customer Case Management                          |

(6 продуктов — превышает минимум 3–5; можно оставить 5, объединив Cards и Payments, но лучше оставить отдельно из-за
антифрода.)

## Формат контракта data product

- **Владелец** и **Data Steward**.
- **Схема** (Avro/JSON Schema) и версия.
- **SLA** (freshness, availability).
- **Качество** (правила, тесты).
- **Lineage** в DataHub.
- **Доступ** (RBAC/ABAC, маскирование ПДн).
- **Потребители** и их ожидания.

## Как связаны с BIAN

- **Party Data Management** → Customer 360.
- **Current Account / Savings Account** → Accounts & Balances 360.
- **Payment Execution / Financial Transaction Capture** → Payments & Transactions 360.
- **Consumer Loan / Corporate Loan** → Credit Portfolio.
- **Card Management** → Cards & Fraud Signals.
- **Customer Case Management** → Service Quality 360.

## Почему не полный Data Mesh

- После слияния двух банков нет зрелых доменных data-команд.
- Нужен единый governance и каталог (DataHub).
- Регуляторная отчётность требует централизованного контроля.
- Data products дают пользу (владельцы, контракты, SLA), не создавая организационного хаоса.