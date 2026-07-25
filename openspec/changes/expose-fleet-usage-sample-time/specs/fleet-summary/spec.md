## MODIFIED Requirements

### Requirement: Fleet summary requires API key authentication

The system SHALL expose `GET /api/fleet/summary` for trusted local fleet consumers. The route MUST require a valid Bearer API key even when global proxy API-key authentication is disabled.

#### Scenario: Missing fleet summary key is rejected

- **WHEN** a client calls `GET /api/fleet/summary` without a Bearer token
- **THEN** the system returns 401
- **AND** no account summary payload is returned

#### Scenario: Valid fleet summary key returns account capacity

- **WHEN** a client calls `GET /api/fleet/summary` with a valid Bearer API key
- **THEN** the response includes `accounts[]`
- **AND** each account includes `accountId`, `displayName`, `email`, `status`, `planType`, `primary`, `secondary`, `lastRefreshAt`, and `usageRecordedAt`
- **AND** each window includes `remainingPercent`, `resetAt`, and `windowMinutes`

## ADDED Requirements

### Requirement: Fleet summary reports per-account quota sample time

Fleet summary responses MUST expose `usageRecordedAt` for each account: the
timestamp of the newest persisted usage sample backing the quota values reported
in that account's `primary` and `secondary` windows. The value MUST be derived
from the account's persisted usage samples, so a fleet consumer can determine
whether the reported quota is current.

`usageRecordedAt` MUST be `null` when the account has no persisted usage sample.

`lastRefreshAt` MUST continue to report the account's auth-token refresh
timestamp and MUST NOT be redefined as a quota-sample time. Because writing a
usage sample does not refresh the auth token, `lastRefreshAt` MUST NOT be
required to change when `usageRecordedAt` changes.

#### Scenario: Usage sample time accompanies reported quota

- **GIVEN** an account with a persisted usage sample backing its reported windows
- **WHEN** a client calls `GET /api/fleet/summary` with a valid Bearer API key
- **THEN** that account's `usageRecordedAt` equals the newest persisted usage-sample timestamp for the account

#### Scenario: Account without usage history reports no sample time

- **GIVEN** an account with no persisted usage sample
- **WHEN** a client calls `GET /api/fleet/summary` with a valid Bearer API key
- **THEN** that account's `usageRecordedAt` is `null`

#### Scenario: A newer usage sample advances the sample time independently of token refresh

- **GIVEN** a fleet consumer has read an account's `usageRecordedAt`
- **WHEN** a newer usage sample is persisted for that account without an auth-token refresh
- **AND** the consumer calls `GET /api/fleet/summary` again
- **THEN** that account's `usageRecordedAt` reflects the newer sample
- **AND** `lastRefreshAt` is not required to have changed

### Requirement: Fleet summary usage sample time follows fleet usage visibility policy

`usageRecordedAt` MUST follow the same visibility rule as the account's quota
windows and `lastRefreshAt`. When fleet usage visibility is denied for the
calling API key, `usageRecordedAt` MUST be `null`, so quota freshness metadata is
not disclosed to a key that is not permitted to read quota.

#### Scenario: Usage sample time is hidden when usage visibility is denied

- **GIVEN** an API key that is not permitted to view fleet account pool usage
- **WHEN** that key calls `GET /api/fleet/summary`
- **THEN** each account's `usageRecordedAt` is `null`
- **AND** each account's `lastRefreshAt` is `null`
