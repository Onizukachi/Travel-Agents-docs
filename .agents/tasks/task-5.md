# Пересчет стоимости пакета вниз для валютных заказов

## 1. Цель

- Дать сотрудникам контролируемый инструмент для ручного уменьшения стоимости валютного заказа после частичной оплаты.
- Пересчитывается только стоимость пакета.
- Допуслуги не участвуют в расчете и не изменяются.
- Пересчет действует до `18:00` дня применения: если клиент не закрыл долг, изменения по пакету откатываются.

## 2. Основные правила

- Применимо только к валютным заказам.
- Должна быть частичная оплата клиента.
- У заказа должен оставаться долг по пакету.
- Одновременно у заказа может быть только один активный down-recalculation.
- После автоотмены можно запускать новый пересчет в тот же день.
- Логика учета оплат клиента остается такой же, как в [update_price_service.rb](/Users/hikaru/projects/work/leveltravel/app/services/order/update_price_service.rb):
  - оплаты и сертификат учитываются на уровне всего заказа;
  - отдельно выделять, какая часть оплаты пришлась на допуслуги, не пытаемся;
  - это осознанное упрощение.

## 3. Что меняется и что не меняется

### Меняется

- `package.net_price`
- `package.fuel_charge`

### Не меняется

- `order_extras`
- `package.stat_exchange_rate`
- `package.stat_currency_name`
- платежи клиента
- платежи туроператору

## 4. Модель данных

### Таблица `order_price_recalculations`

- `order_id`
  Заказ, для которого выполнялся пересчет.
- `package_id`
  Пакет, чья цена менялась.
- `status`
  Статус пересчета: `applied`, `completed`, `canceled`, `rejected`.
- `source_currency`
  Валюта пакета на момент пересчета.
- `source_rate`
  Курс пакета на момент пересчета.
- `target_currency`
  Валюта, по которой считается новый пакет.
- `target_rate`
  Целевой курс пересчета.
- `package_price_before`
  Полная цена пакета до пересчета (`net_price + fuel_charge`).
- `package_price_after`
  Полная цена пакета после пересчета.
- `recalculatable_amount`
  Часть стоимости пакета, которую разрешено уменьшать.
- `net_price_before`
  `package.net_price` до применения.
- `fuel_charge_before`
  `package.fuel_charge` до применения.
- `net_price_after`
  `package.net_price` после применения.
- `fuel_charge_after`
  `package.fuel_charge` после применения.
- `applied_at`
  Когда пересчет был применен.
- `expires_at`
  Дедлайн автоотмены. Для первой итерации: сегодня в `18:00`.
- `completed_at`
  Когда пересчет успешно завершился оплатой долга.
- `canceled_at`
  Когда был выполнен откат.
- `rejection_reason`
  Причина отказа при `apply`.
- `cancel_reason`
  Причина отката.
- `initiator_id`
  Сотрудник, который запустил пересчет.
- `created_at`, `updated_at`
  Служебные поля.

### Зачем нужны эти поля

- `source_*`, `target_*`
  Чтобы в preview, журнале и экспорте было видно, какой курс был у заказа и какой курс применили.
- `package_price_before`, `package_price_after`
  Чтобы быстро видеть итог изменения пакета без повторного расчета.
- `recalculatable_amount`
  Чтобы было видно, какая часть пакета вообще подпадала под уценку.
- `net_price_*`, `fuel_charge_*`
  Это фактический snapshot для точного применения и отката.
- `status` и временные поля
  Чтобы поддерживать жизненный цикл: применен, завершен, отменен, отклонен.
- `rejection_reason`, `cancel_reason`
  Для аудита и понятного объяснения сотруднику.

### Индексы

- `index(order_id, status)` для активных и исторических пересчетов по заказу.
- `index(package_id)` для аудита по пакету.
- `index(expires_at, status)` для фоновой автоотмены.
- `index(initiator_id)` для журнала и выгрузок.

## 5. Модели

- `OrderPriceRecalculation`
  История пересчетов вниз, активное состояние пересчета и snapshot package price components.
- `Order`
  Источник общей оплаты клиента, долга, платежей ТО и связи с пересчетами.
- `Package`
  Источник валюты, курса и изменяемых полей `net_price` / `fuel_charge`.
- `Payment`
  Источник оплат клиента и `operator_exchange_rate`.
- `OperatorPayment`
  Источник оплат ТО для ограничения пересчета по regular flights.
- `Markup::Application`
  Источник нижней границы по applied exchange-rate markup.
- `OrderLog`
  Дополнительный аудит в карточке заказа.

## 6. Сервисы

- `Order::PriceRecalculations::Eligibility`
  Проверяет, можно ли запускать пересчет.
- `Order::PriceRecalculations::PackageReplacementDetector`
  Определяет, меняли ли пакет.
- `Order::PriceRecalculations::TargetRateResolver`
  Возвращает `source_currency`, `source_rate`, `target_currency`, `target_rate`.
- `Order::PriceRecalculations::MarkupFloorResolver`
  Возвращает минимально допустимую цену пакета по applied валютному markup.
- `Order::PriceRecalculations::RecalculatableAmountCalculator`
  Считает, какая часть пакета может быть уменьшена.
- `Order::PriceRecalculations::Preview`
  Собирает расчет до применения.
- `Order::PriceRecalculations::Apply`
  Повторно проверяет условия, сохраняет историю и обновляет пакет.
- `Order::PriceRecalculations::Complete`
  Закрывает активный пересчет как успешный после погашения долга.
- `Order::PriceRecalculations::CancelExpired`
  Откатывает пакет после `18:00`, если долг не закрыт.

## 7. Правила определения значений

### Определение замены пакета

Основной источник:
- `OrderLog kind: :package_replace`

Fallback:
- `order.versions` с изменением `package_id`

### Исходные и целевые курс/валюта

Исходные значения:
- `source_currency = package.stat_currency_name`
- `source_rate = package.stat_exchange_rate`

Целевые значения:
- `target_currency = package.stat_currency_name`
- если пакет меняли:
  - `target_rate = package.stat_exchange_rate`
- если пакет не меняли:
  - `target_rate = last_paid_payment.operator_exchange_rate`

`last_paid_payment`:
- `order.payments.paid_and_not_fully_refunded.where.not(operator_exchange_rate: nil).order(:created_at).last`

Если `target_rate` не найден или `target_rate >= source_rate`, пересчет вниз неприменим.

### Релевантный валютный markup

- Берем `Markup::Application` по текущему `package_id`.
- Используем только markup типа `exchange_rate`.
- Для первой итерации берем последнюю запись по `created_at DESC`.
- Если записи нет, markup не ограничивает снижение.

## 8. Формулы расчета

### 8.1. Базовая цена пакета

- `current_package_price = package.full_price`

### 8.2. Учет оплат клиента

Как в `Order::UpdatePriceService`:

- `paid_amount_rub = order.available_certificate_amount + order.payments.paid_and_not_fully_refunded.sum(&:paid_amount)`
- `paid_amount_in_foreign_currency = certificate_part_in_foreign_currency + order.payments.paid_and_not_fully_refunded.sum(&:paid_amount_in_foreign_currency)`

Где:
- `certificate_part_in_foreign_currency = order.available_certificate_amount / payments.first.operator_exchange_rate`
- используется только если у всех релевантных платежей есть `operator_exchange_rate`

### 8.3. Кандидатная новая цена пакета

Если у всех релевантных платежей есть `operator_exchange_rate`:

- `debt_in_foreign_currency = package.full_price_in_foreign_currency - paid_amount_in_foreign_currency`
- `candidate_package_price = target_rate * debt_in_foreign_currency + paid_amount_rub`

Если не у всех платежей есть `operator_exchange_rate`:

- `candidate_package_price = target_rate * package.full_price_in_foreign_currency`

### 8.4. `recalculatable_amount`

Обычный случай:

- `client_package_debt = [current_package_price - paid_amount_rub, 0].max`
- `recalculatable_amount = client_package_debt`

Regular flights:

- `package_operator_paid = operator_payments.where(service_type: 'order').sum(&:paid_amount)`
- `operator_package_unpaid = [current_package_price - package_operator_paid, 0].max`
- `recalculatable_amount = [client_package_debt, operator_package_unpaid].min`

### 8.5. Ограничение снижения

Пересчет не должен уменьшать пакет ниже:

- цены по `source_rate`
- цены по applied exchange-rate markup

Порог по исходному курсу:

- если у всех платежей есть курс:
  - `source_package_floor_price = source_rate * debt_in_foreign_currency + paid_amount_rub`
- иначе:
  - `source_package_floor_price = source_rate * package.full_price_in_foreign_currency`

Порог по markup:

- `markup_package_floor_price = markup_application.base_price + markup_application.markup_value`

Финальная цена пакета:

- `package_price_after = [candidate_package_price, source_package_floor_price, markup_package_floor_price].max`

Если `package_price_after >= current_package_price`, пересчет вниз не применяется.

### 8.6. Распределение по компонентам пакета

- `ratio = package_price_after / current_package_price`
- `net_price_after = package.net_price * ratio`
- `fuel_charge_after = package.fuel_charge * ratio`

Последний шаг распределения должен компенсировать округление так, чтобы:

- `net_price_after + fuel_charge_after == package_price_after`

## 9. Flow

### Preview

1. Сотрудник вводит `order_id`.
2. `Eligibility` проверяет применимость.
3. `PackageReplacementDetector` определяет факт замены пакета.
4. `TargetRateResolver` возвращает курсы.
5. `RecalculatableAmountCalculator` считает допустимую базу пересчета.
6. `MarkupFloorResolver` возвращает нижнюю границу.
7. `Preview` показывает:
   - курс заказа;
   - целевой курс;
   - цену пакета до/после;
   - сумму, которая может быть уменьшена;
   - причину отказа, если заказ не подходит.

### Apply

1. Повторяем все проверки preview.
2. Если заказ не подходит, создаем `OrderPriceRecalculation` со статусом `rejected`.
3. Если подходит:
   - сохраняем `net_price_before`, `fuel_charge_before`, `package_price_before`;
   - рассчитываем `*_after`;
   - обновляем `package.net_price` и `package.fuel_charge`;
   - создаем `OrderPriceRecalculation` со статусом `applied`;
   - ставим `applied_at` и `expires_at`;
   - пишем `OrderLog`.

### Complete

- Если до `18:00` заказ перестал иметь долг, активный пересчет переводится в `completed`.

### CancelExpired

1. После `18:00` воркер ищет активные записи `applied`.
2. Если долг не закрыт:
   - `package.net_price` и `package.fuel_charge` откатываются из snapshot;
   - запись переводится в `canceled`;
   - проставляются `canceled_at`, `cancel_reason`;
   - создается `OrderLog`.

## 10. Примеры расчета

### Кейс 1. Обычный валютный заказ, все платежи с курсом

- `package.full_price = 120_000`
- `package.full_price_in_foreign_currency = 1_200`
- `source_rate = 100`
- `target_rate = 90`
- клиент оплатил `60_000`
- в валюте это `600`

Расчет:

- `debt_in_foreign_currency = 1_200 - 600 = 600`
- `candidate_package_price = 90 * 600 + 60_000 = 114_000`

Итог:

- `package_price_before = 120_000`
- `package_price_after = 114_000`
- снижение: `6_000`

### Кейс 2. Не у всех платежей есть курс

- `package.full_price = 120_000`
- `package.full_price_in_foreign_currency = 1_200`
- `source_rate = 100`
- `target_rate = 90`

Расчет:

- `candidate_package_price = 90 * 1_200 = 108_000`

Итог:

- пересчитываем весь валютный пакет целиком;
- платежи в валютной части отдельно не раскладываем.

### Кейс 3. Есть допуслуги, но они не участвуют

- пакет: `120_000`
- допуслуги: `15_000`
- полная цена заказа: `135_000`
- после пересчета пакет стал `114_000`

Итог:

- пакет меняется: `120_000 -> 114_000`
- допуслуги остаются `15_000`
- новая полная цена заказа становится `129_000`

### Кейс 4. Regular flight, ТО еще не оплачен хвост

- `current_package_price = 120_000`
- долг клиента по пакету: `50_000`
- ТО оплачено `90_000`
- `operator_package_unpaid = 30_000`

Итог:

- `recalculatable_amount = min(50_000, 30_000) = 30_000`
- сильнее уменьшать нельзя, даже если курс позволяет.

### Кейс 5. Курс дает слишком сильное снижение, срабатывает floor

- `current_package_price = 120_000`
- `candidate_package_price = 105_000`
- `source_package_floor_price = 110_000`
- `markup_package_floor_price = 112_000`

Итог:

- `package_price_after = max(105_000, 110_000, 112_000) = 112_000`

## 11. Файлы реализации

- `db/migrate/*_create_order_price_recalculations.rb`
- `app/models/order_price_recalculation.rb`
- `app/services/order/price_recalculations/eligibility.rb`
- `app/services/order/price_recalculations/package_replacement_detector.rb`
- `app/services/order/price_recalculations/target_rate_resolver.rb`
- `app/services/order/price_recalculations/markup_floor_resolver.rb`
- `app/services/order/price_recalculations/recalculatable_amount_calculator.rb`
- `app/services/order/price_recalculations/preview.rb`
- `app/services/order/price_recalculations/apply.rb`
- `app/services/order/price_recalculations/complete.rb`
- `app/services/order/price_recalculations/cancel_expired.rb`
- `app/admin/order_tools.rb`
- `app/views/admin/order_tools/*`
- `app/admin/order_price_recalculations.rb`
- `app/workers/order_price_recalculations_cancel_worker.rb`
- `app/workers/orders_with_debt_update_price_worker.rb`
- `config/locales/ru.yml`

## 12. Следующий шаг

- Реализовать миграцию и модель `OrderPriceRecalculation`.
- Затем собрать package-only preview и apply поверх текущей логики `Order::UpdatePriceService`.
