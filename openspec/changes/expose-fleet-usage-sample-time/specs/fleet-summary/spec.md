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
timestamp of the newest usage sample persisted for that account across its
primary, secondary, and monthly windows. The value MUST be derived from the
account's persisted usage samples, so a fleet consumer can determine how recently
codex-lb last sampled that account's quota upstream.

`usageRecordedAt` describes the account's most recent sampling, not any single
reported window. It MUST reflect the newest persisted sample even when the
response omits that window's quota — for example when an elapsed primary reset is
displayed as absent, or when a monthly-window plan suppresses the primary and
secondary figures. Reporting the newest sampling activity is what lets a consumer
distinguish "codex-lb has not talked to this account recently" from "this window
is not currently reported", so the two values are deliberately independent.

`usageRecordedAt` MUST be `null` when the account has no persisted usage sample.

`lastRefreshAt` MUST continue to report the account's auth-token refresh
timestamp and MUST NOT be redefined as a quota-sample time. Because writing a
usage sample does not refresh the auth token, `lastRefreshAt` MUST NOT be
required to change when `usageRecordedAt` changes.

#### Scenario: Usage sample time accompanies reported quota

- **GIVEN** an account with a persisted usage sample backing its reported windows
- **WHEN** a client calls `GET /api/fleet/summary` with a valid Bearer API key
- **THEN** that account's `usageRecordedAt` equals the newest persisted usage-sample timestamp for the account

#### Scenario: Sample time survives a window whose quota is not reported

- **GIVEN** an account whose newest persisted usage sample is a primary-window row whose reset has already elapsed
- **AND** the response therefore reports no primary quota for that account
- **WHEN** a client calls `GET /api/fleet/summary` with a valid Bearer API key
- **THEN** that account's `usageRecordedAt` still equals that newest persisted usage-sample timestamp

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
