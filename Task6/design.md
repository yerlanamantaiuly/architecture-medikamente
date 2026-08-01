# Задание 6. Движок классификации данных (Analytics + Privacy by Design)

Диаграмма C2: [`c2-classification-engine.drawio`](c2-classification-engine.drawio).

## 1. Назначение

Перед пакетной загрузкой в аналитическое хранилище потоки из CRM, Lab Integration и Payment **не размечены** и могут **менять схему**. Движок:

1. регистрирует/проверяет схему;
2. присваивает класс конфиденциальности (`public` / `internal` / `pii` / `pii.special` / `payment`);
3. по политике решает: куда писать, что маскировать/обезличивать, что шифровать;
4. пишет lineage и audit.

## 2. Контейнеры движка (C2)

| Контейнер | Роль |
|-----------|------|
| **Batch Orchestrator** | расписание, retry, gate по версии схемы |
| **Ingestion Landing** | приём batch без доверия к «входным» меткам |
| **Schema Registry & Evolution** | Avro/JSON Schema; backward/forward compat; блок breaking change |
| **Classification Engine** | rules (имена/паттерны/словарь) + ML-признаки; confidence score |
| **Policy Engine (OPA)** | маршрутизация в зоны; deny в serving без класса |
| **Mask / De-identify** | tokenization, generalization, агрегаты для BI/ML |
| **Data Catalog** | теги, владельцы, lineage |
| **Quality & Metrics** | оценка качества классификации |
| **KMS adapter** | ключи для RAW/CURATED |
| **Audit Log** | доступ к сырым/ПДн зонам |

## 3. Слои хранилища и доп. меры

| Слой | Содержимое | Доступ | Меры |
|------|------------|--------|------|
| **RAW** | близко к источнику; возможны ПДн | только engine + steward | encrypt at rest; без SQL для аналитиков; TTL/retention |
| **CURATED** | нормализованные таблицы с **column tags** | ограниченный (need-to-know) | RLS/ABAC; частичный PII под маской; отдельный ключ |
| **ANONYMIZED / SERVING** | витрины BI/ML | аналитики, LLM-фичи | по умолчанию без прямых ПДн; re-id risk review |
| **Metadata** | схемы, версии, labels | платформенная | источник правды для CI/каталога |

Дополнительно к слоям:

- **нет класса → нельзя в SERVING** (fail-closed);
- **crypto-shredding / TTL** на RAW по `data.retention`;
- **раздельные ключи KMS** raw vs curated;
- **CI-проверка**: пайплайн не мержится, если новая колонка без правила классификации;
- связка с **Privacy & Consent** (таксономия тегов из Task1/Task2).

Учёт эволюции схем: Schema Registry хранит версии; Classification Engine версионирует правила (`rule_set_id` + `schema_fingerprint`); при новой колонке — статус `unclassified` → карантин в RAW до approve steward.

## 4. Метрики эффективности классификации

| Метрика | Смысл | Как помогает оптимизировать |
|---------|--------|-----------------------------|
| **Coverage** | % полей/колонок с присвоенным классом | находит «дыры», блокирует рост unclassified |
| **Precision / Recall** (по золотому набору steward) | точность vs полнота детекта `pii*` | тюнинг rules/ML; снижение ложных `public` |
| **False Negative rate для `pii.special`** | пропуск медданных | P0-алерт; ужесточение словарей |
| **Time-to-classify** | задержка batch от landing до tagged curated | узкие места оркестрации/модели |
| **Quarantine volume** | объём unclassified в RAW | приоритет ручной разметки |
| **Policy deny / reroute count** | сколько записей не пустили в serving | качество контрактов источников |
| **Schema break attempts** | попытки breaking change | дисциплина продюсеров CRM/Lab |
| **Re-identification risk score** (выборочно на витринах) | остаточный риск в ANONYMIZED | донастройка обобщения/k-anon |

Метрики уходят в Victoria Metrics; падение recall по `pii.special` или рост quarantine — стоп выкладки serving.

## 5. Scalability и рост

| Ось роста | Подход |
|-----------|--------|
| Объём данных | горизонтальные batch-воркеры (Spark/Flink); партиции по дате/источнику; object storage для RAW; ClickHouse/колоночные витрины для SERVING |
| Число источников | плагины ingest + независимые rule packs; Schema Registry per domain |
| Число пользователей BI | SERVING масштабируется отдельно; RAW/CURATED не открываются «всем»; кэш витрин |
| Сложность классов | вынос ML-модели классификации в отдельный сервис с GPU/CPU pool по мере нужды; rules остаются быстрым путём |
| Филиалы ×5 | партиционирование `clinic_id`; квоты оркестратора; backpressure на ingest |

При росте: сначала scale-out оркестрации и SERVING, затем выделение Classification Engine в отдельный деплой с автоскейлом по размеру batch.

## 6. Связь с PbD

Классификация **до** попадания в озеро реализует Privacy by Default: аналитика работает с ANONYMIZED, конфиденциальное остаётся в контролируемых RAW/CURATED с шифрованием, тегами, аудитом и политиками — без перевалидации каждой новой интеграции с нуля.
