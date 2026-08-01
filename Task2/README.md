# Task2 — Проектирование решения (Privacy by Design, C4 Context To-Be)

## Артефакт

[`c4-context-to-be.drawio`](c4-context-to-be.drawio) — диаграмма **C4 Context** целевого состояния (MVP + контур на год вперёд по интеграциям и аналитике).

Открывать в [diagrams.net](https://app.diagrams.net/) или расширении Draw.io.

## Новые блоки Privacy by Design

Эти системы пересекают почти все бизнес-компоненты и закрывают риски из Task1 **до** полной реализации продуктов:

| Блок | Зачем (PbD) |
|------|-------------|
| **IAM / AuthZ** | единая идентификация; RBAC + ABAC (`patient_id`, роль, согласие) — пациент видит только свои данные |
| **Privacy & Consent** | каталог/теги `data.class`, согласия, минимизация полей, retention и право на удаление |
| **Audit & Monitoring** | журнал доступа к ПДн, алерты на аномалии (задел под Victoria Metrics / SIEM) |
| **KMS / Secrets** | ключи для encryption at rest / field-level; раздельные ключи для `pii.special` |

Принципы в архитектуре:

- **Privacy by Design / by Default** — теги и политики на входе в CRM, Lab, Portal, а не «потом».
- **Data Minimization** — API-контракты (лаборатория, уведомления, платежи) без лишних ПДн.
- **Data Lineage** — происхождение данных учитывается при ETL в аналитику и в audit.

## Аналитический слой с учётом PbD

**Analytics Platform** (Data Lake / BI / ML) — отдельный контур:

1. **raw** — ограниченный доступ, шифрование, без прямого доступа аналитиков к ПДн в открытом виде.  
2. **curated** — нормализованные сущности с тегами.  
3. **anonymized / serving** — витрины для BI, LLM и ML **после обезличивания** (связь с Privacy & Consent и Audit).

Потоки из CRM, Lab Integration и Payment идут в озеро только через ETL с деперсонализацией; выгрузки аудируются. Так компания может развивать BI/ML без повторной перевалидации каждой интеграции «с нуля».

## MVP vs целевой контур (кратко)

- **MVP (месяцы):** Client Portal + Staff Portal + CRM + напоминания; PbD-блоки обязательны с первого релиза.  
- **Далее:** Payment Gateway, Lab API, мобильный ЛК, Analytics Platform, голосовые/push-оповещения — на том же каркасе политик и тегов.
