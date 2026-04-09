# Release Notes: Task-1 (Coupons Legacy Cleanup)

## Scope
- Consolidated coupon logic into `Coupon` model; removed legacy coupon concerns.
- Removed legacy coupon generation branch (`CouponGroup`, `CouponMaker`, `CouponBulkCreator`, legacy PAPI coupons endpoints, legacy logs/mailers/factories previously tied to removed flow).
- `Coupon` switched to thin-contract mode with `ignored_columns` for legacy DB fields.
- Coupon creation moved to `PromoCampaign + PromoCampaigns::CouponCreator` flow.
- Removed `is_generated` usage from runtime/admin/tests/locales.
- Updated SQL calculations to use `promo_campaigns.value/relative` instead of `coupons.value/relative`.
- Added integrity check task: `coupons:assert_promo_campaign_integrity`.

## No DB Migrations In This PR
- This rollout is code-only.
- Physical legacy columns/tables remain in DB intentionally for safe staged removal later.

## Disabled/Removed Runtime Entry Points
- Legacy PAPI coupons flow removed (`v3/coupons#create`, `v3/coupons/array_coupons`).
- Legacy admin flows for removed coupon branch (`coupon_maker`, `coupon_group`, generation logs) are not present.

## Operational Runbook
1. Deploy code.
2. Run integrity check:
   - `bin/rake coupons:assert_promo_campaign_integrity`
3. Expected result:
   - `Coupons integrity OK: all coupons have promo_campaign_id`
4. If failed:
   - task prints first problematic coupon IDs/PINs;
   - stop rollout progression and investigate data anomalies before continuing.

## Rollback Notes
- Code rollback is safe because DB schema was not changed in this PR.
- If rollback is required, revert application code only.
