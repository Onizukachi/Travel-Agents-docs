# Интеграция Vanta POS-кредитования: осталась только обработка callback

## Бизнес-контекст

Основная серверная часть интеграции Vanta POS-кредитования уже реализована для текущей платежной модели проекта.

На текущий момент:
- процессор Vanta уже сделан;
- нужные endpoint-ы уже реализованы;
- остается только обработка callback/webhook;
- реализацию callback-части пока не начинаем, ждем отдельные указания по точному сценарию.

Ключевое ограничение задачи:
- новую сущность кредита не вводим;
- используем существующий `Payment`;
- все события от Vanta обрабатываем через webhook;
- до финального статуса только логируем и сохраняем контекст;
- только `blank.payed` должен приводить к фактической оплате заказа;
- `blank.cooled` должен временно скрывать долг клиента через existing-механику `cooling` и `frozen_amount`.

Целевой пользовательский сценарий:
1. Клиент уходит в виджет Vanta.
2. Vanta шлет промежуточные статусы заявки.
3. Если пришел `blank.cooled`, мы узнаем `coolingExpiresAt` через SOAP, ставим платеж в охлаждение и показываем заказ как без долга.
4. После этого либо приходит `blank.payed` и мы финализируем оплату, либо клиент/банк отменяет заявку и мы снимаем охлаждение без оплаты заказа.

## Текущее состояние проекта

### Что уже сделано

- Реализован `PaymentProcessor::Vanta`
- Реализованы все нужные endpoint-ы для работы с Vanta API
- Настроена админка для управления webhook-подписками
- Подписки webhook можно создавать на нужные события

### Что остается

- Реализовать обработку входящих callback/webhook от Vanta
- Убрать временные заглушки в callback-части
- После этого интеграцию можно считать завершенной

Важно:
- callback-часть пока намеренно не реализуем;
- следующая итерация будет после отдельных указаний по желаемой логике обработки.

### Уже есть

- Входной endpoint: `/payment/notify_vanta` в [`app/controllers/vanta_notifications_controller.rb`](/home/hikaru/projects/work/leveltravel/app/controllers/vanta_notifications_controller.rb)
- Пустой обработчик [`app/apis/vanta_callback_processor.rb`](/home/hikaru/projects/work/leveltravel/app/apis/vanta_callback_processor.rb)
- Создание ссылки на виджет через [`app/apis/payment_processor/vanta.rb`](/home/hikaru/projects/work/leveltravel/app/apis/payment_processor/vanta.rb) и [`app/apis/payment_processor/vanta_endpoints/pay_link.rb`](/home/hikaru/projects/work/leveltravel/app/apis/payment_processor/vanta_endpoints/pay_link.rb)
- Реализованы endpoint-ы Vanta для работы с webhook-подписками и платежной частью
- Готовая инфраструктура платежа:
  - `payment.mark_authorized!`
  - `payment.mark_captured!(amount)`
  - `payment.update_cooling(expires_at)`
  - `payment.reset_cooling`
  - `payment.frozen?`
  - `payment.cooling?`

### Технический долг, который останется до этапа callback-реализации

- В [`app/controllers/vanta_notifications_controller.rb`](/home/hikaru/projects/work/leveltravel/app/controllers/vanta_notifications_controller.rb) есть `binding.pry`
- В [`app/apis/vanta_callback_processor.rb`](/home/hikaru/projects/work/leveltravel/app/apis/vanta_callback_processor.rb) есть `binding.pry`
- На Vanta сейчас нет тестов ни на контроллер, ни на процессор
- Контроллер ожидает результат с `success?`, а сам процессор пока ничего не возвращает

### Важное ограничение по текущему коду

Сейчас `Payment::CREDIT_PROCESSORS` включает только `TBankCredit`, поэтому Vanta не должен идти через `payment_credit` и через `capture_t_bank_credit`. Для Vanta нужно использовать обычный жизненный цикл `Payment`: `mark_authorized!` -> `mark_captured!`.

## Подтвержденные факты из документации

### Webhook API

Base URL подписок:
- `https://api.b2pos.ru/webhooks/?method={method}`

Ожидаемый ответ на входящий webhook:
- HTTP `200`
- тело строго `OK`

Если ответ не `200 OK` с телом `OK`, Vanta считает уведомление недоставленным и повторяет отправку до 15 раз по экспоненте.

### События webhook

Подтвержденные события из PDF:
- `blank.new`
- `blank.wait_for_decision`
- `blank.approve`
- `blank.rejected`
- `blank.canceled`
- `blank.auth_canceled`
- `blank.error`
- `blank.wait_for_auth`
- `blank.cooled`
- `blank.payed`

### Семантика событий

- До выбора банка по одной заявке могут приходить противоречащие статусы по разным банкам.
- До `blank.payed` поля `bankId` и `bankName` фактически неинформативны.
- Только `blank.payed` гарантированно содержит:
  - выбранный банк
  - `sum`
- `sum` приходит в копейках.
- Успешный жизненный цикл заявки заканчивается на `blank.payed`.

### Формат webhook

Полезные поля:
- `event` — имя события
- `creditId` — идентификатор анкеты; это же кандидат на `profileID` для SOAP-статуса
- `orderId` — наш идентификатор заказа, который мы отправляем в виджет; у нас это `payment.order_payment_number`
- `sum` — сумма к перечислению в копейках; meaningful только для `blank.payed`
- `date` — дата перечисления
- `clientJson` — клиентский payload из инициализации виджета
- `bankId`, `bankName` — meaningful только на поздних этапах

### SOAP статус заявки

Из заметок по задаче и документации:
- endpoint: `https://api.b2pos.ru/loan/wsdl.php`
- для запроса нужны `userID`, `userToken` и `orderID` либо `profileID`
- в нашей реализации используем `orderID`, а не `profileID`
- запрашивать статус по одной заявке нельзя чаще 1 раза в 30 секунд
- в ответе нужно брать `coolingExpiresAt`
- `decision = 8` соответствует охлаждению

## Ключевые решения и обоснования

### 1. Не создаем `PaymentCredit`, весь контекст живет в `Payment`

Почему:
- это прямое требование задачи;
- для Vanta нам не нужен отдельный lifecycle кредитной сущности;
- состояние оплаты уже выражается через поля `Payment` и `metadata`.

Что хранить в `Payment`:
- в `data`:
  - только данные финального успешного статуса `blank.payed`
  - `credit_id`
  - `bank_id`
  - `bank_name`
  - `vanta_sum_kopecks`
  - `vanta_paid_at`
- в `metadata`:
  - `cooling_expires_at`
  - `vanta_status_checked_at`

### 2. Все промежуточные события только логируем

До `blank.payed` платеж не должен считаться оплаченным.

Обрабатываем как информационные:
- `blank.new`
- `blank.wait_for_decision`
- `blank.approve`
- `blank.wait_for_auth`

Отрицательные финализации без оплаты:
- `blank.rejected`
- `blank.canceled`
- `blank.auth_canceled`
- `blank.error`

Если отрицательный статус приходит после `blank.cooled`, нужно:
- снять `cooling`
- разморозить платеж
- закрыть текущую попытку без оплаты заказа

### 3. `blank.cooled` = authorize локально + cooling через SOAP

Требуемое поведение:
- получить `coolingExpiresAt` через SOAP по `orderID`
- записать cooling в `payment.metadata`
- поставить
  - `frozen_amount = payment.requested_amount`
  - `frozen_at = Time.current`
- не переводить заказ в `paid`

Почему так:
- пользователь не должен видеть долг;
- у нас уже есть готовая модель охлаждения через `Payment::Coolable`;
- это ровно повторяет желаемую семантику из задачи.

Важно:
- на `blank.cooled` нельзя делать capture;
- cooling должен быть идемпотентным;
- если webhook ретраится, повторная заморозка не должна ломать платеж.

### 4. `blank.payed` = финализация без внешнего capture API

Требуемая последовательность:
1. Если платеж еще не заморожен, вызвать `payment.mark_authorized!`
2. Если есть `cooling`, вызвать `payment.reset_cooling`
3. Пометить платеж списанным через `payment.mark_captured!(payment.requested_amount)`
4. Перевести заказ в `paid`, если `order.may_mark_paid?`
5. Записать логи и payload Vanta

Почему не `payment.capture!`:
- у Vanta в текущей задаче нет реального server-side commit/capture API;
- пользователь прямо зафиксировал: автосписаний не будет, нужно локально пройти состояния `frozen` -> `captured`.

### 5. Сумму из `blank.payed` нужно валидировать до финализации

Правило:
- `params[:sum]` приходит в копейках
- сравниваем `params[:sum].to_i` с `payment.requested_amount.to_i * 100`

Если сумма не совпадает:
- не финализируем платеж;
- пишем явный лог расхождения;
- возвращаем `400`, чтобы Vanta продолжила retry;
- оставляем платеж без изменений до ручного разбора или повторной корректной доставки.

### 6. Логи должны быть идемпотентны

У Vanta есть ретраи и повторные события по банкам.

Поэтому:
- логируем событие в читаемом виде;
- избегаем дублей по одинаковому тексту, как это сделано в `TBankCreditCallbackProcessor`.

### 7. SOAP-клиент нужен отдельным небольшим классом

Предпочтительный вариант размещения:
- `app/apis/payment_processor/vanta_endpoints/payment_info.rb`

Требования к нему:
- Savon
- явный timeout/error handling
- нормализация ответа до простого hash
- SOAP operation: `StatusSelectedOpty`
- request root: `StatusSelectedOptyRequest`
- response root: `StatusSelectedOptyResponse`
- троттлинг не чаще раза в 30 секунд на один `orderID`

Не надо:
- делать большой service layer
- тащить новую доменную сущность

## Закрытые решения по спорным местам

### 1. Несовпадение credentials для SOAP и текущего конфига

Для SOAP используем отдельные credentials:
- `userID: 375212`
- `userToken: b32b80...`
- их нужно брать из Vanta-конфига отдельными ключами, не переиспользуя webhook/widget `token`

### 2. Точное имя SOAP метода

Используем `StatusSelectedOpty`.

Перед кодом надо проверить:
- точное имя operation в Savon client
- правильную форму envelope/body для `StatusSelectedOptyRequest`
- namespace

### 3. Какой `result` брать из SOAP ответа

Для текущей задачи основной метод получения статуса заявки — `StatusSelectedOpty` по `orderID`.

Ожидаем один агрегированный ответ по заявке, в котором нас интересуют:
- `status`
- `coolingExpiresAt`
- `bank`
- `bankName`
- `amount`

Если ответ не содержит `coolingExpiresAt` при `status = 8`, это считаем ошибкой интеграции и логируем явно.

### 4. Что считать финальным отказом после cooling

Принятое правило:
- `blank.rejected`
- `blank.canceled`
- `blank.auth_canceled`
- `blank.error`

для платежа в `cooling/frozen` означают:
- `reset_cooling`
- снять визуальное состояние охлаждения без перевода платежа в `finished`
- не переводить заказ в `paid`

Технически это значит:
- `payment.reset_cooling`
- `payment.update_columns(frozen_amount: 0, frozen_at: nil)`

## Структурированный список задач

- [ ] Шаг 1: Удалить `binding.pry` из [`app/controllers/vanta_notifications_controller.rb`](/home/hikaru/projects/work/leveltravel/app/controllers/vanta_notifications_controller.rb) и [`app/apis/vanta_callback_processor.rb`](/home/hikaru/projects/work/leveltravel/app/apis/vanta_callback_processor.rb)
- [ ] Шаг 2: Привести `VantaNotificationsController` к корректному контракту webhook: `200 OK` + `OK` при успешной обработке, `400` только для действительно невалидного/необрабатываемого запроса
- [ ] Шаг 3: Определить формат результата `VantaCallbackProcessor` и выбрать один стиль на весь flow: `Dry::Monads::Result` предпочтительнее, потому что контроллер уже ожидает `success?`
- [ ] Шаг 4: В `VantaCallbackProcessor` ввести константы событий и единый диспетчер `case params[:event].to_s`
- [ ] Шаг 5: Добавить нормализацию входящих Vanta params и сохранить в `payment.data` только финальные поля `blank.payed`: `creditId`, `bankId`, `bankName`, `sum`, `date`
- [ ] Шаг 6: Реализовать идемпотентное логирование всех промежуточных событий без изменения состояния оплаты
- [ ] Шаг 7: Реализовать обработку отрицательных статусов `blank.rejected`, `blank.canceled`, `blank.auth_canceled`, `blank.error`, включая сценарий выхода из cooling
- [ ] Шаг 8: Добавить SOAP-клиент получения статуса заявки Vanta с Savon и явной нормализацией ответа
- [ ] Шаг 9: Добавить троттлинг SOAP-запроса не чаще одного раза в 30 секунд на `orderID`
- [ ] Шаг 10: Реализовать `blank.cooled`: запросить `coolingExpiresAt` через `StatusSelectedOpty` по `orderID`, поставить `payment.update_cooling`, `frozen_amount`, `frozen_at`, записать лог старта охлаждения
- [ ] Шаг 11: Реализовать `blank.payed`: проверить сумму, при необходимости авторизовать платеж, снять cooling, списать платеж через `mark_captured!`, отметить заказ оплаченным
- [ ] Шаг 12: Добавить request/controller specs на `/payment/notify_vanta` для всех supported events
- [ ] Шаг 13: Добавить unit/spec на `VantaCallbackProcessor`, включая идемпотентность и негативные ветки
- [ ] Шаг 14: Добавить tests на `blank.cooled` с заглушкой SOAP-ответа и на `blank.payed` с суммой в копейках
- [ ] Шаг 15: Проверить, что после `blank.cooled` клиент не видит долг, а после финального отказа долг возвращается

## Точная карта поведения по событиям

### `blank.new`

- записать лог о создании заявки
- вернуть успех без изменения `Payment`

### `blank.wait_for_decision`

- записать лог "ожидание решения"
- вернуть успех

### `blank.approve`

- записать лог "заявка одобрена банком"
- не менять `Payment`

### `blank.wait_for_auth`

- записать лог "ожидает авторизации"
- не менять `Payment`

### `blank.cooled`

- получить статус по SOAP
- найти `coolingExpiresAt`
- если `coolingExpiresAt` найден:
  - `payment.update_cooling(cooling_expires_at)`
  - `payment.update_columns(frozen_amount: payment.requested_amount, frozen_at: Time.current)`
  - записать лог старта охлаждения
- если `coolingExpiresAt` не найден:
  - записать error log
  - не финализировать платеж

### `blank.payed`

- сохранить в `payment.data` только:
  - `credit_id`
  - `bank_id`
  - `bank_name`
  - `vanta_sum_kopecks`
  - `vanta_paid_at`
- провалидировать `sum`
- если платеж еще не frozen, вызвать `payment.mark_authorized!`
- снять cooling
- `payment.mark_captured!(payment.requested_amount)`
- если `order.may_mark_paid?`, перевести заказ в `paid`
- записать лог успешной оплаты
- если `sum` не совпала:
  - записать error log
  - вернуть `400`
  - не менять платеж и заказ

### `blank.rejected` / `blank.canceled` / `blank.auth_canceled` / `blank.error`

- записать лог отказа/отмены/ошибки
- если платеж был в cooling/frozen:
  - `payment.reset_cooling`
  - `payment.update_columns(frozen_amount: 0, frozen_at: nil)`
- заказ не переводить в `paid`

## Заметки для восстановления сессии

### Что уже изучено

- PDF `Webhooks_Партнерское_API_Документация_POS_API_Confluence (2).pdf`
- текущий `task-1.md`
- `VantaCallbackProcessor`
- `VantaNotificationsController`
- похожие реализации:
  - `YandexSplitCallbackProcessor`
  - `TBankCreditCallbackProcessor`
  - `YandexSplitCoolingCheckerWorker`

### Самые важные принятые решения

- Vanta делаем без `PaymentCredit`
- до `blank.payed` только логируем
- каждый webhook payload в `payment.data` не сохраняем
- в `payment.data` пишем только финальные данные `blank.payed`: `credit_id`, банк, сумма, дата
- `blank.cooled` реализуем через существующий `Payment::Coolable`
- SOAP статус для cooling получаем по `orderID`, а не по `profileID`
- SOAP credentials берём из Vanta-конфига отдельными ключами `soap_user_id` и `soap_user_token`
- `blank.payed` локально делает `authorized -> captured -> order paid`
- сумму из webhook валидируем в копейках
- при mismatch суммы логируем ошибку и возвращаем `400`, чтобы Vanta ретраила
- если после `blank.cooled` пришёл отрицательный финал, просто снимаем cooling и разморозку, не ставим `finished`

### С чего продолжать в следующей сессии

1. Удалить `binding.pry`
2. Определить контракт результата процессора и поправить контроллер
3. Добавить event constants и logging skeleton
4. Поднять SOAP client spike и подтвердить реальные credentials/method name
5. После подтверждения SOAP завершить `blank.cooled`
6. Потом закрыть `blank.payed` и тесты

### Файлы, которые точно понадобятся

- [`app/apis/vanta_callback_processor.rb`](/home/hikaru/projects/work/leveltravel/app/apis/vanta_callback_processor.rb)
- [`app/controllers/vanta_notifications_controller.rb`](/home/hikaru/projects/work/leveltravel/app/controllers/vanta_notifications_controller.rb)
- [`app/apis/payment_processor/vanta.rb`](/home/hikaru/projects/work/leveltravel/app/apis/payment_processor/vanta.rb)
- [`app/apis/payment_processor/vanta_endpoints/pay_link.rb`](/home/hikaru/projects/work/leveltravel/app/apis/payment_processor/vanta_endpoints/pay_link.rb)
- [`app/models/payment.rb`](/home/hikaru/projects/work/leveltravel/app/models/payment.rb)
- [`app/models/concerns/payment/coolable.rb`](/home/hikaru/projects/work/leveltravel/app/models/concerns/payment/coolable.rb)
- [`app/apis/yandex_split_callback_processor.rb`](/home/hikaru/projects/work/leveltravel/app/apis/yandex_split_callback_processor.rb)
- [`app/apis/t_bank_credit_callback_processor.rb`](/home/hikaru/projects/work/leveltravel/app/apis/t_bank_credit_callback_processor.rb)
- [`app/workers/yandex_split_cooling_checker_worker.rb`](/home/hikaru/projects/work/leveltravel/app/workers/yandex_split_cooling_checker_worker.rb)
