# Рефакторинг payment_type для advanced receipts

## Что хотели сделать

Убрать определение `payment_type` из `LineItemsV2::Advanced::Builder`.

Причина: билдер товарных позиций не должен угадывать бизнес-сценарий по косвенным признакам вроде `params.present?` или `order.is_certificate?`. Контекст операции уже известен уровнем выше, в `Receipts::Builder`, где есть `receipt_type` и `operation_type`.

Ожидаемая схема:

- `advanced + purchase` -> `advanced_capture`
- `advanced + refund` -> `advanced_refund`
- `detailed + purchase` -> `full_prepayment`
- `detailed + refund` -> `partial_refund`
- если заказ является сертификатом и тип `full_prepayment`, использовать `prepayment`

## Что сделали

- Перенесли выбор `payment_type` в `Receipts::Builder`.
- Убрали автоопределение типа из `LineItemsV2::Advanced::Builder`.
- Сделали `LineItemsV2::Advanced::Builder.create_detailed` явным: он передает `full_prepayment`.
- Для сертификатных заказов нормализуем `full_prepayment` в `prepayment`.
- Перестали использовать внутренний alias `refund` для detailed refund.
- Переименовали advanced payment type class `Refund` в `PartialRefund`.
- Advanced payment type classes больше не определяют метод `payment_type`; тип хранится в `PaymentType::Context` и передается туда из билдера.
- Обновили ручную фискализацию: detailed refund теперь использует `LineItemV2::REFUND_DETAILED_TYE`.
- Обновили спеки для `Receipts::Builder` и `LineItemsV2::Advanced::Builder`.

## Что еще желательно сделать

- Переименовать константу `LineItemV2::REFUND_DETAILED_TYE` в `REFUND_DETAILED_TYPE`, оставив временный alias для совместимости.
- Подумать, можно ли заменить строковые типы в `Receipts::Builder::PAYMENT_TYPES` на константы для всех случаев, включая `prepayment`.
- Проверить, нужны ли отдельные классы `AdvancedCapture`, `AdvancedRefund`, `PartialRefund`, если они почти не содержат поведения.
- Отдельно починить старую проблему тестовой схемы/фабрики `partner :subagent_wl`: сейчас полный прогон `spec/services/receipts/builder_spec.rb` падает с `unknown attribute header_mobile_image`.
- Отдельно разобрать RuboCop violations в `Admin::ManualFiscalizationController`, которые не связаны с этим изменением.
