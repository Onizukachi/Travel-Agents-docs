# Задача: развести WL-коммуникации и депозитную механику субагентов

## 1. Бизнес контекст для задачи

Есть два разных типа партнёров-субагентов:
- API-first-line: фронт, оплаты и клиентские коммуникации на стороне партнёра; у нас используется депозитный счёт для бронирований.
- WL-hooks: фронт и оплаты на стороне LT, но коммуникации по изменениям заказов у партнёра; от LT остаются только авторизационные SMS.

Сейчас у партнёра `MOSGORTUR_ID` одновременно включены `is_outer_first_line` и `is_wl_hooks`, что приводит к неоднозначному поведению.

Новые требования:
- при `is_wl_hooks = true` отключать транзакционные письма и SMS по заказам в модуле `Notifier` (`app/services/notifier.rb`);
- ввести у партнёра флаг `deposit_allowed` (по умолчанию `false`) и завязать депозитную механику (`DepositTransaction` и проверку остатка депозита) на него;
- депозит включается только для партнёров Озон и старой Альфы (`CRPO`, `CRPO_PP`);
- `ExternalNotification` по изменениям заказа для субагентов должен продолжить работать.

## 2. Ключевые решения и обоснования

- Решение 1: добавить `partners.deposit_allowed:boolean, default: false, null: false`.
  Обоснование: явный бизнес-флаг на партнёре проще и безопаснее текущих косвенных условий через `subagent_wl?` и client_id.

- Решение 2: перевести условие депозитной логики на `partner.deposit_allowed?`.
  Обоснование: единая точка включения для:
  - создания `DepositTransaction` на capture/refund;
  - проверки достаточности депозита в subagent hooks.

- Решение 3: в `Notifier` мьютить именно транзакционные order-уведомления для `is_wl_hooks`.
  Обоснование: требование касается транзакционных email/SMS по заказам, а не всех видов коммуникаций.

- Решение 4: не менять pipeline `ExternalNotification`.
  Обоснование: по требованию уведомления субагентам об изменениях заказов должны сохраниться.

- Решение 5: покрыть изменения точечными RSpec-тестами на `Notifier` и subagent hooks.
  Обоснование: эти точки напрямую отвечают за бизнес-эффект и регрессы.

## 3. Структурированный список задач

- [ ] Шаг 1: Подтвердить точный список партнёров для включения `deposit_allowed` (идентификаторы/критерий поиска для `OZON`, `CRPO`, `CRPO_PP`).
- [ ] Шаг 2: Добавить миграцию `deposit_allowed` в `partners` со значением по умолчанию `false` и обновить `db/schema.rb`.
- [ ] Шаг 3: Добавить `deposit_allowed` в модель `Partner` (mass-assignment/админ-форма/показ в админке) в существующем стиле проекта.
- [ ] Шаг 4: Изменить депозитную проверку в `Papi::V3::OrderSubagentMethods#process_payment`: проверять остаток депозита только если `order.partner.deposit_allowed?`.
- [ ] Шаг 5: Изменить депозитную механику в `Payment`/`Subagent::PaymentProcessor`: создание `DepositTransaction` на capture/refund только при `deposit_allowed`.
- [ ] Шаг 6: Обновить `Notifier`: при `is_wl_hooks = true` отключить транзакционные order email/SMS, не затрагивая `gasket_sms` и `ExternalNotification`.
- [ ] Шаг 7: Добавить/обновить RSpec:
  - `spec/services/notifier_spec.rb` (кейс `is_wl_hooks`);
  - `spec/controllers/papi/v3/orders_controller/hooks_spec.rb` (ветка с выключенным депозитом).
- [ ] Шаг 8: Выполнить точечный прогон тестов по изменённым файлам и зафиксировать результаты.

## 4. Заметки для восстановления сессии

- Текущее состояние проекта:
  - В `app/services/notifier.rb` уже есть сложная логика mute с отдельным исключением для `MOSGORTUR_ID`.
  - В `app/models/payment.rb` метод `deposit_allowed?` сейчас не использует отдельный флаг, а опирается на `subagent_wl?` и client_id.
  - В `app/controllers/concerns/papi/v3/order_subagent_methods.rb` проверка остатка депозита выполняется без флага и всегда для subagent notify `status=paid`.
  - В `app/services/subagent/payment_processor.rb` и `Payment#mark_captured!/#outer_partner_refund!` создание `DepositTransaction` уже условное, но через текущую реализацию `deposit_allowed?`.
  - В `spec/controllers/papi/v3/orders_controller/hooks_spec.rb` тесты предполагают обязательную депозитную проверку и создание транзакций.

- Критически важные принятые решения:
  - Нужен явный флаг `deposit_allowed` на `Partner`.
  - Логику депозита и проверку остатка переводим на новый флаг.
  - `ExternalNotification` не меняем.
  - Авторизационные SMS (`gasket_sms`) не блокируем.

- Точка продолжения при обрыве сессии:
  - Сначала закрыть открытые вопросы по точной трактовке "транзакционных писем" и по идентификации партнёров `OZON/CRPO/CRPO_PP`.
  - Затем начать с миграции `deposit_allowed` + wiring в `Partner`, после этого обновить бизнес-условия в `Notifier`, `OrderSubagentMethods`, `Payment`, `Subagent::PaymentProcessor` и тесты.
