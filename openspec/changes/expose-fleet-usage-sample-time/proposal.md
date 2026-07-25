## Why

Issue #1461 reports that `GET /api/fleet/summary` gives fleet consumers no way to
tell whether the quota values in `primary` and `secondary` are fresh.

`lastRefreshAt` looks like the answer, but it is not. The fleet mapper exposes
`AccountSummary.last_refresh_at`:

- `app/modules/fleet/mappers.py` — `last_refresh_at=account.last_refresh_at if include_usage else None`

and the account mapper populates that from `Account.last_refresh`:

- `app/modules/accounts/mappers.py` — `last_refresh_at=account.last_refresh`

`Account.last_refresh` is the **OAuth/auth-token** refresh timestamp. Writing a
usage sample does not touch it, and refreshing a token does not write a usage
sample, so the field is unrelated to the quota values it appears to describe.

The reporter's observed sequence:

1. `GET /api/fleet/summary` returns weekly `remainingPercent: 57` with an old `lastRefreshAt`.
2. An operator clicks **Force probe** on that account in the dashboard.
3. Weekly remaining changes from 57% to 53%, so a fresh sample was persisted.
4. A new `GET /api/fleet/summary` reflects the new quota — but `lastRefreshAt` is unchanged.

`POST /api/fleet/refresh` has the same gap: it reports `usageWritten` and
`generatedAt` for the batch, but exposes no per-account sample time, so a
consumer still cannot attribute freshness to an individual account.

Downstream fleet consumers therefore cannot distinguish current quota from stale
quota, and may flag fresh values as stale (or trust genuinely stale ones).

## What Changes

- Add `usageRecordedAt` to each `GET /api/fleet/summary` account: the timestamp
  of the newest persisted usage sample backing that account's reported windows.
- Derive it from the usage rows the account summary already loads, taking the
  newest `recorded_at` across the account's primary, secondary, and monthly
  windows. It therefore describes exactly the samples the response reports.
- Leave `lastRefreshAt` in place and unchanged, so existing consumers keep
  working. Its meaning is now documented rather than altered: it is the
  auth-token refresh time, not a quota-sample time.
- Return `usageRecordedAt` as `null` for an account with no persisted usage
  sample, and whenever fleet usage visibility is denied — the same rule
  `lastRefreshAt` already follows, so the field cannot leak quota freshness to
  an API key that is not allowed to see quota.

No new query is introduced: `build_account_summaries` already receives the
per-account `UsageHistory` rows, so the timestamp is read from data the endpoint
loads today. This matters because the fleet summary runs across the whole
account pool, where a per-account lookup would be an N+1 regression.

## Impact

- Affected capability: `fleet-summary`.
- Additive API change: one new nullable field. No field is removed, renamed, or
  repurposed, so existing fleet consumers are unaffected until they opt in.
- No schema change, no migration, and no new setting.
- Fleet consumers can now determine per-account quota freshness directly, and
  the `POST /api/fleet/refresh` gap closes for free: a follow-up
  `GET /api/fleet/summary` shows which accounts actually got a new sample.
