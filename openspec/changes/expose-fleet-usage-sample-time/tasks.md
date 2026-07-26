- [x] Add a `usage_recorded_at` field to `AccountSummary` carrying the newest
  persisted usage-sample timestamp for the account.
- [x] Populate it in the account mapper from the primary, secondary, and monthly
  `UsageHistory` rows already passed in, without adding a query.
- [x] Surface it on `FleetAccountSummary` as `usageRecordedAt`, gated by the
  existing fleet usage-visibility rule.
- [x] Leave `lastRefreshAt` untouched for backward compatibility.
- [x] Return `null` when the account has no persisted usage sample.
- [x] Update the exact-field-set assertion in
  `tests/integration/test_fleet_summary_api.py` to include the new field.
- [x] Add product-path coverage: the field reflects the newest sample, is `null`
  without usage history, is `null` when usage visibility is denied, and moves
  when a newer sample is written while `lastRefreshAt` does not.
- [x] Document the field and the `lastRefreshAt` distinction in the
  `fleet-summary` capability spec.
- [x] State explicitly that the sample time describes the account's most recent
  sampling rather than any single reported window, and cover the elapsed-reset case.
