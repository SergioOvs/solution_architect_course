# Масштабирование баз данных: стратегии по сервисам
Выбор архитектуры для каждой БД продиктован паттерном нагрузки: частотой записи, требованиями к консистентности 
и объёмом данных.

## Bidding Service (PostgreSQL)
<b>Стратегия:</b> Master + Read Replica.<br>
Конфигурация бизнес-правил читается редко (при промахе кэша) и обновляется ещё реже. Объём данных мал, горизонтальное 
шардирование не требуется.
* <b>Write:</b> Обновление `bid_multiplier` и `min_acceptable` идёт в Master.
* <b>Read:</b> При недоступности Redis Bidding Service читает конфигурацию из Read Replica, чтобы не нагружать Master. 

## Campaign Service (PostgreSQL)
<b>Стратегия:</b> Master-Slave + Партиционирование + CQRS.<br>
Сервис разделяет транзакционные записи (Dashboard)и массовые чтения (Ad Server).
* <b>Репликация:</b> Read Replica обслуживает запросы при промахе кэша и построение отчётов в Dashboard.
* <b>CQRS:</b> Write-модель (Master) хранит нормализованные данные. Read-модель (Redis/Read Replica) — денормализованный 
плоский профиль кампании по `campaign_ref`, обновляемый через Kafka (`campaign.changed`).
* <b>Партиционирование:</b> Таблица `campaigns` разбита по диапазону `created_at` (например, по кварталам). Это ускоряет
архивацию старых данных и фильтрацию по дате.

## Budget Service (PostgreSQL)
<b>Стратегия:</b> Master + Read Replica + Оптимистичные локи.<br>
Шардирование исключено: распределённые транзакции недопустимы из-за риска нарушения консистентности баланса.
* <b>Write:</b> Резервирование и списание средств идут строго в Master с использованием
`UPDATE ... WHERE version = :expected_version`.
* <b>Read:</b> RTB hot path использует только Redis. Read Replica задействована исключительно для отображения баланса 
в Dashboard.

## Statistics Service (ClickHouse)
<b>Стратегия:</b> Native Clustering (ReplicatedMergeTree + Distributed tables).<br>
ClickHouse масштабируется встроенными средствами. Записи из Kafka распределяются по шардам.
* <b>Шардирование:</b> По campaign_ref (хэш `sipHash64(campaign_ref) % num_shards`). Большинство аналитических запросов 
фильтруются по кампании.
* <b>Запросы:</b> Выполняются через виртуальную таблицу `Distributed`, которая агрегирует данные со всех шардов.

## Analytics Service (ClickHouse / DWH)
</b>Стратегия:</b> Materialized Views + Read Replicas.<br>
Сервис строит витрины поверх сырых событий из Statistics Service.
* </b>Materialized Views:</b> Агрегация данных на лету. Пример витрины campaign_daily_stats с движком SummingMergeTree
для мгновенного подсчёта impressions, clicks и wins.
* </b>Изоляция:</b> Тяжёлые пользовательские запросы направляются на Read Replicas, чтобы не мешать ingestion-пайплайну
(записи сырых событий).

## Financial Service (PostgreSQL)
<b>Стратегия:</b> Master-Slave с синхронной репликацией.<br>
Требуется строгая ACID-консистентность и нулевая потеря данных. Шардирование недопустимо.
* <b>Репликация:</b> Используется <b><i>Synchronous replication</i></b>. Транзакция списания подтверждается только после
записи на Standby-узел. Это увеличивает latency write-операций, но гарантирует сохранность данных при падении Primary.
* <b>Failover:</b> Standby работает в режиме hot standby для мгновенного переключения. Чтение данных (аудит) возможно 
только с реплики.