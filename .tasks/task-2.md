# Переносы операторского платежа

## 1. Бизнес-контекст задачи

В менеджерке заказа у операторского платежа уже есть процесс возврата: менеджер может открыть popup, указать сумму, выбрать фактический возврат или ожидание возврата, после чего система обновляет суммы и пишет лог в заказ.

Нужно добавить аналогичный процесс для переносов между заявками. Перенос фиксируется на исходном операторском платеже и создает новый операторский платеж в целевом заказе. Это нужно для ситуаций, когда деньги не возвращаются от туроператора, а переносятся с одной заявки на другую.

В таблицу `operator_payments` уже добавлены новые поля:

- `amount_awaiting_transfer` - сумма, ожидающая переноса.
- `transferred_amount` - уже перенесенная сумма.

## 2. Текущий технический контекст

Рабочая директория: `/home/hikaru/projects/work/leveltravel`.

Связанные с текущей задачей файлы:

- `app/assets/javascripts/manager/templates/payments/_operator_payment.hjs`
  - Сейчас отображает суммы операторского платежа: запрошено, списано, возвращено, ожидает возврат, дату ожидания возврата.
  - Кнопка `возврат` находится внутри блока `can_process` и показывается по `can_refund`.
  - Здесь же находится новый селектор статусов документов: заявление на возврат, заявление на перенос, перенесено.
- `app/assets/javascripts/manager/templates/shared/popups/_operator_payment_process.hjs`
  - Сейчас общий popup называется `operator_payment_process`, имеет `title="Возврат"` и action `/operator_payments/:id/action/refund`.
  - В popup есть подпись `Сумма для возврата`, поле суммы и radio-переключатель `Возврат` / `Ожидание возврата`.
- `app/assets/javascripts/manager/views/popups2/operator_payment_process_popup.coffee`
  - Сейчас рассчитан только на возврат.
  - Использует `data.state_refund`, `record.left_amount`, `record.left_awaiting_amount`.
  - По умолчанию выбирает `refund`.
- `app/assets/javascripts/manager/models/operator_payment.coffee`
  - Уже содержит поля возврата и статусы документов.
  - Новые поля `amount_awaiting_transfer` и `transferred_amount` еще нужно добавить.
  - Сейчас есть `can_refund`, `left_amount`, `left_awaiting_amount`, `has_awaiting_refund`.
- `app/controllers/manager/operator_payments_controller.rb`
  - Сейчас есть action `refund`.
  - Для `awaiting_refund` вызывает `payment.awaiting_refund!(params[:amount])`, обновляет `awaiting_refund_at`, сбрасывает `have_written_statement` и пишет лог.
  - Для фактического возврата вызывает `payment.refund!(params[:amount])` и пишет лог.
- `app/models/operator_payment.rb`
  - Уже содержит refund-логику: `refund!`, `awaiting_refund!`.
  - Новые поля переноса нужно добавить в `attr_accessible`, validations и методы обработки.
  - AASM сейчас содержит состояния `created`, `paid`, `awaiting_refund`, `refunded`, `canceled`.
- `app/serializers/operator_payment_serializer.rb`
  - Новые поля переноса нужно добавить в attributes.
- `config/routes/manager_controller_actions.rb`
  - Для operator payments сейчас есть `/action/refund`, `/action/transaction`, `/action/upload`, `/action/change_state`, `/action/change_statement_step`, `/action/change_statement_step_date`.
  - Для переноса, вероятно, нужен новый route `/action/transfer`.
- `config/locales/ru.yml`
  - Нужно добавить локализацию новых атрибутов.
- `db/schema.rb`
  - Уже содержит:
    - `amount_awaiting_transfer`, decimal, precision 12, scale 2, default `0.0`.
    - `transferred_amount`, decimal, precision 12, scale 2, default `0.0`.

## 3. Требуемое поведение

### Отображение сумм переноса

Добавить отображение новых полей рядом с текущими суммами операторского платежа:

- `amount_awaiting_transfer` - сумма в ожидании переноса.
- `transferred_amount` - перенесенная сумма.

Цвет для отображения использовать как у сегмента в `app/assets/javascripts/manager/templates/order/title.hjs` вокруг `segment_for_render` (голубоватый сегмент).

Важно: в формулировке задачи указано добавить отображение в `app/assets/javascripts/manager/templates/shared/popups/_operator_payment_process.hjs` после `awaiting_refund_at` на строке 96, но `awaiting_refund_at` находится в `app/assets/javascripts/manager/templates/payments/_operator_payment.hjs`. Решение: добавлять значения именно в `_operator_payment.hjs` после блока `awaiting_refund_at`, потому что это строка операторского платежа и там уже отображаются аналогичные суммы.

### Кнопка переноса

Добавить кнопку `перенос` в `app/assets/javascripts/manager/templates/payments/_operator_payment.hjs` после кнопки `возврат`.

Логика показа должна быть аналогична кнопке `возврат`, но учитывать атрибуты переноса:

- платеж должен быть списан (`captured_amount > 0`);
- должна оставаться сумма, доступная к переносу;
- должен быть доступен оператор (`operator.id`);
- сумму уже перенесенную и ожидающую переноса нужно учитывать отдельно от возвратов.

Предлагаемый frontend-computed:

- `has_transferred_amount`: `transferred_amount > 0`.
- `has_awaiting_transfer`: `amount_awaiting_transfer > 0`.
- `left_transfer_amount`: `captured_amount - transferred_amount`.
- `left_awaiting_transfer_amount`: `captured_amount - amount_awaiting_transfer`.
- `can_transfer`: по аналогии с `can_refund`, но на базе transfer-сумм.
- `can_process`: должен учитывать и `can_refund`, и `can_transfer`, чтобы dropdown с действиями был виден при доступном переносе.

### Popup переноса

Открывать форму, аналогичную форме возврата `app/assets/javascripts/manager/templates/shared/popups/_operator_payment_process.hjs`.

Тексты для режима переноса:

- вместо `Сумма для возврата` писать `Сумма для переноса`;
- radio-кнопки: `Перенос` и `Ожидание переноса`;
- после ввода суммы добавить input для номера заказа;
- по умолчанию в input номера заказа подставлять id текущего заказа.

Default amount для переноса:

- если есть `amount_awaiting_transfer`, подставлять его;
- иначе подставлять всю доступную сумму: `captured_amount - transferred_amount`.

Поле номера заказа принимает внутренний `orders.id`. По умолчанию в input подставляется `id` текущего заказа.

### Backend: ожидание переноса

При получении состояния `awaiting_transfer`:

- проверить, что `amount_awaiting_transfer + amount <= captured_amount`;
- если проверка не проходит, вернуть `{ success: false }`;
- если проверка проходит:
  - увеличить `amount_awaiting_transfer` на `amount`;
  - создать новый `OperatorPayment` на сумму, пришедшую с фронта;
  - привязать новый платеж к заказу, который пришел с фронта;
  - оставить новый платеж в начальном состоянии как при обычном создании операторского платежа;
  - название нового платежа выставить по аналогии с `app/assets/javascripts/manager/views/popups2/operator_payment_popup.coffee:94`: `Тур №<operator_booking_id> [<index>]`;
  - записать лог в исходный заказ:
    - `Написано на перенос с заявки 11299374 на заявку 11390522 сумма 99227,20`

Номера заявок для лога брать из `order.operator_booking_id`:

- исходная заявка - `payment.order.operator_booking_id`;
- целевая заявка - `target_order.operator_booking_id`.

### Backend: фактический перенос

Если на фронте выбран `transfer` / `Перенос`:

- проверить, что `transferred_amount + amount <= captured_amount`;
- если проверка не проходит, вернуть `{ success: false }`;
- если проверка проходит:
  - увеличить `transferred_amount` на `amount`;
  - пересчитать и обновить `amount_awaiting_transfer`;
  - создать новый `OperatorPayment` на сумму, пришедшую с фронта;
  - привязать новый платеж к заказу, который пришел с фронта;
  - оставить новый платеж в начальном состоянии как при обычном создании операторского платежа;
  - записать лог в исходный заказ:
    - `Перенесено с заявки 11299374 на заявку 11390522 сумма 99227,20`

Предлагаемая логика пересчета `amount_awaiting_transfer`: уменьшать ожидание переноса на сумму фактического переноса, но не ниже нуля, по аналогии с `refund!`, где `amount_awaiting_refund` уменьшается после фактического возврата.

## 4. Предлагаемые технические решения

- Не добавлять новые gems.
- Сделать перенос отдельным backend action, например `transfer`, чтобы не перегружать action `refund` дополнительными ветками.
- Для popup использовать отдельный `operator_payment_transfer_process` по аналогии с возвратом, чтобы не менять существующий refund flow.
- Backend-логику переноса лучше вынести в методы модели `OperatorPayment`, например `awaiting_transfer!` и `transfer!`, чтобы контроллер остался тонким.
- Создание нового операторского платежа лучше держать в сервисе, если понадобится больше полей или побочных эффектов. Минимальный вариант - приватный метод контроллера, но при росте логики предпочтительнее сервис `OperatorPayments::Transfer`.
- Все суммы приводить через `to_d`, как в текущих refund-методах.
- Для логов использовать формат суммы с запятой и двумя знаками после запятой.
- Добавить AASM/frontend AASM-состояния `awaiting_transfer` и `transferred`.
- Тесты на этом этапе не писать по прямому указанию.

## 5. Принятые решения

1. Файл задачи переименован в `.tasks/task-2.md`.

2. Новые суммы отображать в строке платежа `_operator_payment.hjs` после `awaiting_refund_at`.

3. Default amount для переноса брать из `amount_awaiting_transfer`, если он больше нуля; иначе из `captured_amount - transferred_amount`.

4. Менеджер вводит внутренний `orders.id`; по умолчанию подставляется id текущего заказа.

5. Новый `OperatorPayment` в целевом заказе должен оставаться в начальном состоянии как при обычном создании.

6. Лог писать только в исходный заказ.

7. Добавить AASM-состояния `awaiting_transfer` и `transferred`.

8. Тесты пока не писать.

## 6. Структурированный список задач

- [x] Шаг 1: Уточнить открытые вопросы по имени файла, месту отображения, default amount, идентификатору целевого заказа, полям нового платежа, логам, AASM-состояниям и тестам.
- [ ] Шаг 2: Добавить `amount_awaiting_transfer` и `transferred_amount` в `OperatorPayment`: `attr_accessible`, validations, serializer, frontend model, I18n.
- [ ] Шаг 3: Добавить computed-поля во frontend model для отображения и доступности переноса: `has_awaiting_transfer`, `has_transferred_amount`, `left_transfer_amount`, `left_awaiting_transfer_amount`, `can_transfer`.
- [ ] Шаг 4: Обновить строку операторского платежа: отобразить новые суммы после `awaiting_refund_at`, добавить голубоватую стилизацию, добавить кнопку `перенос` после `возврат`.
- [ ] Шаг 5: Реализовать popup переноса: тексты, radio `Перенос` / `Ожидание переноса`, default amount, input номера заказа.
- [ ] Шаг 6: Добавить route и backend action для переноса.
- [ ] Шаг 7: Реализовать backend-проверки суммы для `awaiting_transfer` и `transfer`.
- [ ] Шаг 8: Реализовать создание нового `OperatorPayment` в целевом заказе с корректным именем и суммой.
- [ ] Шаг 9: Реализовать логи по заданным текстам с заявками из `operator_booking_id`.
- [ ] Шаг 10: Добавить или обновить AASM/frontend состояния, если подтверждено, что transfer должен быть частью `state`.
- [ ] Шаг 11: Выполнить статическую проверку измененных Ruby/Coffee/HJS/SCSS файлов.
- [ ] Шаг 12: Тесты не писать по прямому указанию.
- [ ] Шаг 13: Проверить UI в менеджерке по flow из `AGENTS.md`.
- [ ] Шаг 14: Синхронизировать `.tasks/task-2.md` в `../agents_md/`, выполнить там `git pull`, затем commit и push зеркала согласно правилам проекта.

## 7. Заметки для восстановления сессии

Задача продолжает работу после `.tasks/task-1.md`, где был добавлен селектор статусов документов операторского платежа. Рабочая директория перед началом расширения была чистой.

На момент создания этого файла новые DB-поля уже присутствуют в `db/schema.rb`, но прикладной код еще не обновлен. Ключевая неоднозначность по месту отображения новых сумм решена: отображать в шаблоне строки платежа `_operator_payment.hjs`.

Следующая точка продолжения: приступить к реализации по структурированному списку задач.
