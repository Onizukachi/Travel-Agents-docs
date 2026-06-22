# Пересчет стоимости тура вниз для валютных направлений

## 1. Бизнес контекст для задачи

- Нужно дать сотрудникам контролируемый инструмент для ручного пересчета цены заказа вниз после частичной оплаты туристом.
- Пересчет нужен только для валютных заказов, где у клиента есть частичная оплата, а туроператору еще не выплачено 100% стоимости основного пакета.
- Бизнес-цель: снизить цену остатка к доплате при укреплении курса, не затрагивая фиксированные допуслуги и не создавая произвольных ручных корректировок.
- Инструмент должен быть прозрачен для сотрудника: до применения нужен превью-расчет с текущим курсом заказа, новым курсом и новой рублевой суммой.
- Инструмент должен быть аудируемым: история пересчетов, статусы, инициатор, экспорт.
- Инструмент должен быть ограниченным по правилам: не давать пересчитывать неподходящие заказы, не позволять многократные бессистемные пересчеты, автоматически отменять пересчет, если клиент не оплатил в день пересчета до 18:00.

## 2. Ключевые решения и обоснования

- Базироваться на существующем `Order::UpdatePriceService`, но не расширять его напрямую без выделения отдельного сервиса под decrease-flow.
  Обоснование: текущий сервис спроектирован под повышение цены, намеренно запрещает уменьшение и не покрывает превью, историю, окно действия и отмену.
- Логику eligibility, preview, apply и cancel разделить на отдельные операции.
  Обоснование: правила сложные и различаются по смыслу; отдельные сервисы проще тестировать и безопаснее развивать.
- Историю пересчетов хранить в отдельной таблице, а не только в `OrderLog`.
  Обоснование: нужны статусы, даты применения/отмены, курсы до/после, валюта, инициатор и экспорт в Excel с фильтруемыми структурированными полями.
- В админке делать двухшаговый flow: `preview` перед `apply`.
  Обоснование: это соответствует ТЗ и снижает риск случайного применения пересчета.
- Нижнюю границу пересчета считать не как отдельный минимальный курс с маркапом, а как минимальную допустимую рублевую цену после применения валютного маркапа.
  Обоснование: `markup_applications` хранит applied result в рублях (`base_price`, `markup_value`), а не прямой курс. Для первой итерации безопаснее считать floor по рублевой цене и брать итоговый нижний порог как максимум из цены по курсу бронирования/замены, цены по курсу последней оплаты и цены с учетом валютного маркапа.
- Для регулярных рейсов пересчитывать только остаток, не покрытый оплатой в ТО.
  Обоснование: это отдельное бизнес-правило, которое нельзя вывести из текущего механизма пересчета целиком по пакету.
- Дополнительные услуги, не относящиеся к ТО, исключать из базы пересчета.
  Обоснование: по ТЗ они фиксированные и не должны уменьшаться вместе с валютной частью тура.
- Автоотмену после 18:00 реализовывать фоновым процессом по активным пересчетам.
  Обоснование: правило зависит от времени и факта оплаты в тот же день; это надежнее оформляется как отдельная периодическая обработка активных записей истории.

## 2.1. Целевая модель данных

### Таблица `order_price_recalculations`

- `order_id`
  Заказ, к которому относится пересчет.
- `package_id`
  Пакет, цена которого реально менялась.
- `status`
  Состояние пересчета. Плановые значения: `applied`, `completed`, `canceled`, `rejected`.
- `source_currency`
  Исходная валюта заказа до пересчета.
- `source_rate`
  Исходный курс заказа до пересчета.
- `target_currency`
  Целевая валюта, по которой считается пересчет.
- `target_rate`
  Целевой курс пересчета.
- `price_before`
  Полная цена заказа до пересчета.
- `price_after`
  Полная цена заказа после пересчета.
- `recalculatable_amount`
  База пересчета, а не размер скидки. Это часть mutable basket, которая реально участвует в снижении.
- `net_price_before`
  Значение `package.net_price` до применения.
- `fuel_charge_before`
  Значение `package.fuel_charge` до применения.
- `net_price_after`
  Значение `package.net_price` после применения.
- `fuel_charge_after`
  Значение `package.fuel_charge` после применения.
- `mutable_extras_before`
  JSON-снапшот активных `order_extras` с `is_split = true` до применения.
- `mutable_extras_after`
  JSON-снапшот тех же допов после применения.
- `applied_at`
  Когда пересчет применили к заказу.
- `expires_at`
  Когда пересчет должен быть отменен, если клиент не закрыл долг. Для первой итерации: сегодня в `18:00`.
- `completed_at`
  Когда заказ успешно вышел из режима пересчета, потому что клиент закрыл долг.
- `canceled_at`
  Когда был выполнен откат.
- `rejection_reason`
  Причина отказа, если сотрудник нажал `apply`, а заказ не прошел правила.
- `cancel_reason`
  Причина отмены после применения.
- `initiator_id`
  Сотрудник, инициировавший пересчет.
- `created_at`, `updated_at`
  Служебные timestamps.

### Индексы

- Индекс на `order_id, status` для поиска активных/исторических пересчетов по заказу.
- Индекс на `package_id` для аудита и связи с пакетом.
- Индекс на `expires_at, status` для фоновой отмены.
- Индекс на `initiator_id` для журнала и экспорта.

### Формат `mutable_extras_before` / `mutable_extras_after`

```json
[
  { "order_extra_id": 11, "name": "Выбор рейса", "price": 3000.0 },
  { "order_extra_id": 15, "name": "Трансфер", "price": 4500.0 }
]
```

Эти поля нужны не только для истории, но и для точного отката цен сплитуемых допов после `18:00`.

## 2.2. Границы ответственности

### Что остается в существующих моделях

- `Order`
  Источник общей цены, оплат туриста, допов, признаков долга и состояния заказа.
- `Package`
  Источник валюты, курса и изменяемых полей `net_price` / `fuel_charge`.
- `OrderExtra`
  Источник допуслуг и признака `is_split`.
- `Payment`
  Источник оплат туриста и `operator_exchange_rate`.
- `OperatorPayment`
  Источник оплаты ТО.
- `Markup::Application`
  Источник applied валютного маркапа.
- `OrderLog`
  Дополнительный аудит для карточки заказа.

### За что отвечает новая модель `OrderPriceRecalculation`

- Хранит историю пересчетов вниз.
- Хранит активное состояние пересчета.
- Хранит snapshot полей для отката.
- Дает данные для Excel-экспорта.
- Защищает от одновременных активных пересчетов одного заказа.

### За что отвечают сервисы

- `Order::PriceRecalculations::Eligibility`
  Проверяет, можно ли запускать пересчет.
- `Order::PriceRecalculations::PackageReplacementDetector`
  Определяет, меняли ли пакет.
- `Order::PriceRecalculations::TargetRateResolver`
  Возвращает `source_currency`, `source_rate`, `target_currency`, `target_rate`.
- `Order::PriceRecalculations::MarkupFloorResolver`
  Возвращает нижний рублевый порог по валютному маркапу.
- `Order::PriceRecalculations::RecalculatableAmountCalculator`
  Считает mutable basket и `recalculatable_amount`.
- `Order::PriceRecalculations::Preview`
  Собирает данные для интерфейса перед применением.
- `Order::PriceRecalculations::Apply`
  Создает запись истории и меняет цены пакета/сплитуемых допов.
- `Order::PriceRecalculations::Complete`
  Закрывает активный пересчет как успешный, если долг погашен.
- `Order::PriceRecalculations::CancelExpired`
  Выполняет откат после `18:00`.

## 2.3. Правила определения значений

### Определение замены пакета

Основной источник:
- `OrderLog kind: :package_replace`, который пишет `Order::Operations::AssignPackageOperation`.

Fallback:
- `order.versions` с изменением `package_id`, потому что `Order` находится под `paper_trail`, а `package_id` не исключен из versioning.

Не использовать:
- `package_change_histories`, потому что это история change requests из PAPI, а не факт реальной замены.

### Определение исходных и целевых курса/валюты

Исходные значения:
- `source_currency = package.stat_currency_name`
- `source_rate = package.stat_exchange_rate`

Целевые значения:
- если пакет меняли:
  - `target_currency = package.stat_currency_name`
  - `target_rate = package.stat_exchange_rate`
- если пакет не меняли:
  - `target_currency = package.stat_currency_name`
  - `target_rate = last_paid_payment.operator_exchange_rate`

`last_paid_payment`:
- берем из `order.payments.paid_and_not_fully_refunded.where.not(operator_exchange_rate: nil).order(:created_at).last`

### Определение релевантного валютного маркапа

- Берем `Markup::Application` с `package_id = current package.id` и `markup_type = 'exchange_rate'`.
- Для первой итерации берем последнюю запись по `created_at DESC`.
- Если записи нет, валютный маркап не ограничивает цену.

### Определение fixed и mutable допов

Fixed extras:
- `order.order_extras.active.non_split`
- Это допы с `is_split = false`.
- Они не меняются и не входят в базу пересчета.

Mutable extras:
- `order.order_extras.active.is_split`
- Это допы с `is_split = true`.
- Они входят в mutable basket и изменяются вместе с пакетом.

## 2.4. Правила расчета

### Mutable basket

- `current_package_price = package.full_price`
- `current_split_extras_price = order.order_extras.active.is_split.sum(&:price)`
- `current_fixed_extras_price = order.order_extras.active.non_split.sum(&:price)`
- `current_mutable_price = current_package_price + current_split_extras_price`

### Оплата туриста

- Источник: `payments.paid_and_not_fully_refunded`
- Для первой итерации считаем, что оплата туриста сначала гасит mutable basket.
- `client_paid_mutable_amount = [total_client_paid_amount, current_mutable_price].min`
- `client_mutable_debt = [current_mutable_price - client_paid_mutable_amount, 0].max`

### Оплата ТО

- Пакетная часть: `operator_payments` по `service_type = 'order'`
- Допчасть: `operator_payments` по `service_type = 'order_extra'`, если связанный `order_extra.active.is_split`
- `operator_paid_mutable_amount = package_operator_paid + split_extras_operator_paid`
- `operator_mutable_unpaid = [current_mutable_price - operator_paid_mutable_amount, 0].max`

### `recalculatable_amount`

Обычный случай:
- `recalculatable_amount = client_mutable_debt`

Regular flights:
- `recalculatable_amount = [client_mutable_debt, operator_mutable_unpaid].min`

Смысл:
- для обычных рейсов пересчитываем долг клиента по mutable basket;
- для regular flights дополнительно ограничиваем пересчет хвостом, который еще не оплачен ТО.

### Кандидатная новая цена пакета

Если у всех релевантных платежей есть `operator_exchange_rate`:
- `paid_amount_rub = order.available_certificate_amount + payments.sum(&:paid_amount)`
- `paid_amount_in_foreign_currency = certificate_part_in_foreign_currency + payments.sum(&:paid_amount_in_foreign_currency)`
- `debt_in_foreign_currency = package.full_price_in_foreign_currency - paid_amount_in_foreign_currency`
- `candidate_package_price = target_rate * debt_in_foreign_currency + paid_amount_rub`

Если не у всех платежей есть курс:
- `candidate_package_price = target_rate * (package.stat_net_price_usd + package.stat_fuel_charge_usd)`

### Пропорция пересчета mutable basket

- `recalc_ratio = candidate_package_price / current_package_price`
- Для down-flow `recalc_ratio` должен быть `> 0` и `<= 1`
- `candidate_mutable_price = current_mutable_price * recalc_ratio`

### Нижний рублевый порог

Порог по исходному курсу:
- считаем `source_package_floor_price` той же формулой, что и `candidate_package_price`, но с `source_rate`
- `source_ratio = source_package_floor_price / current_package_price`
- `source_mutable_floor_price = current_mutable_price * source_ratio`

Порог по валютному маркапу:
- `markup_package_floor_price = markup_application.base_price + markup_application.markup_value`
- `markup_ratio = markup_package_floor_price / current_package_price`
- `markup_mutable_floor_price = current_mutable_price * markup_ratio`

Финальная mutable price:
- `final_mutable_price = [candidate_mutable_price, source_mutable_floor_price, markup_mutable_floor_price].max`

Финальная полная цена заказа:
- `price_after = final_mutable_price + current_fixed_extras_price - applicable_discount`

### Что именно меняется при apply

Изменяются:
- `package.net_price`
- `package.fuel_charge`
- `order.order_extras.active.is_split.price`

Не изменяются:
- `package.stat_exchange_rate`
- `package.stat_currency_name`
- `order.order_extras.active.non_split.price`
- платежи туриста
- платежи ТО

Распределение:
- `package.net_price`, `package.fuel_charge` и каждый `active is_split extra` уменьшаются пропорционально общей корректировке mutable basket.
- Последний проход распределения должен компенсировать округления, чтобы сумма после пересчета совпала с `final_mutable_price`.

## 2.5. Flow работы

### Preview

1. Сотрудник вводит `order_id`.
2. Система находит заказ и пакет.
3. `Eligibility` проверяет применимость.
4. `PackageReplacementDetector` определяет, меняли ли пакет.
5. `TargetRateResolver` возвращает `source_*` и `target_*`.
6. `RecalculatableAmountCalculator` считает mutable basket, fixed extras и `recalculatable_amount`.
7. Сервис preview считает:
   - `candidate_package_price`
   - `candidate_mutable_price`
   - порог по `source_rate`
   - порог по `exchange_rate markup`
   - `final_mutable_price`
   - `price_after`
8. Интерфейс показывает:
   - текущий курс заказа
   - целевой курс пересчета
   - цену до/после
   - базу пересчета
   - какие fixed extras исключены
   - причину отказа, если заказ не подходит

### Apply

1. Повторно выполняются все проверки preview.
2. Если заказ не подходит, создается `OrderPriceRecalculation` со статусом `rejected`.
3. Если подходит:
   - сохраняется snapshot package и split extras в `*_before`
   - рассчитываются `*_after`
   - обновляются package и split extras
   - создается запись `OrderPriceRecalculation` со статусом `applied`
   - ставятся `applied_at = now` и `expires_at = today 18:00`
   - создается `OrderLog`

### Complete

- Если до `18:00` у заказа больше нет долга, активный пересчет переводится в `completed`.
- Откат после этого не выполняется.

### CancelExpired

1. После `18:00` воркер ищет активные записи со статусом `applied`.
2. Если заказ не стал без долга:
   - `package.net_price` и `package.fuel_charge` возвращаются из `*_before`
   - цены split extras возвращаются из `mutable_extras_before`
   - запись переводится в `canceled`
   - пишутся `canceled_at`, `cancel_reason`
   - создается `OrderLog`
3. После автоотмены новый пересчет в тот же день разрешен.

## 3. Структурированный список задач

- [x] Шаг 1: Изучить текущий ручной пересчет вверх в `app/admin/order_tools.rb` и `app/services/order/update_price_service.rb`.
- [x] Шаг 2: Уточнить у бизнеса правила источника курса, fixed extras, regular flights, многократных пересчетов и поведения после автоотмены.
- [x] Шаг 3: Спроектировать доменную модель истории пересчетов и состояния процесса.
- [x] Шаг 4: Спроектировать сервисы eligibility, preview, apply, cancel и источник курса для каждого сценария.
- [x] Шаг 5: Спроектировать изменения в ActiveAdmin: форма, preview-экран, подтверждение применения, журнал, экспорт.
- [x] Шаг 6: Спроектировать фоновую отмену после 18:00 и правило проверки оплаты в день пересчета.
- [ ] Шаг 7: Реализовать миграцию и модель `OrderPriceRecalculation`.
- [ ] Шаг 8: Реализовать сервисы `Eligibility`, `PackageReplacementDetector`, `TargetRateResolver`, `MarkupFloorResolver`, `RecalculatableAmountCalculator`, `Preview`, `Apply`, `Complete`, `CancelExpired`.
- [ ] Шаг 9: Реализовать обновление package и split extras с сохранением snapshots и откатом.
- [ ] Шаг 10: Реализовать admin flow в `app/admin/order_tools.rb` и partial для preview/apply.
- [ ] Шаг 11: Добавить отдельный ActiveAdmin resource для журнала пересчетов и Excel-экспорт.
- [ ] Шаг 12: Добавить воркер автоотмены и исключение активных down-recalculation заказов из `OrdersWithDebtUpdatePriceWorker`.
- [ ] Шаг 13: После реализации прогнать релевантные проверки и затем синхронизировать `.agents` / `AGENTS.md` в `../agents_md`.

## 3.1. Чеклист по файлам

- Миграция:
  - создать `db/migrate/*_create_order_price_recalculations.rb`
- Модель:
  - добавить `app/models/order_price_recalculation.rb`
  - добавить ассоциации в `app/models/order.rb`
- Сервисы:
  - `app/services/order/price_recalculations/eligibility.rb`
  - `app/services/order/price_recalculations/package_replacement_detector.rb`
  - `app/services/order/price_recalculations/target_rate_resolver.rb`
  - `app/services/order/price_recalculations/markup_floor_resolver.rb`
  - `app/services/order/price_recalculations/recalculatable_amount_calculator.rb`
  - `app/services/order/price_recalculations/preview.rb`
  - `app/services/order/price_recalculations/apply.rb`
  - `app/services/order/price_recalculations/complete.rb`
  - `app/services/order/price_recalculations/cancel_expired.rb`
- Админка:
  - обновить `app/admin/order_tools.rb`
  - добавить/обновить partial в `app/views/admin/order_tools/`
  - добавить `app/admin/order_price_recalculations.rb`
- Воркеры:
  - добавить `app/workers/order_price_recalculations_cancel_worker.rb`
  - доработать `app/workers/orders_with_debt_update_price_worker.rb`
- Локализация:
  - добавить тексты/статусы в `config/locales/ru.yml`

## 4. Заметки для восстановления сессии

- Уже изучен текущий flow ручного пересчета вверх:
  - `app/admin/order_tools.rb` добавляет простой `page_action :recalculate_price` без превью.
  - `app/views/admin/order_tools/_recalculate_price.html.haml` содержит только поле `order_id` и submit.
  - `Order::UpdatePriceService` пересчитывает стоимость только вверх и возвращает `false`, если новая цена меньше текущей.
- Текущий сервис умеет считать неоплаченную валютную часть, если у всех платежей есть `operator_exchange_rate`; иначе пересчитывает весь валютный пакет.
- Для долга по базовому пакету используется `Order#has_debt_for_base_package?`, что уже близко к новому бизнес-правилу, но не учитывает отдельно оплату ТО, fixed extras, окно 04:00-18:00 и отмену.
- В репозитории уже есть инфраструктура для `xlsx`-экспорта через `Axlsx`, значит историю пересчетов можно встроить в существующий стек без новых gem.
- Уже подтверждены пользователем бизнес-решения:
  - курс на момент бронирования/замены пакета пока берем из `package.stat_exchange_rate`;
  - валютный маркап искать через `markup_applications` и `markup_rules.markup_type = 'exchange_rate'`;
  - нижнюю границу применять по рублевой цене, а не пытаться восстанавливать отдельный минимальный курс из `markup_applications`; для первой итерации брать последнюю запись `markup_application` типа `exchange_rate` по текущему `package_id`;
  - из пересчета исключаем не-сплитуемые допуслуги (`order_extras.is_split = false`);
  - сплитуемые допуслуги участвуют в корректировке цены и должны сохраняться в истории отдельным snapshot;
  - успешным завершением пересчета считается полная оплата остатка после пересчета, когда у заказа больше нет долга;
  - для регулярных рейсов база пересчета ограничивается частью, не оплаченной ТО;
  - одновременно допускается только один активный пересчет на заказ, но после автоотмены повторный запуск в тот же день разрешен;
  - доступ можно строить на текущем permission `admin_order_tools :recalculate_price`.
  - целевые курс и валюта определяются так:
    - если пакет меняли, берем `target_currency` и `target_rate` из текущего пакета;
    - если пакет не меняли, берем `target_rate` из последнего оплаченного платежа, а `target_currency` из текущего пакета.
- Дополнительно исследовано по коду:
  - оплата туриста в валюте уже снапшотится в `payments.operator_exchange_rate`;
  - частичная/полная оплата ТО берется из `operator_payments`, есть методы `operator_total_paid`, `operator_fully_paid?`, `has_operator_debt?`;
  - `markup_applications` уже хранят примененный markup для пакета, но пока неочевидно, как из `base_price` и `markup_value` корректно получить минимальный допустимый курс;
  - есть фоновый `OrdersWithDebtUpdatePriceWorker`, который ежедневно повышает цену по заказам с долгом; его нужно учесть, чтобы он не конфликтовал с новым down-flow.
  - факт замены пакета надежнее всего определять по `OrderLog kind: :package_replace`, который пишет `Order::Operations::AssignPackageOperation`;
  - как fallback можно использовать `order.versions` с изменением `package_id`, потому что `Order` находится под `paper_trail`, а `package_id` не исключен из versioning;
  - `package_change_histories` не подходят для этой задачи, потому что они фиксируют change requests из PAPI, а не фактическую замену пакета в заказе.
- Текущее согласованное ТЗ:
  - новая таблица `order_price_recalculations` хранит историю, активное состояние и snapshots package/split extras;
  - fixed extras (`is_split = false`) не входят в mutable basket и не изменяются;
  - split extras (`is_split = true`) входят в mutable basket и корректируются пропорционально вместе с пакетом;
  - `recalculatable_amount` считается от mutable basket, а для regular flights ограничивается также хвостом, не оплаченным ТО;
  - `target_rate` и `target_currency` берутся из текущего пакета после замены пакета, иначе `target_rate` берется из последнего платежа;
  - lower bound считается по рублевой цене: максимум из candidate price, порога по `source_rate` и порога по applied `exchange_rate` markup;
  - при `apply` создается запись `applied`, а при неуспешном `apply` создается запись `rejected`;
  - после `18:00` воркер откатывает `package.net_price`, `package.fuel_charge` и split extras из snapshot, если долг не погашен;
  - после автоотмены разрешен новый пересчет в тот же день.
- Следующая точка продолжения:
  - перейти к реализации миграции, модели и сервисов из раздела `3.1`;
  - первым делом реализовать `OrderPriceRecalculation`, `PackageReplacementDetector`, `TargetRateResolver` и `RecalculatableAmountCalculator`, чтобы зафиксировать доменную модель и базовые формулы до интеграции в админку.
