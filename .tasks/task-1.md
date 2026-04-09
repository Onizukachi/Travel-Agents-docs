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
- Переносим чтение шаблонных значений на `coupon.promo_campaign` (делегаты/методы на модели `Coupon`).
- Удаляем все участки кода, где legacy-поля `coupons` используются напрямую.
- Добавляем rake-задачу валидации целостности (`coupons:assert_promo_campaign_integrity`) и используем её как deployment-check.
- Физические колонки и таблицы legacy остаются в БД как «мертвые» до отдельного релизного окна с миграциями.

## 4. Подробный план реализации (один PR)
- [ ] Шаг 1: Зафиксировать финальный контракт `Coupon` и список legacy-колонок для `ignored_columns`.
- [ ] Шаг 2: Переписать `app/models/coupon.rb` с явной структурой (ассоциации, валидации, enum/state-методы, делегаты к `promo_campaign`, состояние использования).
- [ ] Шаг 3: Упростить/удалить concerns `Coupons::Base/Relations/Validations/Callbacks` или встроить нужную логику в новую `Coupon` (частично: из concerns удалены `coupon_group`-связи и параметры).
- [ ] Шаг 4: Перевести расчеты скидок и сертификатов на `promo_campaign`-источник (Order, Payment, receipts, serializers, AppliedCertificate, Certificate).
- [ ] Шаг 5: Обновить `PromoCampaigns::CouponCreator`, чтобы в `Coupon` писались только целевые поля.
- [x] Шаг 6: Удалить `CouponGroup` целиком: модель, контроллер, views, factory/spec, роуты, воркер `CouponGenerator`, mailer `CouponMailer`.
- [x] Шаг 7: Удалить `CouponMaker` целиком: контроллер, view, routes, роль/меню-линки (роль/права в `lib/new_roles.json` и `lib/old_roles.json` не трогались по договоренности).
- [x] Шаг 8: Удалить `CouponBulkCreator`, `CouponGenerationScheduler`, `CouponGenerationLog`, ActiveAdmin-ресурс логов и связанные упоминания (миграции/схема не трогались).
- [x] Шаг 9: Удалить `Papi::V3::CouponsController` и маршруты `v3/coupons#create` + `v3/coupons/array_coupons`.
- [x] Шаг 10: Удалить `CouponFactory` и все вызовы из удаляемых потоков.
- [x] Шаг 11: Удалить `ArchivedCoupon` и все его runtime-упоминания (`Order#coupon` fallback, админка archived coupons, старые проверки).
- [ ] Шаг 12: Обновить `Coupons::CouponChecker` и связанные сервисы так, чтобы условия/ограничения читались только из `promo_campaign`.
- [ ] Шаг 13: Обновить админку `app/admin/coupons.rb` и `check_coupon` на отображение/работу только через `promo_campaign`-данные (частично: удалены `coupon_group`-колонки/ссылки/фильтры).
- [ ] Шаг 14: Удалить фильтры/формы/параметры, завязанные на legacy-колонки (`value`, `relative`, `title`, `description`, `coupon_group_id`, и т.д.) (частично: удален `coupon_group_id`).
- [ ] Шаг 15: Добавить rake-задачу `coupons:assert_promo_campaign_integrity` (падает, если найдены купоны без `promo_campaign_id`).
- [ ] Шаг 16: Почистить `lib/tasks/coupons.rake` от legacy conversion-task'ов и оставить только актуальные проверки/утилиты (частично: убран `coupon_group_id: nil` фильтр).
- [ ] Шаг 17: Удалить/обновить тесты legacy-модулей; добавить тесты на новый контракт `Coupon` и запрет обращения к ignored-полям (частично: удалены factory/spec для `CouponGroup`).
- [ ] Шаг 18: Прогнать релевантные спеки: promo_campaign/coupon/order/payment/receipts/admin (частично: прогнаны `spec/models/certificate_spec.rb` и `spec/models/promo_campaign_spec.rb:484`).
- [ ] Шаг 19: Проверить маршруты и админ-UI после удаления legacy entrypoints (частично: удаленные маршруты проверены через `rails routes`).
- [ ] Шаг 20: Подготовить release notes для выката (новые отключенные endpoint'ы и runbook по rake-check).

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
  - завершить Шаги 1-5 (`Coupon` контракт, `ignored_columns`, перевод на `promo_campaign`),
  - закрыть Шаги 12-14 (CouponChecker + финальная чистка админки/форм/фильтров),
  - доделать Шаги 15-20 (integrity rake-check, зачистка `lib/tasks/coupons.rake`, тесты, release notes).
