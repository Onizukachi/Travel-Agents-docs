# Задача: «Тонкая» модель coupons и удаление legacy-ветки купонов

## 1. Бизнес контекст для задачи
- `PromoCampaign` — единственный источник шаблона купона: номинал, тип скидки, лимиты, условия, срок действия, контент.
- `Coupon` — только носитель состояния экземпляра: `pin`, `uses_left`, `promo_campaign_id`, `certificate_id`, `manager_id`.
- Legacy-ветка генерации/создания купонов (`CouponGroup`, `CouponMaker`, `CouponBulkCreator`, `PAPI V3 CouponsController`) удаляется полностью.
- В рамках текущего PR **не делаем миграции схемы** (по процессу деплоя миграции применяются до кода).

## 2. Подтвержденные решения и обоснования
- Решение A: legacy-компоненты удаляем сразу, без депрекации.
  Обоснование: решение владельца домена, неактуальные потоки исключаются из продукта.
- Решение B: считаем, что в проде нет купонов без `promo_campaign_id`.
  Обоснование: подтверждено бизнесом; data-backfill не нужен как обязательный этап, но нужен защитный rake-check.
- Решение C: `ArchivedCoupon` и его упоминания удаляем как неактуальные.
  Обоснование: подтверждено бизнесом как устаревший путь.
- Решение D: один большой PR.
  Обоснование: согласовано.
- Решение E: «тонкость» модели достигаем в коде без физического удаления колонок из таблицы `coupons`.
  Обоснование: безопасно при порядке деплоя «миграции -> код», исключает падение рантайма из-за отсутствующих колонок до переключения кода.

## 3. Техническая стратегия (без миграций)
- В `Coupon` задаем `self.ignored_columns` для всех legacy-полей, которые не должны существовать в доменной модели.
- Объединяем всю логику из `Coupons::Base/Relations/Validations/Callbacks` напрямую в `app/models/coupon.rb` и удаляем concerns как источник логики.
- Переносим чтение шаблонных значений на `coupon.promo_campaign` (делегаты/методы на модели `Coupon`).
- Удаляем все участки кода, где legacy-поля `coupons` используются напрямую.
- Добавляем rake-задачу валидации целостности (`coupons:assert_promo_campaign_integrity`) и используем её как deployment-check.
- Физические колонки и таблицы legacy остаются в БД как «мертвые» до отдельного релизного окна с миграциями.

## 4. Подробный план реализации (один PR)
- [x] Шаг 1: Зафиксировать финальный контракт `Coupon` и список legacy-колонок для `ignored_columns`.
- [x] Шаг 2: Переписать `app/models/coupon.rb` с явной структурой (ассоциации, валидации, enum/state-методы, делегаты к `promo_campaign`, состояние использования).
- [x] Шаг 3: Полностью объединить concerns `Coupons::Base/Relations/Validations/Callbacks` в `app/models/coupon.rb`, удалить concerns-файлы и оставить логику только в модели `Coupon`.
- [x] Шаг 4: Перевести расчеты скидок и сертификатов на `promo_campaign`-источник (Order, Payment, receipts, serializers, AppliedCertificate, Certificate) (через `Coupon`-делегаты к `promo_campaign` и обновление SQL-расчетов в `Order.compact_orders_query`, `Accounting`, `AccountingOld`).
- [x] Шаг 5: Обновить `PromoCampaigns::CouponCreator`, чтобы в `Coupon` писались только целевые поля.
- [x] Шаг 6: Удалить `CouponGroup` целиком: модель, контроллер, views, factory/spec, роуты, воркер `CouponGenerator`, mailer `CouponMailer`.
- [x] Шаг 7: Удалить `CouponMaker` целиком: контроллер, view, routes, роль/меню-линки (роль/права в `lib/new_roles.json` и `lib/old_roles.json` не трогались по договоренности).
- [x] Шаг 8: Удалить `CouponBulkCreator`, `CouponGenerationScheduler`, `CouponGenerationLog`, ActiveAdmin-ресурс логов и связанные упоминания (миграции/схема не трогались).
- [x] Шаг 9: Удалить `Papi::V3::CouponsController` и маршруты `v3/coupons#create` + `v3/coupons/array_coupons`.
- [x] Шаг 10: Удалить `CouponFactory` и все вызовы из удаляемых потоков.
- [x] Шаг 11: Удалить `ArchivedCoupon` и все его runtime-упоминания (`Order#coupon` fallback, админка archived coupons, старые проверки).
- [x] Шаг 12: Обновить `Coupons::CouponChecker` и связанные сервисы так, чтобы условия/ограничения читались только из `promo_campaign`.
- [x] Шаг 13: Обновить админку `app/admin/coupons.rb` и `check_coupon` на отображение/работу только через `promo_campaign`-данные.
- [x] Шаг 14: Удалить фильтры/формы/параметры, завязанные на legacy-колонки (`value`, `relative`, `title`, `description`, `coupon_group_id`, и т.д.).
- [x] Шаг 15: Добавить rake-задачу `coupons:assert_promo_campaign_integrity` (падает, если найдены купоны без `promo_campaign_id`).
- [x] Шаг 16: Почистить `lib/tasks/coupons.rake` от legacy conversion-task'ов и оставить только актуальные проверки/утилиты.
- [x] Шаг 17: Удалить/обновить тесты legacy-модулей; добавить тесты на новый контракт `Coupon` и запрет обращения к ignored-полям (`spec/models/coupon_spec.rb`, `spec/factories/coupon.rb`, `spec/support/shared_contexts/prepare_certificate_context.rb`, стабилизация `spec/factories/user.rb` / `spec/factories/promo_campaign.rb` для надежного создания промо/купонов).
- [ ] Шаг 18: Прогнать релевантные спеки: promo_campaign/coupon/order/payment/receipts/admin (частично: прогнаны `spec/models/coupon_spec.rb`, `spec/models/certificate_spec.rb`, `spec/services/coupons/coupon_checker_spec.rb`, `spec/workers/promo_campaigns/coupon_generator_worker_spec.rb`, `spec/mindbox/state_processor_spec.rb`, `spec/services/line_items_v2/advanced/builder_full_prepayment_cases_spec.rb`, `spec/apis/payments/uniteller_spec.rb:217`, `spec/apis/payments/uniteller_spec.rb:248`, `spec/models/order_spec.rb:404`, `spec/models/order_spec.rb:405`, `spec/models/promo_campaign_spec.rb:411`, `spec/models/promo_campaign_spec.rb:457`, `spec/models/promo_campaign_spec.rb:476`, `spec/models/promo_campaign_spec.rb:488`, `spec/models/promo_campaign_spec.rb`, `spec/workers/promo_campaigns/archived_cleanup_worker_spec.rb`, `spec/services/promo_campaigns/conditions_params_parser_spec.rb`; 108 examples + 78 examples, 0 failures).
- [ ] Шаг 19: Проверить маршруты и админ-UI после удаления legacy entrypoints (частично: проверены `rails routes` для coupon/promo/check_coupon/score_transfer; legacy endpoints `coupon_maker/coupon_group/papi/v3/coupons` отсутствуют).
- [x] Шаг 20: Подготовить release notes для выката (новые отключенные endpoint'ы и runbook по rake-check) — `.tasks/task-1-release-notes.md`.

## 5. Список полей `coupons`, которые считаются legacy в коде
- `relative`
- `value`
- `used`
- `conditions`
- `expires_at`
- `coupon_group_id`
- `is_generated`
- `prepaid`
- `data`
- `title`
- `description`
- `search_types`
- `personal_uses_limit`
- `max_value`

Целевые рабочие поля в коде:
- `id`
- `pin`
- `uses_left`
- `promo_campaign_id`
- `certificate_id`
- `manager_id`
- `created_at`
- `updated_at`

## 6. Заметки для восстановления сессии
- Согласовано с бизнесом:
  - legacy ветка удаляется сразу;
  - купонов без `promo_campaign_id` не ожидается;
  - `ArchivedCoupon` удаляем;
  - один PR;
  - миграций схемы в этом PR не делаем.
- Основной технический риск: скрытые обращения к legacy-колонкам `coupons` в редких потоках.
  Стратегия: `ignored_columns` + полный grep-аудит + целевые спеки.
- Точка продолжения:
  - доделать Шаг 18 (расширенный прогон + admin UI),
  - закрыть Шаг 19 (маршруты и UI-проверка после удаления entrypoints).
