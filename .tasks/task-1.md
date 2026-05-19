# Селектор статусов операторского платежа

## 1. Бизнес-контекст задачи

В менеджерке заказа в блоке операторских платежей менеджеры фиксируют состояние документов и переносов, связанных с возвратами у туроператора. Сейчас есть только отдельное поле «Заявление на возврат ТО»: чекбокс и дата. Нужно расширить этот интерфейс до селектора статусов, чтобы в том же месте можно было управлять тремя независимыми состояниями операторского платежа:

- «Заявление на возврат ТО» через существующие поля `have_written_statement` и `written_statement_at`.
- «Заявление на перенос» через новые поля `have_written_transfer` и `written_transfer_at`.
- «Перенесено» через новые поля `has_transferred` и `transferred_at`.

Несколько состояний могут быть отмечены одновременно. UI редактирует только состояние, выбранное в селекторе.

## 2. Ключевые решения и обоснование

- Селектор отображать всегда в строке операторского платежа. Текущий блок фактически всегда присутствует в шаблоне, но скрывается CSS-классом: `show_written_statement` дает `fill`, если `amount_awaiting_refund > 0` или `written_statement_at` не `null`. Новая задача требует работы не только с возвратом, поэтому завязка на `amount_awaiting_refund` больше не подходит.
- Порядок выбора значения по умолчанию: «Заявление на возврат ТО», затем «Заявление на перенос», затем «Перенесено». Если несколько чекбоксов отмечены, выбирается первое отмеченное значение в этом порядке; если не отмечено ничего, выбирается «Заявление на возврат ТО».
- Использовать корректные английские имена новых полей: `has_transferred` и `transferred_at`, а не `has_transfered` и `transfered_at`.
- Boolean-поля в базе: `default: false, null: false`. Date/time-поля без default, `null` разрешен.
- Поведение чекбокса оставить идентичным текущему: при включении чекбокса соответствующая дата выставляется в текущее время, после этого пользователь может изменить дату вручную.
- Не выключать другие состояния при включении выбранного состояния: бизнес-состояния независимы и могут быть отмечены одновременно.
- Тесты на этом этапе не писать по прямому указанию.

## 3. Структурированный список задач

- [x] Шаг 1: Добавить миграцию для `operator_payments` с полями `have_written_transfer`, `written_transfer_at`, `has_transferred`, `transferred_at`.
- [x] Шаг 2: Применить миграцию через `bundle exec rails db:migrate` и проверить, что `db/schema.rb` содержит только релевантные изменения.
- [x] Шаг 3: Добавить новые поля в `OperatorPayment`: `attr_accessible`, сериализацию и локализацию атрибутов.
- [x] Шаг 4: Обновить фронтовую модель `operator_payment.coffee`: новые атрибуты, список статусов селектора, выбор статуса по умолчанию, computed-значения для активного чекбокса и даты.
- [x] Шаг 5: Обновить шаблон `_operator_payment.hjs`: оставить чекбокс и date picker, добавить селектор между ними, привязать значения к выбранному статусу и показывать блок всегда.
- [x] Шаг 6: Обновить контроллер `Manager::OperatorPaymentsController`, чтобы изменение чекбокса и даты принимало выбранный тип статуса и обновляло только соответствующие поля.
- [x] Шаг 7: Обновить CoffeeScript controller actions, чтобы они передавали выбранный статус и скрывали нужную кнопку применения без глобального `id="apply_btn"` конфликта.
- [x] Шаг 8: Скорректировать SCSS для нового селектора и сохранения текущего компактного расположения.
- [x] Шаг 9: Выполнить минимальную статическую проверку измененных файлов и, при возможности, открыть URL проверки в менеджерке для ручной проверки поведения.
- [x] Шаг 10: Синхронизировать `.tasks/task-1.md` в `../agents_md/`, выполнить там `git pull`, затем commit и push зеркала согласно правилам проекта.

## 4. Заметки для восстановления сессии

Текущее состояние проекта: задача начинается в `/home/hikaru/projects/work/leveltravel`. До реализации изучены файлы `app/assets/javascripts/manager/templates/payments/_operator_payment.hjs`, `app/assets/javascripts/manager/models/operator_payment.coffee`, `app/assets/javascripts/manager/controllers/order_operator_payments_controller.coffee`, `app/controllers/manager/operator_payments_controller.rb`, `app/serializers/operator_payment_serializer.rb`, `app/models/operator_payment.rb`, `db/schema.rb`, `config/locales/ru.yml`, `app/assets/stylesheets/manager/order.scss`.

Уже выполненная работа: выяснено, что старый блок «Заявление на возврат ТО» находится в шаблоне всегда, но скрыт CSS (`.written_statement { display: none; }`). Он становится видимым только когда computed-свойство `show_written_statement` добавляет класс `fill`. Это свойство возвращало true при `amount_awaiting_refund > 0` или `written_statement_at != null`. Чекбокс `have_written_statement` сам по себе не управлял видимостью блока. Добавлены миграция, поля модели/serializer/локализации, backend-обработка выбранного статуса, фронтовый селектор, привязки чекбокса/date picker к выбранному статусу и стили. `bundle exec rails db:migrate` выполнен; `schema.rb` вручную очищен от нерелевантного шума локальной регенерации и оставляет только version задачи и новые поля. Проверены Ruby-синтаксис измененных серверных файлов, компиляция CoffeeScript измененных файлов, `git diff --check`; страница `https://manager.leveltravel.dev/orders/all/255012167` открыта через Playwright, селектор с тремя значениями виден в блоке операторских платежей.

Критически важные решения: селектор нужно показывать всегда; порядок статусов для default selection фиксированный: refund statement, written transfer, transferred; новые поля называются `has_transferred` и `transferred_at`; несколько статусов могут быть включены одновременно; при включении чекбокса дата выставляется текущим временем по аналогии с существующим заявлением на возврат; тесты пока не писать.

Точка продолжения при прерывании: проверить финальный `git status` и подготовить краткий отчет.

## 5. Новая итерация требований

Дата обновления контекста: 2026-05-19.

Новые требования:

- Убрать статус `canceled` у `OperatorPayment` и все связанное с ним на беке и фронте.
- При создании целевого `OperatorPayment` в `OperatorPayments::Transfer` логировать создание так же, как это делает `Manager::ResourceController` через `ManagerTracker` при обычном создании операторского платежа. Пример ожидаемого текста: `Создан операторский платеж "Тур №116355601776 [4]" — 333 USD`.
- В форме переноса `app/assets/javascripts/manager/views/popups2/operator_payment_transfer_process_popup.coffee` и шаблоне `app/assets/javascripts/manager/templates/shared/popups/_operator_payment_transfer_process.hjs` добавить выбор валюты по аналогии с формой создания `OperatorPayment` в `app/assets/javascripts/manager/templates/shared/popups/_operator_payment_editor.hjs`.
- Валюта по умолчанию в форме переноса должна браться из исходного `OperatorPayment`, с которого создается перенос.
- Слева от селектора валюты должен быть курс, редактируемый вручную и обновляемый кнопкой динамической загрузки курса.
- При создании целевого `OperatorPayment` в `OperatorPayments::Transfer` сохранять выбранные `currency` и `currency_rate`, если они пришли с фронта.

## 6. Что уже выяснено по новой итерации

- Список валют в обычной форме создания операторского платежа сейчас не формируется на беке. Он захардкожен во view `Manager.OperatorPaymentPopup` как `currencies: ['USD', 'EUR', 'RUB']`.
- Курс в обычной форме обновляется через `Manager.OperatorPaymentPopup#update_rates`: берется `record.currency`, приводится к lower-case и ищется в `currency_rates`.
- `currency_rates` у оператора загружаются через `Operator#currency_rates` на фронте, который вызывает action `currency_rates` у `Manager::OperatorsController`.
- `Manager::OperatorsController#currency_rates` возвращает курсы только для `eur` и `usd`: `Currency.convert(..., 'rub', 1, operator_id, false)`. Курс для `rub` явно не возвращается.
- Форма переноса сейчас отправляет только `amount`, `target_order_id` и `state_transfer`; валюты и курса в данных попапа нет.
- `OperatorPayments::Transfer#create_target_operator_payment!` сейчас создает целевой платеж только с `name` и `requested_amount`.
- В `Manager::ResourceController` лог создания идет через `ManagerTracker.new(:create, object, current_user)` до `save`, затем `track.save!` после успешного `save`.
- `ManagerTracker` для `OperatorPayment` строит текст из `object.requested_amount` и `object.currency`, поэтому для корректного лога целевого платежа валюта должна быть выставлена до вызова трекера.

## 7. План работ по новой итерации

- [x] Шаг 1: Уточнить бизнес-правила по валюте `RUB` в форме переноса: курс для `RUB` должен быть `1`.
- [x] Шаг 2: Уточнить, что значит "убрать статус canceled": статус остается на беке, на фронте скрывается возможность перехода в него.
- [ ] Шаг 3: Скрыть frontend action `cancel` для `OperatorPayment`, не меняя backend AASM и финансовую логику исторических `canceled`.
- [ ] Шаг 4: Добавить валюту и курс в данные попапа переноса: defaults из исходного платежа `record.currency` и `record.currency_rate`; если данных нет, fallback `currency = 'USD'`, `currency_rate = 0.0`.
- [ ] Шаг 5: Добавить в шаблон переноса блок "курс + кнопка обновления + селектор валюты" по паттерну `_operator_payment_editor.hjs`.
- [ ] Шаг 6: Добавить в transfer popup загрузку актуальных курсов через `record.operator` или `order.package.operator`, выбрать курс по текущей валюте, для `RUB` ставить `1`, обновлять `data.currency_rate` по кнопке.
- [ ] Шаг 7: Передавать `currency` и `currency_rate` из `Manager::OperatorPaymentsController#transfer` в `OperatorPayments::Transfer`.
- [ ] Шаг 8: В `OperatorPayments::Transfer#create_target_operator_payment!` сохранять пришедшие `currency` и `currency_rate`; если они не пришли, использовать fallback `USD` и `0.0`.
- [ ] Шаг 9: В `OperatorPayments::Transfer` логировать создание целевого платежа в целевом заказе через `ManagerTracker` так же, как в `Manager::ResourceController`; сумма в логе остается как в `ManagerTracker`.
- [ ] Шаг 10: Обновить/добавить focused specs для `OperatorPayments::Transfer`: создание целевого платежа с валютой/курсом, fallback `USD`/`0.0`, лог создания в целевом заказе, отсутствие создания при `state = transfer`.
- [ ] Шаг 11: Выполнить статические проверки CoffeeScript/Ruby и focused RSpec для измененного сервиса.

## 8. Вопросы для уточнения

- Ответ: существующие `operator_payments.state = 'canceled'` не мигрировать; backend-статус оставить.
- Ответ: финансовую backend-логику с `canceled` не менять; скрыть только frontend-переход в статус.
- Ответ: для валюты `RUB` курс должен быть `1`.
- Ответ: если фронт не прислал валюту/курс, fallback `currency = 'USD'`, `currency_rate = 0.0`.
- Ответ: лог создания целевого платежа писать в целевой заказ; в исходном заказе уже пишется лог переноса.
- Ответ: сумма в логе остается как в `ManagerTracker`.
