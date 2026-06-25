# Пересчет цены пакета вниз для валютных заказов

## 1. Что сделано

- Добавлен отдельный сценарий ручного пересчета цены пакета вниз для валютного заказа.
- Старый пересчет вверх сохранен как отдельное действие в `Order Tools`.
- Для нового сценария добавлены:
  - история пересчетов в БД;
  - отдельная админка с журналом;
  - автоматическое завершение и автоматический откат.

## 2. Почему сделано именно так

- Пересчет вниз вынесен отдельно, чтобы не ломать и не переопределять старую логику пересчета вверх.
- Меняется только пакет, потому что задача относится к валютной стоимости основного туристического продукта, а не к допуслугам.
- Все расчеты собраны в одном сервисе `Order::PriceRecalculations::Calculation`, чтобы:
  - не дублировать формулы между preview и apply;
  - не размазывать бизнес-правила по нескольким одноразовым сервисам;
  - держать единый источник истины для расчета.
- Контроллер вызывает `Calculation` напрямую для предпросмотра, а `Apply` вызывает его повторно для применения. Это избавило от пустой обертки `Preview`.
- Автообработка вынесена в воркер по расписанию, чтобы:
  - завершать пересчет, если клиент успел закрыть долг;
  - откатывать пересчет после `18:00`, если долг остался.

## 3. Что меняется в заказе

### Меняется

- `package.net_price`
- `package.fuel_charge`

### Не меняется

- `order_extras`
- `package.stat_exchange_rate`
- `package.stat_currency_name`
- клиентские платежи
- платежи туроператору

## 4. Основные правила

- Пересчет применим только к валютным заказам.
- У заказа должен быть хотя бы один оплаченный платеж.
- У заказа должен оставаться долг по пакету.
- Одновременно у заказа может быть только один активный down-recalculation.
- После отмены или завершения можно запускать новый пересчет повторно.
- Если целевой курс не уменьшает цену пакета, пересчет не применяется.

## 5. Таблица `order_price_recalculations`

Создана таблица для аудита и жизненного цикла пересчета.

### Поля

- `order_id` — заказ, для которого запускался пересчет.
- `package_id` — пакет, чья цена менялась.
- `status` — `applied`, `completed`, `canceled`, `rejected`.
- `source_currency`, `source_rate` — исходная валюта и курс пакета.
- `target_currency`, `target_rate` — целевая валюта и курс пересчета.
- `package_price_before`, `package_price_after` — цена пакета до и после пересчета.
- `net_price_before`, `fuel_charge_before` — snapshot компонентов пакета до применения.
- `net_price_after`, `fuel_charge_after` — snapshot компонентов пакета после применения.
- `applied_at`, `expires_at`, `completed_at`, `canceled_at` — временные точки жизненного цикла.
- `rejection_reason` — причина отказа.
- `cancel_reason` — причина отката.
- `initiator_id` — пользователь, запустивший пересчет; поле nullable, потому что запуск может быть автоматическим.
- `created_at`, `updated_at`

### Ограничения

- В таблице связей не используются foreign keys.
- Для обязательных связей стоят `not null`.
- Для `initiator_id` `null` разрешен.

### Индексы

- `(order_id, status)`
- `package_id`
- `(expires_at, status)`
- `initiator_id`

## 6. Модели и админка

- Добавлена модель `OrderPriceRecalculation`.
- В `Order` добавлена связь `has_many :order_price_recalculations`.
- Добавлена отдельная ActiveAdmin-страница `order_price_recalculations` для просмотра истории, статусов и ключевых значений пересчета.
- Добавлены переводы модели, атрибутов и статусов в `ru.yml`.

## 7. Как считается пересчет

Расчет выполняет `Order::PriceRecalculations::Calculation`.

### Что проверяется до расчета

- нет ли уже активного пересчета;
- валютный ли пакет;
- есть ли оплаченные платежи;
- остался ли долг по пакету;
- есть ли валютная стоимость пакета;
- есть ли исходный курс пакета;
- можно ли определить целевой курс;
- меньше ли целевой курс, чем исходный.

### Откуда берется целевой курс

- `source_currency = package.stat_currency_name`
- `source_rate = package.stat_exchange_rate`
- если пакет меняли, `target_rate = source_rate`
- если пакет не меняли, `target_rate = operator_exchange_rate` последнего оплаченного платежа

Признак замены пакета определяется внутри `Calculation` по:
- `OrderLog kind: 'package_replace'`
- fallback через `order.versions` с изменением `package_id`

### Базовые суммы

- `current_package_price = package.full_price`
- `paid_amount_rub = order.available_certificate_amount + сумма оплаченных платежей`

Если у всех релевантных платежей есть `operator_exchange_rate`, дополнительно считается валютный хвост долга:

- `paid_amount_in_foreign_currency`
- `debt_in_foreign_currency`

### Кандидатная новая цена

Если у всех платежей есть курс:

- `candidate_package_price = target_rate * debt_in_foreign_currency + paid_amount_rub`

Если хотя бы у одного платежа курс отсутствует:

- `candidate_package_price = target_rate * package.full_price_in_foreign_currency`

### Ограничение по regular flights

Если заказ относится к regular flights, уменьшение ограничивается не только долгом клиента, но и неоплаченным хвостом туроператору:

- `client_package_debt = current_package_price - paid_amount_rub`
- `package_operator_paid = сумма operator_payments по service_type: 'order'`
- `operator_package_unpaid = current_package_price - package_operator_paid`
- `recalculatable_amount = min(client_package_debt, operator_package_unpaid)`

Для обычного случая:

- `recalculatable_amount = client_package_debt`

### Нижние границы

Новая цена пакета не должна быть ниже:

- цены по исходному курсу `source_rate`;
- цены по последнему примененному `exchange_rate` markup;
- цены, которая нарушает ограничение по допустимой части уменьшения.

Итоговая формула:

- `package_price_after = max(candidate_package_price, source_package_floor_price, markup_package_floor_price, recalculatable_floor_price)`

Если `package_price_after >= current_package_price`, пересчет не применяется.

### Раскладка по компонентам пакета

После определения новой полной цены пакета:

- считается `ratio = package_price_after / current_package_price`
- пропорционально пересчитываются `net_price_after` и `fuel_charge_after`

Это сделано для того, чтобы применять и откатывать не абстрактную сумму, а реальные поля пакета.

## 8. Жизненный цикл пересчета

### Preview

- В админке пользователь вводит `order_id`.
- Контроллер вызывает `Order::PriceRecalculations::Calculation`.
- Если заказ подходит, показываются:
  - исходный курс;
  - целевой курс;
  - цена пакета до;
  - цена пакета после.
- Если заказ не подходит, показывается текстовая причина отказа.

### Apply

- `Order::PriceRecalculations::Apply` повторно вызывает `Calculation`.
- Если заказ не подходит, создается запись `OrderPriceRecalculation` со статусом `rejected`.
- Если заказ подходит:
  - обновляются `package.net_price` и `package.fuel_charge`;
  - создается запись `OrderPriceRecalculation` со статусом `applied`;
  - сохраняются snapshot-поля и временные метки;
  - пишется `OrderLog`.

### Complete

- Если долг по пакету закрыт до истечения срока действия пересчета, запись переводится в `completed`.

### CancelExpired

- Если после `18:00` долг по пакету не закрыт, пакет откатывается к сохраненным значениям `net_price_before` и `fuel_charge_before`, а запись переводится в `canceled`.

## 9. Воркер и расписание

- Добавлен `OrderPriceRecalculationsCancelWorker`.
- Он запускается каждые `10` минут.
- Внутри он:
  - завершает активные пересчеты, если долг уже закрыт;
  - после `18:00` отменяет активные пересчеты, если долг остался.

Такой polling выбран, чтобы не держать отдельную джобу на каждый пересчет и при этом вовремя реагировать и на оплату, и на дедлайн.

## 10. Изменения в интерфейсе

В `Order Tools` теперь два независимых действия:

- старый пересчет вверх для валютного заказа;
- новый пересчет цены пакета вниз для валютного заказа.

Для нового сценария доступны:

- `Предпросмотр`
- `Применить`

Также добавлен журнал пересчетов в отдельной админке.

## 11. Файлы реализации

- `db/migrate/20260624115324_create_order_price_recalculations.rb`
- `app/models/order_price_recalculation.rb`
- `app/services/order/price_recalculations/acceptability.rb`
- `app/services/order/price_recalculations/calculation.rb`
- `app/services/order/price_recalculations/apply.rb`
- `app/services/order/price_recalculations/complete.rb`
- `app/services/order/price_recalculations/cancel_expired.rb`
- `app/admin/order_tools.rb`
- `app/views/admin/order_tools/_recalculate_price.html.haml`
- `app/admin/order_price_recalculations.rb`
- `app/workers/order_price_recalculations_cancel_worker.rb`
- `lib/schedule.rb`
- `config/locales/ru.yml`
- `spec/services/order/price_recalculations/apply_spec.rb`
- `spec/services/order/price_recalculations/calculation_spec.rb`
- `spec/workers/order_price_recalculations_cancel_worker_spec.rb`

## 12. Что дополнительно было упрощено по ходу реализации

- Убран отдельный `Preview`, потому что он только проксировал вызов `Calculation`.
- Убраны мелкие одноразовые сервисы расчета и резолвинга; их логика встроена в `Calculation`.
- Переводы и тексты в админке приведены к формулировке `цена пакета`, чтобы интерфейс звучал точнее.
- Для модели пересчетов добавлены русские названия атрибутов и статусов, чтобы журнал в админке читался без технических кодов.
