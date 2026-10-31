# Целевая архитектура данных и аналитическая платформа

## Цель

Спроектировать целевую архитектуру данных объединённого банка через 12 месяцев:
операционные хранилища, аналитическую платформу, пайплайны, каталог данных и модель управления.

## Состав

- `target-data-architecture.md` — описание To-Be архитектуры по слоям.
- `architecture-options.md` — сравнение DWH на PostgreSQL, Data Lakehouse и Data Mesh, выбор подхода.
- `dwh-lakehouse.md` — сравнение DWH PG и Lakehouse, роль Iceberg и Trino, границы данных, ограничения.
- `data-pipelines.md` — 6 пайплайнов (включая простой пайплайн DWH PG на +3 мес) и 2 сценария больших исторических
  данных.
- `analytical-model.md` — модель «звезда» для транзакций и обоснование.
- `data-products.md` — 5 доменных data-продуктов, связанных с BIAN.
- `diagrams/target-architecture.puml` — PlantUML-схема To-Be архитектуры.
- `diagrams/pipeline-flow.puml` — схема потоков пайплайнов.
- `diagrams/star-schema.puml` — PlantUML-схема «звезды».
- `diagrams/data-mesh-domains.puml` — домены и data-продукты.

## Ключевые решения

1. **Гибридная архитектура:** DWH на PostgreSQL для регламентированной и управленческой отчётности + Data Lakehouse (
   Iceberg + Trino + объектное хранилище) для истории, событий, ML.
2. **MDM Customer Hub** — золотая запись клиента; **RDM** — справочники, продукты, тарифы.
3. **Integration Layer:** ESB для sync, Kafka для async/streaming, Debezium CDC из операционных БД.
4. **Airflow + dbt** — оркестрация и трансформации; **Spark/Flink** — обработка больших исторических и событийных
   данных.
5. **DataHub** — каталог, lineage, владельцы; **data products** — в формате Data Mesh-lite, без полного
   децентрализованного меша.
6. **Data Mesh-элементы** применяются выборочно: 5 доменных data-продуктов с владельцами, но платформа централизована.
7. **BIAN** используется для доменов: Party Data Management, Current Account, Customer Agreement, Card Management,
   Consumer Loan, Payment Execution, Financial Transaction Capture, Customer Case Management, Reference Data Management.

## Допущения

- Полный отказ от RB Core/Cards/Loans — к концу 12-го месяца; до этого split-master, описанный в Task2.
- Trino/Iceberg выбраны как референсные open-source компоненты Lakehouse.
- BI-инструмент — деталь реализации; архитектурно важна роль BI, а не вендор.
- Графовая БД и поисковый движок остаются вспомогательными компонентами (AML/антифрод, поиск по обращениям), а не ядром
  DWH.