# GET /v1/agent/motivation/incentives

Проект контракта. Актуализация 12.11.2026. Основа: прежний `../response-example.json`, Parquet от 09.11.2026 и новые экраны.

Это предлагаемый контракт, не реализованный сервис. Правило со статусом **Предложение** или **Открыто** не подтверждено. Сравнение «было / стало» описывает документированный пример, не доказанный работающий сервис.

Редакция ниже не меняет правила исходного текста. Повторы сведены в одно место: комментарии JSONC и второй полный пример удалены как дубли, их правила перенесены в таблицы. Единственный эталонный JSON — [фикстура A](#фикстура-a-агент-20fc435e-b1dd-4269-8866-880c76ca48f5). Значения и порядок ключей в нём совпадают с исходным полным примером.

| Статус | Что означает |
|---|---|
| Факт | Берётся из колонки как есть, без пересчёта |
| Правило | Зафиксировано в этом проекте |
| Предложение | Рабочая формула есть, но она в открытых вопросах |
| Открыто | Реализации нет. Подставлять удобное значение нельзя |

| Роль | Что читать |
|---|---|
| Разработчик | Паспорт, инварианты, модель, маппинг, сборка, формулы |
| Тестировщик | Инварианты, фикстуры, приёмка, открытые вопросы |
| Ревьюер контракта | Открытые вопросы и слабые места |

Проект OpenAPI: [incentives-openapi.proposed.json](incentives-openapi.proposed.json). Корневой `openapi.json` остаётся прежним документом. Формат ошибок, таймауты и авторизация — существующие сервисные, в этом тексте не переопределяются.

---

## 1. Паспорт запроса

| Параметр | Значение |
|---|---|
| Сервис | motivation-api |
| Метод | `GET /v1/agent/motivation/incentives` |
| Авторизация | Действующая авторизация сервиса. Отдельно читается обязательный заголовок `X-User-Id`, значение — UUID |
| Path, query, body | Нет |
| Успех | `200 application/json`. Отчёт только вызывающего агента |
| Источники | Общие Parquet по всем агентам в S3. Фильтр `agent_id` = UUID вызывающего |
| Чтение | AWS SDK for Java v2 |

Обозначения файлов S3, не таблиц новой БД:

- `summary` = `incentives_v3_agent_export_summary`
- `events` = `incentives_v3_agent_export_events`

Перед сборкой строки отбираются по `agent_id` вызывающего. `agent_id` в JSON ответа нет.

---

## 2. Инварианты

Эти правила уже зафиксированы. Их нельзя «упростить» при реализации.

1. В ответе всегда три секции, по одному объекту, в порядке `RECENT_LOANS`, `DEFERRED_LOANS`, `PROTECTION_PLANS`.
2. Суммы секций и `totalAmount` переносятся из `summary`. Их не складывают из месяцев и не пересчитывают, в том числе если секция отрицательная. `zeroingFlg` суммы не меняет.
3. Loan-секции группируются по `events.accrual_month_dt` внутри `period_cd`. `event_month_dt` не заменяет ни `period_cd`, ни месяц блока. Удержание с `event_month_dt = 2026-11-01` и `accrual_month_dt = 2026-08-01` остаётся в августе своей секции.
4. `payoutMonth` берётся из `summary.payout_month_dt`. Прибавлять месяц к дате события нельзя. В фикстуре A выплата `2026-12` при текущем месяце `2026-11` — совпадение данных, не формула.
5. `holdbackOriginMonth` берётся только из `summary.holdback_origin_month_dt`. Это месяц предупреждения, не ключ группировки и не первый элемент `months[]`. `null` в источнике остаётся `null`, месяц не угадывается.
6. Подписи, ставки строки, диапазон уровня и сумма строки берутся из выгрузки. Текущий `/reference` не подменяет исторические `product_condition_text`, `bonus_per_loan_amt` и диапазоны. Отсутствие `Basic` в `/reference` не удаляет `Basic` из отчёта.
7. `payoutSharePct` — число процентов: `80` означает 80%, не `0.2`. В контрольной формуле делитель `100`. Для защиты оба процентных поля не заполняются подстановкой `100%` или множителя.
8. `items[].amount` = готовый `bonus_amt` одной строки. Строку не пересчитывают. `deductionMultiplier` принадлежит этой строке: его не суммируют и не выбирают произвольным значением после `GROUP BY`. В одном месяце множители могут различаться.
9. `bonusRange` берётся из `bonus_per_loan_min_amt` / `bonus_per_loan_max_amt`, не из `MIN`/`MAX` по `bonus_per_loan_amt`.
10. Уровень месяца берётся из `events.bonus_tier_name` строк этого месяца, не из текущего уровня `summary`.
11. Группа создаётся по фактическому `product_type_name`, включая `Credit Lines`. Пустую группу не создают. Строка входит только в свою группу.
12. Полный дубль строки events удаляется до сумм, остаётся первая строка. Совпадение только названия или только суммы дублем не является. Разные строки одного продукта сохраняются.
13. Нулевые суммы `summary` — нормальный отчёт. Нулевые строки и детализацию не удаляют.
14. Неизвестный `event_type_cd` не классифицируется догадкой. Известные коды: `application_issued` и `rcl_utilized` → `BONUS`; `default_matured` → `DEDUCTION`.
15. `event_type_cd` переносится как в строке. По названию продукта его не выводят. `rcl_utilized` в макете Credit Lines — пример сценария, не правило определения типа.
16. По решению Северина поставщик исключает `application_issued` для продуктов Credit Line и Referred Credit Line. `rcl_utilized` остаётся начислением. `default_matured` остаётся удержанием и в базу начислений не входит. `application_issued` остальных продуктов сохраняется. Отдельный исключающий фильтр backend для уже исключённых поставщиком строк не добавляется.
17. В файлах от 09.11.2026 ещё есть строки до этого исключения. Выполнение договорённости проверяется на новой выгрузке, не на этих файлах.
18. Защита собирается только из `summary`, без `events` и без `months`. На loan-секции поля `earningMonth`, `basisAmount`, `tier`, `items` с уровня секции удаляются и рядом с `months` не дублируются.
19. Имена месячных полей фиксированы: `basisAmount` — база до процента, `amount` — итог месяца с удержаниями. Имена `monthAmount` и месячный `totalAmount` не вводятся.
20. `loadedAt` — время успешной записи агента в кеш. Это не `row_updated_at_dttm` и не timestamp источника. Timestamp источника без часового пояса в UTC произвольно не переводится.
21. `reportId` и `loadedAt` живут только в оболочке кеша. Внутри `report` их нет. Сериализация выносит их на верхний уровень JSON и заново не группирует и не пересчитывает отчёт.
22. Ошибка чтения не заменяет успешный кеш пустым отчётом и не меняет `loadedAt`. Пустой результат вместо сбоя не публикуется.
23. Деньги считаются десятичной арифметикой, не двоичной с плавающей точкой.
24. Названия месяцев клиент форматирует из `YYYY-MM`. Английских названий месяцев в файле нет.
25. Текст предупреждения, переводы служебных надписей и иконки не приходят из этих Parquet. Их нельзя описывать как поля S3.

Изменение вложенности несовместимо с прежним примером. Если старый контракт уже выпущен, переход приложения согласуется до удаления старых полей.

---

## 3. Ломающие изменения

Пути `items` ниже — это строка продукта. Вложенность новой схемы на смысл строки не влияет.

| Было в прежнем примере | Стало | Что делать |
|---|---|---|
| `reportId`, `loadedAt` на верхнем уровне | Сохранены | В кеше только в оболочке. Правило `reportId` для пары файлов открыто. `loadedAt` — успешная загрузка агента |
| `totalAmount`, `zeroingFlg` | Сохранены | Прямой перенос `current_final_payout_amt` и `zeroing_flg` |
| `payoutMonth` не было | `payoutMonth` | `summary.payout_month_dt`. Не вычислять |
| Порогов предупреждения не было | `zeroingDefaultRateThresholdPct`, `zeroingMinDefaultLoanCount` | Одноимённые по смыслу колонки `summary` |
| `sections[].id`, `sections[].amount`, три секции | Сохранены | RECENT: `current_immediate_bonus_amt`. DEFERRED: `holdback_release_after_deduction_amt`. PROTECTION: `current_warranty_bonus_amt` вместо прежнего `warranty_bonus_amt` |
| Один `earningMonth` у loan-секции | `months[].earningMonth` | Несколько месяцев. Группировка по `accrual_month_dt` |
| Месяца предупреждения не было | `DEFERRED_LOANS.holdbackOriginMonth` | `holdback_origin_month_dt`. Не из первого месяца |
| `basisAmount` секции из `summary` | `months[].basisAmount` из events | Gross-полей выгрузки больше нет. Сумма произведений — предложение. Фильтр и округление открыты |
| Один `tier` секции | `months[].tier` | `bonus_tier_name` строк месяца, не текущий уровень summary |
| Диапазона не было | `tier.bonusRange.minAmount` / `maxAmount` | `bonus_per_loan_min_amt` / `bonus_per_loan_max_amt` |
| Месячных итогов не было | `months[].amount`, `bonusAmount`, `deductionAmount` | Общий итог месяца, начисления и удержания отдельно. Сумму всей секции в каждый месяц не копировать |
| Общего процента месяца не было | `months[].payoutSharePct` | Одно значение начислений месяца. Разные проценты внутри месяца — открытый сценарий |
| `sections[].items[]` у loan-секций | `months[].groups[].items[]` | Строки внутри месяца и группы |
| Заголовков групп не было | `groups[].label` | `product_type_name` |
| `items[].id`, `displayOrder` | Сохранены в новой вложенности | Генерацию для месяцев и групп нужно пересмотреть. Позиционные id примера не стабильны между обновлениями |
| `entryType`, `eventTypeCode` | Сохранены | Классификация по `event_type_cd`. Удержания допустимы в обеих loan-секциях |
| `items[].label` | Сохранён | `product_display_name` новой выгрузки. Старое имя Product Loan не искать |
| `description` из конфига | `description` из events | Исторический `product_condition_text` |
| `items[].unitAmount` | `calculation.unitAmount` | `bonus_per_loan_amt` из S3, не из справочника |
| `items[].amount`, `calculation.quantity` | Сохранены | `bonus_amt`, `loan_cnt`. Сумму строки не пересчитывать |
| Процента и множителя строки не было | `calculation.payoutSharePct`, `deductionMultiplier` | Из events. Множитель относится к одной строке |
| У PROTECTION были `basisAmount: null`, `tier: null` | Поля у этого типа удалены | Остаются `earningMonth` и `items`. Объект `months` не добавляется |
| У строки защиты в примере не было label, description и ставки | `label`, `description: null`, `calculation.unitAmount` | `current_warranty_label_text`, подписи нет, `current_warranty_bonus_rate_amt`. Количество — `current_warranty_unit_cnt` |
| Процента и множителя защиты не было | Оба поля `null` | Не подставлять |
| Кеш одного месяца и плоских items | Кеш отчёта с месяцами и группами | Проверка записи агента и TTL 12 часов сохраняются. Меняется модель отчёта |

---

## 4. Модель и сериализация

Все ключи эталонного примера обязательны в своём типе объекта. `null` допустим только в полях, где он указан. `?` в модели — nullability контракта, не способ спрятать ошибочные данные.

Общие форматы:

| Формат | Правило |
|---|---|
| Money | `amount` — десятичная строка с четырьмя знаками после точки, минус допустим. `currency` — `PHP`, настройка сервиса. Колонки валюты в выгрузке нет |
| Месяц | `YYYY-MM`. В Parquet исходный тип — `DATE` |
| `loadedAt` | date-time в UTC |
| Проценты и множители | JSON-число. В памяти `BigDecimal` |
| Кеш и API | Количество, порядок, id и значения вложенных элементов совпадают |

Логическая модель. Конкретные классы выбирает разработчик. Кеш: `Map<UUID, CachedAgentIncentives>`. Ключ — `agent_id`. Обновляется целиком только запись этого агента. Сырые строки S3 нужны на время сборки. Постоянная копия выгрузки в этой модели не хранится.

```text
CachedAgentIncentives
  reportId: String
  loadedAt: Instant
  report: AgentIncentivesReport

AgentIncentivesReport
  payoutMonth: YearMonth?
  zeroingFlg: Int (0 | 1)
  zeroingDefaultRateThresholdPct: BigDecimal?
  zeroingMinDefaultLoanCount: Long?
  totalAmount: Money
  sections: List<RecentLoans | DeferredLoans | ProtectionPlans>

RecentLoans
  id = RECENT_LOANS
  amount: Money
  months: List<MonthBlock>

DeferredLoans
  id = DEFERRED_LOANS
  amount: Money
  holdbackOriginMonth: YearMonth?
  months: List<MonthBlock>

ProtectionPlans
  id = PROTECTION_PLANS
  amount: Money
  earningMonth: YearMonth?
  items: List<Item>

MonthBlock
  earningMonth: YearMonth
  amount: Money
  basisAmount: Money?
  payoutSharePct: BigDecimal?
  bonusAmount: Money
  deductionAmount: Money
  tier: Tier?
  groups: List<ProductGroup>

Tier
  name: String
  bonusRange: BonusRange
BonusRange
  minAmount: Money
  maxAmount: Money
ProductGroup
  label: String
  items: List<Item>
Item
  id: String
  entryType: BONUS | DEDUCTION
  eventTypeCode: String?
  displayOrder: Int          // ≥ 0
  label: String
  description: String?
  amount: Money
  calculation: Calculation
Calculation
  quantity: Long
  unitAmount: Money?
  payoutSharePct: BigDecimal?
  deductionMultiplier: BigDecimal?
Money
  amount: BigDecimal
  currency: String           // PHP
```

`GET` сериализует `report` и добавляет `reportId` и `loadedAt` из оболочки на верхний уровень. Поля внутри объекта сериализуются так: `Money.amount` — строка с четырьмя знаками, `YearMonth` — `YYYY-MM`, `Instant` — UTC date-time, проценты и множители — JSON-число.

Расхождение типов, которое разработчику нельзя тихо выбрать: в таблице контракта `zeroingMinDefaultLoanCount` и `calculation.quantity` названы integer, в модели — `Long` / `Long?`. В JSON это число. Отдельного решения, integer это или int64, нет. В макете Credit Lines процент записан как `80`, в остальных примерах как `80.0`. Для JSON-числа это одно значение.

Порядок ключей в эталонном JSON и в модели не совпадает: в полном примере `holdbackOriginMonth` стоит после `months`, в модели — до `months`. Исходный текст не фиксирует, какой порядок контрактный. Эталон ниже сохраняет порядок исходного полного примера. Расхождение разобрано в слабых местах.

---

## 5. Маппинг полей

Одна строка таблицы — одно поле. Статус относится к правилу заполнения.

### 5.1. Корень

| Поле | JSON | Null | Источник | Статус |
|---|---|---|---|---|
| `reportId` | string | нет | Идентификатор согласованной пары файлов. Не гарантированно `calculation_run_id`. Способ сборки не закреплён | Открыто |
| `loadedAt` | date-time UTC | нет | Успешная публикация записи агента в кеш. Не `row_updated_at_dttm` | Правило |
| `payoutMonth` | `YYYY-MM` | да, если агента нет | `summary.payout_month_dt` | Факт |
| `zeroingFlg` | integer `0` или `1` | нет | `summary.zeroing_flg` | Факт |
| `zeroingDefaultRateThresholdPct` | number | да | `summary.zeroing_default_rate_threshold_pct` | Факт |
| `zeroingMinDefaultLoanCount` | integer | да | `summary.zeroing_min_default_loan_cnt` | Факт |
| `totalAmount` | Money | нет | `summary.current_final_payout_amt` | Факт |
| `sections` | array из 3 элементов | нет | Порядок фиксирован: RECENT, DEFERRED, PROTECTION | Правило |

### 5.2. Секции

| Поле | JSON | Null | Источник | Статус |
|---|---|---|---|---|
| `sections[].id` | enum | нет | `current` → `RECENT_LOANS`. `holdback_origin` → `DEFERRED_LOANS`. Защита из summary, без period | Правило |
| `sections[].amount` | Money | нет | RECENT: `current_immediate_bonus_amt`. DEFERRED: `holdback_release_after_deduction_amt`. PROTECTION: `current_warranty_bonus_amt` | Факт |
| `DEFERRED_LOANS.holdbackOriginMonth` | `YYYY-MM` | да | `summary.holdback_origin_month_dt` | Факт |
| `LoanSection.months` | array | нет | Events этого `period_cd`, группы по `accrual_month_dt`. Предлагаемый порядок месяцев — по убыванию | Правило / порядок — Предложение |
| `PROTECTION.earningMonth` | `YYYY-MM` | да | `summary.current_month_dt` | Факт |
| `PROTECTION.items` | array | нет | Ровно одна строка из колонок `current_warranty_*` | Правило |

Массивы `summary.current_screen_app_month_dts` и `summary.holdback_screen_accrual_month_dts` агрегированы из events и содержат те же месяцы. Блоки ответа из этих массивов не строятся. Строятся из events по `accrual_month_dt`.

### 5.3. Месяц loan-секции

| Поле | JSON | Null | Источник | Статус |
|---|---|---|---|---|
| `earningMonth` | `YYYY-MM` | нет | `events.accrual_month_dt` | Факт |
| `amount` | Money | нет | `SUM(bonus_amt)` всех строк месяца после удаления полных дублей. Начисления и удержания вместе | Предложение |
| `basisAmount` | Money | да | `SUM(loan_cnt × bonus_per_loan_amt)` подходящих начислений. Для месяца только с удержаниями предлагается `null`, а не сумма ставок удержаний | Предложение |
| `payoutSharePct` | number | да | Единый `payout_share_pct` начислений месяца. Не константа 80/20. Если начислений с процентом нет — `null`. `null` у удержания не означает другое условие месяца | Правило единственности; сценарий разных процентов — Открыто |
| `bonusAmount` | Money | нет | `SUM(bonus_amt)` строк `BONUS`. Если их нет — денежный `0`, не `null` | Предложение |
| `deductionAmount` | Money | нет | `SUM(bonus_amt)` строк `DEDUCTION`, знак сохраняется. Если их нет — денежный `0` | Предложение |
| `tier` | object | да | Единые условия строк месяца. Произвольный уровень не выбирается | Правило; поведение при конфликте — Открыто |
| `tier.name` | string | нет, если tier есть | `events.bonus_tier_name` | Факт |
| `tier.bonusRange.minAmount` | Money | нет, если tier есть | `events.bonus_per_loan_min_amt` | Факт |
| `tier.bonusRange.maxAmount` | Money | нет, если tier есть | `events.bonus_per_loan_max_amt` | Факт |
| `groups` | array | нет | Группировка месяца по `product_type_name` | Правило |

Условия месяца объясняются срезом на последний день месяца: внутри блока едины `bonus_tier_name`, min/max и процент начислений. У удержаний процент может быть `null`. Условия разных месяцев и ставки разных продуктов различаться могут. Дополнительно делить месяц не требуется.

Для `DEFERRED_LOANS` удержание считается так: `deductionAmount = SUM(bonus_amt)` при том же `agent_id`, `period_cd = holdback_origin`, `event_type_cd = default_matured`, группа `accrual_month_dt`. Полные дубли снимаются раньше. Та же формула суммы действует и для `RECENT_LOANS`, если удержание попало в `current`.

### 5.4. Группа и строка

| Поле | JSON | Null | Источник | Статус |
|---|---|---|---|---|
| `groups[].label` | string | нет | `events.product_type_name`, готовый заголовок | Факт |
| `groups[].items` | array | нет | Строки группы после удаления полных дублей. Одна запись = одна строка events | Правило |
| `items[].id` | string | нет | Генерирует backend. В примерах id позиционные и иллюстративные | Открыто |
| `items[].displayOrder` | integer ≥ 0 | нет | Предлагается порядок источника внутри группы. Колонки ранга в S3 нет. Порядок групп не задан | Предложение |
| `items[].entryType` | `BONUS` \| `DEDUCTION` | нет | См. инвариант 14 | Правило |
| `items[].eventTypeCode` | string | да | `events.event_type_cd`. У защиты всегда `null` | Факт |
| `items[].label` | string | нет | `events.product_display_name`. У защиты `current_warranty_label_text` | Факт |
| `items[].description` | string | да | `events.product_condition_text`. У защиты `null`: отдельной подписи нет | Факт |
| `items[].amount` | Money | нет | `events.bonus_amt`, знак сохраняется. У защиты `current_warranty_bonus_amt` | Факт |
| `calculation.quantity` | integer | нет | `events.loan_cnt`. У защиты `current_warranty_unit_cnt` | Факт |
| `calculation.unitAmount` | Money | да | `events.bonus_per_loan_amt`. У защиты `current_warranty_bonus_rate_amt` | Факт |
| `calculation.payoutSharePct` | number | да | `events.payout_share_pct`. У защиты `null` | Факт |
| `calculation.deductionMultiplier` | number | да | `events.default_deduction_multiplier` той же строки. У защиты `null` | Факт |
| `items[].code` | не входит в текущий пример и модель | — | Согласована связь `events.product_category_cd` → `items[].code` → `reference.details[].id`. Отдельный `detailsId` не нужен. Колонки в выгрузке ещё нет | Открыто |

Строка защиты, сверх таблицы: `id` фиксирован как `warranty`, `entryType = BONUS`, `displayOrder = 0`. События events у неё нет.

---

## 6. Сборка и ошибки

Кеш независим на каждой копии сервиса. Обновление ленивое, по запросу, без фонового расписания. Истечение TTL не означает, что исходный расчёт в S3 свежий. Блокировка одного `agentId` не синхронизирует загрузку других агентов. Ошибка не сбрасывает TTL: следующий запрос может снова пойти в S3.

1. Проверить авторизацию и `X-User-Id`. Найти запись агента.
2. Если запись есть и возраст по `loadedAt` не больше 12 часов, вернуть её. Иначе взять блокировку `agentId` и проверить снова. Если другой запрос уже обновил запись, вернуть её и в S3 не ходить.
3. Прочитать общие Parquet `summary` и `events`. Проверить структуру и согласованность публикации, затем отобрать `agent_id`. Пустой, неполный или нечитаемый набор не считается отсутствием агента.
4. Если нет ни `summary`, ни events этого агента — нулевой отчёт `200` по ранее согласованному сценарию: nullable-поля и пустые `months`, описанные контрактом. Events без summary — ошибка согласованности. Несколько summary — ошибка. Первую строку summary не выбирать.
5. При имеющемся summary удалить полные дубли events, оставив первую строку. Summary с нулями не повод удалять детализацию.
6. Разложить events: `period_cd = current` → `RECENT_LOANS`, `holdback_origin` → `DEFERRED_LOANS`. Дальше `accrual_month_dt`, затем `product_type_name`. Состав событий и формулы — разделы 7 и 11. `event_month_dt` как запасной ключ не использовать.
7. Собрать шапки месяцев и строки. Уровень, диапазон и процент начислений брать из общих условий месяца. Защиту собрать отдельно из summary.
8. Итоги секций, выплату, флаги обнуления и `holdbackOriginMonth` перенести из summary.
9. После всех проверок одной заменой записать `CachedAgentIncentives` вместе с `reportId` и `loadedAt`. Уже начатые запросы могут дочитать прежний неизменяемый объект. История версий не хранится.
10. При ошибке прежнюю запись и её `loadedAt` не менять. Если успешной записи нет — `503`. Программная ошибка — `500`.

Граница TTL включительная: возраст ровно 12 часов ещё считается свежим, потому что условие записано как «≤ 12 часов».

| Ситуация | Результат |
|---|---|
| Некорректная идентификация | `400` |
| Авторизация или доступ | `401` / `403` |
| Запись свежая, либо её уже обновил параллельный запрос | `200`, без повторного чтения S3 |
| Агента нет ни в summary, ни в events | `200`, нулевой отчёт |
| Ошибка, успешной записи в кеше нет | `503` |
| Программная ошибка | `500` |
| Ошибка, успешная запись уже есть | Запись и `loadedAt` не меняются. Пустой отчёт вместо сбоя не пишется. HTTP-код именно этого запроса не задан |

---

## 7. Формулы

### 7.1. Контрольная формула строки

Экран задаёт базу `loan_cnt × bonus_per_loan_amt` и результат после доли:

```text
loan_cnt × bonus_per_loan_amt × payout_share_pct / 100
```

Это расчёт после процента. Удержания в него сами не входят. Готовый `bonus_amt` этой формулой не заменяется.

По месяцу:

```text
basisAmount      = SUM(quantity × unitAmount) по подходящим начислениям
calculatedBonus  = SUM(quantity × unitAmount × payoutSharePct / 100)
bonusAmount      = SUM(bonus_amt) по начислениям
deductionAmount  = SUM(bonus_amt) по удержаниям, со знаком минус
amount           = bonusAmount + deductionAmount
```

`calculatedBonus` — только контрольное вычисление. Поля API из него нет. Место и точность округления нужны от поставщика. Состав «подходящих начислений» не закрыт, кроме уже принятого: удержания в базу не входят, а исключение `application_issued` для двух кредитных продуктов делает поставщик.

### 7.2. Месяц только с удержаниями

При одинаковом множителе экран показывает одну формулу на все удержания месяца. Отдельное поле API для неё не нужно.

```text
−((quantity₁ × unitAmount₁ + quantity₂ × unitAmount₂ + …) × deductionMultiplier) = months[].amount
```

Количество `1` на экране можно не показывать. Число справа берётся из `months[].amount`. Множители и ставки — из `groups[].items[].calculation`. Фильтр All / Bonuses / Deductions состав этой формулы не меняет.

Согласованный пример другого агента, не фикстуры A. `agent_id` `f899e04a-ab69-4808-9e85-5f1bfcc5d6ef`, August:

| Строка | Расчёт | Множитель | Сумма строки |
|---|---|---|---|
| Referred Small | 1 × 75 | 1.5 | −112.5 |
| Referred Standard | 1 × 150 | 1.5 | −225 |

```text
−((₱75 + ₱150) × 1.5) = −₱337.50
```

`basisAmount` суммой ставок удержаний не заполняется. `null` для месяца без начислений остаётся предложением. Формат при разных множителях и округление открыты.

### 7.3. Credit Lines

Группа добавляется в `months[].groups[]` рядом с `Product Loans with ECP`, только если в этом месяце есть `product_type_name = "Credit Lines"`.

Значения ниже с макета: ₱60, количество 1, ставка ₱75, доля 80%. Это не события агента фикстуры A. Формула макета: `1 × ₱75 × 80% = ₱60`.

```json
{
  "label": "Credit Lines",
  "items": [
    {
      "id": "current-2026-10-credit-line-0",
      "entryType": "BONUS",
      "eventTypeCode": "rcl_utilized",
      "displayOrder": 0,
      "label": "Credit Line",
      "description": "If used within 7 days",
      "amount": { "amount": "60.0000", "currency": "PHP" },
      "calculation": {
        "quantity": 1,
        "unitAmount": { "amount": "75.0000", "currency": "PHP" },
        "payoutSharePct": 80,
        "deductionMultiplier": null
      }
    }
  ]
}
```

`id` иллюстративный. `eventTypeCode` в ответе равен фактическому `events.event_type_cd`.

---

## 8. Привязка к экрану

| Элемент макета | Откуда брать | Чего не делать |
|---|---|---|
| ₱19,900 сверху детализации | `sections[].amount` | Не сумма только первого месяца |
| November / October | `months[].earningMonth` | Название форматирует клиент |
| ₱24,800 или ₱75 перед процентом | `months[].basisAmount` | Не пересчитывать из справочника |
| 80% | `months[].payoutSharePct` | Не делить на 100 при показе |
| ₱19,840 или ₱60 после знака = | `months[].amount` | Готовые бонусы отдельно лежат в `bonusAmount` |
| Удержание в формуле месяца | `months[].deductionAmount` | Для одного удержания ставка и множитель — в `items[].calculation` |
| Silver / Bronze | `months[].tier.name` | Не текущий уровень summary |
| ₱100–₱350 / ₱75–₱300 | `tier.bonusRange` | Не min/max ставок строк |
| Product Loans with ECP / Credit Lines | `groups[].label` | Пустую группу не рисовать |
| Small / Standard / Credit Line | `items[].label` | Не искать старое имя Product Loan |
| Подпись под продуктом | `items[].description` | Не заменять текущим справочником |
| Сумма справа | `items[].amount` | Не пересчитывать строку |
| Количество × ставка × процент | `items[].calculation` | Показывать только когда применимость формулы подтверждена |
| All / Bonuses / Deductions | Фильтр `items` по `entryType` внутри месяца | Итоги шапки при фильтре не пересчитывать. Это предложение |
| Canceled и предупреждение | `zeroingFlg` и корневые `zeroing*` | Готовых английских текстов в этих выгрузках нет |
| Иконка информации у Standard / Large | `items[].code` ← `product_category_cd` → `reference.details[].id` | Колонки ещё нет. Какая версия шторки историческая — открыто |

Шаблон предупреждения описывает условие, а не вычисляет обнуление: «более {zeroingDefaultRateThresholdPct}% и не менее {zeroingMinDefaultLoanCount} неоплаченных займов» за `holdbackOriginMonth`. Факт обнуления — только `zeroingFlg`. Готового текста в S3 нет.

Иконки, переводы надписей Tier / Bonus range per loan / Unpaid и тексты предупреждений — ресурсы UI или отдельно согласованный конфиг.

---

## 9. Фикстуры

`reportId`, `loadedAt`, позиционные `id` и `displayOrder` иллюстративные. `basisAmount = null` у августа — предложение. Остальные числа фикстуры A взяты из выгрузки либо посчитаны явно указанным правилом на этом агенте. Сравнение примера не является тестом работающего API.

### Фикстура A. Агент `20fc435e-b1dd-4269-8866-880c76ca48f5`

Пять строк events, три месячных блока. Защита — шестой объект, из summary, не из events.

| Проверка | Ожидание |
|---|---|
| Выплата | `payoutMonth = 2026-12`, `totalAmount = 180.0000`, `zeroingFlg = 0`, порог `14.0`, минимум займов `3` |
| RECENT | Сумма секции `0.0000`. Один месяц `2026-11`: amount `0`, basis `0`, share `80`, bonus `0`, deduction `0`, tier `Basic`, диапазон `0–0` |
| RECENT, строки | Обе нулевые строки сохраняются. Standard: qty 1, unit 0, amount 0, доля 80, описание `₱4000 - ₱12000 or ₱20000 and more by segment`. Large: qty 2, unit 0, amount 0, описание `₱12000 or ₱20000 and more by segment`. Обе `application_issued` / `BONUS`, группа `Product Loans with ECP` |
| DEFERRED, секция | Сумма `180.0000` из summary. `holdbackOriginMonth = 2026-09` |
| Сентябрь | `2026-09`, Bronze, диапазон `100–300`, basis `2400` = `4×100 + 10×200`, bonus `480` = `80+400`, deduction `0`, share `20`, amount `480`. Standard qty 4 / unit 100 / amount 80, описание `₱4000 - ₱15000`. Large qty 10 / unit 200 / amount 400, описание `₱15000 and more` |
| Август | `2026-08`, хотя `event_month_dt` удержания `2026-11-01`. Platinum, диапазон `200–500`, basis `null`, share `null`, bonus `0`, deduction `-300`, amount `-300`. Одна строка `default_matured` / `DEDUCTION`, Standard, qty 1, unit 200, multiplier `1.5`, amount `-300`, описание `₱4000 - ₱15000` |
| Защита | `earningMonth = 2026-11`, сумма секции `0`. Строка `warranty`: BONUS, `eventTypeCode null`, order 0, label `Loans with protection plans`, description `null`, amount `0`, qty `0`, unit `50`, доля `null`, множитель `null` |
| Совпадение, которое нельзя принимать за правило | `480 + (−300) = 180`, то есть сумма месяцев равна сумме секции. Контракт требует брать `180` из summary, а не из этой суммы |

```json
{
  "reportId": "example-publication-id",
  "loadedAt": "2026-11-09T03:00:00Z",
  "payoutMonth": "2026-12",
  "zeroingFlg": 0,
  "zeroingDefaultRateThresholdPct": 14.0,
  "zeroingMinDefaultLoanCount": 3,
  "totalAmount": {
    "amount": "180.0000",
    "currency": "PHP"
  },
  "sections": [
    {
      "id": "RECENT_LOANS",
      "amount": { "amount": "0.0000", "currency": "PHP" },
      "months": [
        {
          "earningMonth": "2026-11",
          "amount": { "amount": "0.0000", "currency": "PHP" },
          "basisAmount": { "amount": "0.0000", "currency": "PHP" },
          "payoutSharePct": 80.0,
          "bonusAmount": { "amount": "0.0000", "currency": "PHP" },
          "deductionAmount": { "amount": "0.0000", "currency": "PHP" },
          "tier": {
            "name": "Basic",
            "bonusRange": {
              "minAmount": { "amount": "0.0000", "currency": "PHP" },
              "maxAmount": { "amount": "0.0000", "currency": "PHP" }
            }
          },
          "groups": [
            {
              "label": "Product Loans with ECP",
              "items": [
                {
                  "id": "current-2026-11-0-0",
                  "entryType": "BONUS",
                  "eventTypeCode": "application_issued",
                  "displayOrder": 0,
                  "label": "Referred Standard",
                  "description": "₱4000 - ₱12000 or ₱20000 and more by segment",
                  "amount": { "amount": "0.0000", "currency": "PHP" },
                  "calculation": {
                    "quantity": 1,
                    "unitAmount": { "amount": "0.0000", "currency": "PHP" },
                    "payoutSharePct": 80.0,
                    "deductionMultiplier": null
                  }
                },
                {
                  "id": "current-2026-11-0-1",
                  "entryType": "BONUS",
                  "eventTypeCode": "application_issued",
                  "displayOrder": 1,
                  "label": "Referred Large",
                  "description": "₱12000 or ₱20000 and more by segment",
                  "amount": { "amount": "0.0000", "currency": "PHP" },
                  "calculation": {
                    "quantity": 2,
                    "unitAmount": { "amount": "0.0000", "currency": "PHP" },
                    "payoutSharePct": 80.0,
                    "deductionMultiplier": null
                  }
                }
              ]
            }
          ]
        }
      ]
    },
    {
      "id": "DEFERRED_LOANS",
      "amount": { "amount": "180.0000", "currency": "PHP" },
      "months": [
        {
          "earningMonth": "2026-09",
          "amount": { "amount": "480.0000", "currency": "PHP" },
          "basisAmount": { "amount": "2400.0000", "currency": "PHP" },
          "payoutSharePct": 20.0,
          "bonusAmount": { "amount": "480.0000", "currency": "PHP" },
          "deductionAmount": { "amount": "0.0000", "currency": "PHP" },
          "tier": {
            "name": "Bronze",
            "bonusRange": {
              "minAmount": { "amount": "100.0000", "currency": "PHP" },
              "maxAmount": { "amount": "300.0000", "currency": "PHP" }
            }
          },
          "groups": [
            {
              "label": "Product Loans with ECP",
              "items": [
                {
                  "id": "holdback_origin-2026-09-0-0",
                  "entryType": "BONUS",
                  "eventTypeCode": "application_issued",
                  "displayOrder": 0,
                  "label": "Referred Standard",
                  "description": "₱4000 - ₱15000",
                  "amount": { "amount": "80.0000", "currency": "PHP" },
                  "calculation": {
                    "quantity": 4,
                    "unitAmount": { "amount": "100.0000", "currency": "PHP" },
                    "payoutSharePct": 20.0,
                    "deductionMultiplier": null
                  }
                },
                {
                  "id": "holdback_origin-2026-09-0-1",
                  "entryType": "BONUS",
                  "eventTypeCode": "application_issued",
                  "displayOrder": 1,
                  "label": "Referred Large",
                  "description": "₱15000 and more",
                  "amount": { "amount": "400.0000", "currency": "PHP" },
                  "calculation": {
                    "quantity": 10,
                    "unitAmount": { "amount": "200.0000", "currency": "PHP" },
                    "payoutSharePct": 20.0,
                    "deductionMultiplier": null
                  }
                }
              ]
            }
          ]
        },
        {
          "earningMonth": "2026-08",
          "amount": { "amount": "-300.0000", "currency": "PHP" },
          "basisAmount": null,
          "payoutSharePct": null,
          "bonusAmount": { "amount": "0.0000", "currency": "PHP" },
          "deductionAmount": { "amount": "-300.0000", "currency": "PHP" },
          "tier": {
            "name": "Platinum",
            "bonusRange": {
              "minAmount": { "amount": "200.0000", "currency": "PHP" },
              "maxAmount": { "amount": "500.0000", "currency": "PHP" }
            }
          },
          "groups": [
            {
              "label": "Product Loans with ECP",
              "items": [
                {
                  "id": "holdback_origin-2026-08-0-0",
                  "entryType": "DEDUCTION",
                  "eventTypeCode": "default_matured",
                  "displayOrder": 0,
                  "label": "Referred Standard",
                  "description": "₱4000 - ₱15000",
                  "amount": { "amount": "-300.0000", "currency": "PHP" },
                  "calculation": {
                    "quantity": 1,
                    "unitAmount": { "amount": "200.0000", "currency": "PHP" },
                    "payoutSharePct": null,
                    "deductionMultiplier": 1.5
                  }
                }
              ]
            }
          ]
        }
      ],
      "holdbackOriginMonth": "2026-09"
    },
    {
      "id": "PROTECTION_PLANS",
      "earningMonth": "2026-11",
      "amount": { "amount": "0.0000", "currency": "PHP" },
      "items": [
        {
          "id": "warranty",
          "entryType": "BONUS",
          "eventTypeCode": null,
          "displayOrder": 0,
          "label": "Loans with protection plans",
          "description": null,
          "amount": { "amount": "0.0000", "currency": "PHP" },
          "calculation": {
            "quantity": 0,
            "unitAmount": { "amount": "50.0000", "currency": "PHP" },
            "payoutSharePct": null,
            "deductionMultiplier": null
          }
        }
      ]
    }
  ]
}
```

### Фикстура B

Макет Credit Lines из раздела 7.3. В фикстуру A не подмешивать и пять строк агента A этими данными не дополнять.

### Фикстура C

Агент `f899e04a-ab69-4808-9e85-5f1bfcc5d6ef`, август, общий множитель `1.5`. Проверяет только объединённую формулу удержаний. Тело всего отчёта для этого агента не задано.

---

## 10. Критерии приёмки проекта

Проверки относятся к описанному примеру и согласованной структуре. Сценарии из открытых вопросов этими пунктами не закрываются.

| ID | Ожидание |
|---|---|
| AC-1 | В обеих loan-секциях месяц берётся из `accrual_month_dt`. `event_month_dt` не переносит удержание в другую секцию и не переносит августовское удержание фикстуры A в ноябрь |
| AC-2 | Все 5 строк events фикстуры A сохранены в 3 месячных блоках. Группа каждой строки равна её `product_type_name` |
| AC-3 | Сентябрь: Bronze, диапазон 100–300, база 2400, бонусы 480. Август: Platinum, диапазон 200–500, удержание −300. Сумма `DEFERRED_LOANS` = 180 и взята из summary |
| AC-4 | Нулевые ноябрьские строки на месте. `Basic` не удалён из-за отсутствия `Basic` в `/reference` |
| AC-5 | Готовые суммы и исторические подписи не заменены текущим справочником |
| AC-6 | Полные дубли не удваивают базу и суммы. Разные строки одного продукта сохраняются |
| AC-7 | Ошибка S3 не заменяет успешный кеш пустым ответом и не сбрасывает время загрузки |
| AC-8 | Числовые вычисления десятичные. Сравнение примера не является тестом работающего API |

AC-2 действует на выгрузку, в которой эти 5 строк ещё есть. После исключения поставщиком `application_issued` для Credit Line состав новой выгрузки будет другим. Один и тот же чеклист на оба файла без версии выгрузки не переносится.

---

## 11. Открытые вопросы

Пока вопрос не закрыт, соответствующее поведение не считается требованием. Временную реализацию нельзя выдавать за контракт.

| ID | Вопрос | Что уже решено и не нужно переоткрывать | Что блокирует |
|---|---|---|---|
| Q1 | Окончательный состав остальных начислений, входящих в `basisAmount` | Исключение `application_issued` для Credit Line и Referred Credit Line делает поставщик. `default_matured` в базу не входит. Отдельный фильтр backend на исключённые строки не добавляется | Фильтр базы и приёмку формулы месяца |
| Q2 | Подтвердить `SUM(bonus_amt)` как итог месяца, округление и допустимый разрыв с summary | В проекте итог месяца предлагается из `bonus_amt`, не из контрольной формулы | `months[].amount`, `bonusAmount`, `deductionAmount` и сверку с суммой секции |
| Q3 | Для месяца только с удержаниями: оставить ли `basisAmount = null` и как показывать разные множители | Объединённая формула при общем множителе согласована. Новое поле API не требуется | Отрисовку deduction-only месяца |
| Q4 | Какая историческая версия информационной шторки | Связь `items[].code` ← `product_category_cd` = `reference.details[].id`. Отдельный `detailsId` не нужен. Колонку предстоит добавить. В прежних файлах `code` нет | Иконку информации. В текущий JSON поле не входит |
| Q5 | Стабильные id и порядок групп и строк | Отдельного ключа и ранга в S3 нет. Позиционные id примера стабильность не гарантируют | `items[].id`, `displayOrder`, порядок `groups[]` |
| Q6 | Production-объекты S3, готовность согласованной пары и формула `reportId` | Timestamp источника без часового пояса в UTC произвольно не переводится | Шаг сборки «проверить согласованность публикации» и поле `reportId` |

---

## 12. Слабые места

Ниже то, что в исходном тексте есть, но разработчик и тестировщик не могут выполнить однозначно. Это не новые требования.

1. **Нулевой `200` не описан.** Алгоритм отсылает к ранее согласованному сценарию. В этом документе нет тела: остаются ли три секции, есть ли строка `warranty`, чем заполнены `zeroingFlg` и деньги. Тест «агента нет» написать нельзя.

2. **Ошибки согласованности без кода и без определения.** Не задано, что такое пустой, неполный и нечитаемый набор и чем это отличается от «агента нет». Не задан HTTP-код для нескольких summary, events без summary и несхождения пары файлов. Если успешный кеш уже есть, не сказано, получает ли текущий запрос старый `200`, `503` или `500`. AC-7 запрещает только порчу кеша.

3. **Шаг проверки пары файлов обязателен и одновременно открыт.** Q6 не даёт ни признака согласованной публикации, ни формулы `reportId`. Шаг 3 сборки в текущем виде не реализуется.

4. **Дубль и «первая строка» зависят от порядка чтения.** Полный дубль не определён набором колонок. «Первая» строка при нестабильном порядке Parquet не воспроизводится. От этого зависят суммы, `displayOrder` и id. Q5 это признаёт, но дедупликация уже стоит в алгоритме как правило.

5. **Неизвестный код некуда деть.** `entryType` обязателен и не nullable, а неизвестный `event_type_cd` нельзя угадывать. Не сказано, проваливать всю загрузку или пропускать строку. То же для `period_cd` вне `current` и `holdback_origin`: такой строки нет ни в одной секции, и это не названо ошибкой.

6. **Конфликт условий месяца не описан.** Требуются единые tier, диапазон и процент начислений, произвольный выбор запрещён. Нет реакции, если в одном `accrual_month_dt` два `bonus_tier_name`, две пары min/max или два `payout_share_pct`. Разные проценты упомянуты как открытый сценарий, разный tier — нет. Условия, при которых `tier = null`, тоже не заданы.

7. **Порядок групп отсутствует.** Порядок месяцев — предложение, убывание. Порядок строк — предложение, порядок источника. Порядок `groups[]` не задан даже как предложение. «Рядом с Product Loans» для Credit Lines не является сортировкой.

8. **`items[].code` висит между контрактом и следующим изменением.** Поле есть в привязке к экрану и в Q4. Его нет в модели, маппинге обязательных ключей и эталонном JSON. Нельзя понять, входит ли оно в эту версию ответа.

9. **Суммы секции и месяцев в фикстуре A случайно сходятся.** `180 = 480 − 300`. Тест «секция равна сумме месяцев» будет зелёным и противоречит правилу «не пересчитывать». Нет примера, где summary и сумма месяцев различаются, и нет допуска на расхождение. Q2 это держит открытым. Ноябрьский `basisAmount = 0.0000` посчитан по предложению и выглядит как факт, в отличие от августа, где `null` честно помечен.

10. **Типы целых чисел расходятся.** `zeroingMinDefaultLoanCount`: integer в контракте и `Long?` в модели. `quantity`: integer и `Long`. Отдельного диапазона нет.

11. **`X-User-Id` не доведён до проверки.** Не сказано, что заголовок всегда равен `agent_id` в Parquet и субъекту токена. Не разделены отсутствие заголовка, не-UUID и чужой UUID между `400` и `401`/`403`.

12. **Кеш локальный, а свежесть данных им не гарантируется.** Две реплики могут отдать разные отчёты. TTL считается от записи в кеш, не от расчёта. Это записано, но ожидаемого наблюдения для теста нет: какой максимальный возраст Parquet допустим, не определено. `loadedAt` легко спутать с `row_updated_at_dttm` и с timestamp источника без таймзоны.

13. **Предупреждение не связано с фактом обнуления.** Пороги только подставляются в текст. Нет правила показа при `zeroingFlg = 1` и `null` в пороге или месяце, и нет правила при `zeroingFlg = 0`. Шаблон в документе русский, рядом сказано, что готовых английских текстов нет. Язык клиента не фиксирован.

14. **Формулу на экране показывать некогда.** Показ `количество × ставка × процент` разрешён только при подтверждённой применимости, а применимость закрывают Q1–Q3. До этого расхождение формулы с `bonus_amt` не является дефектом. Это ограничение стоит держать в заголовке тест-плана.

15. **Нулевая защита не определена как UI-состояние.** В фикстуре строка остаётся при quantity `0`, amount `0` и ставке `50`. Не сказано, можно ли скрыть секцию. Не сказано, бывает ли ставка `null` при ненулевой сумме, хотя `unitAmount` в модели nullable.

16. **Месячные массивы summary не являются ни источником, ни проверкой.** Они «содержат те же месяцы», но расхождение с events не названо ошибкой согласованности и не названо допустимым.

17. **AC-8 ослабляет AC-1–AC-7, если читать их как smoke живого сервиса.** Пункты 1–7 проверяют фикстуру и сборку описанного примера. Они не подтверждают, что сервис уже так отвечает.

---

## Приложение. Колонки, на которые опирается контракт

`summary`: `payout_month_dt`, `zeroing_flg`, `zeroing_default_rate_threshold_pct`, `zeroing_min_default_loan_cnt`, `current_final_payout_amt`, `current_immediate_bonus_amt`, `holdback_release_after_deduction_amt`, `current_warranty_bonus_amt`, `holdback_origin_month_dt`, `current_month_dt`, `current_warranty_label_text`, `current_warranty_unit_cnt`, `current_warranty_bonus_rate_amt`, `current_screen_app_month_dts`, `holdback_screen_accrual_month_dts`.

Не используются как источник ответа: `row_updated_at_dttm`, произвольный timestamp без часового пояса, прежнее имя `warranty_bonus_amt`, удалённые gross-поля базы.

`events`: `agent_id`, `period_cd`, `accrual_month_dt`, `event_month_dt` (только как антипример ключа), `event_type_cd`, `bonus_amt`, `loan_cnt`, `bonus_per_loan_amt`, `payout_share_pct`, `bonus_tier_name`, `bonus_per_loan_min_amt`, `bonus_per_loan_max_amt`, `product_type_name`, `product_display_name`, `product_condition_text`, `default_deduction_multiplier`.

Ожидается позже и в текущий ответ не входит: `product_category_cd`.
