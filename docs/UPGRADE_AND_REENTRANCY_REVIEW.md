# Upgrade, reentrancy and gas review

The contract is a Soroban crate (`apexchainx_calculator`); there is no Solidity `AccessControl` or `ProxyUpgrade` contract. The equivalents are:

- **AccessControl** - the role checks `require_admin` and `require_operator` in `apexchainx_calculator/src/lib.rs`, documented in `docs/AUTH_MODEL.md` and pinned by `auth_matrix_tests.rs`.
- **ProxyUpgrade** - `migrate()` and the `STORAGE_VERSION` steps (v0 -> v1 -> v2 -> v3), documented in `docs/MIGRATION_STATE_CONSUMPTION.md`.

## Reentrancy (ProxyUpgrade #738, ReentrancyGuard #752)

Reentrancy in the Solidity sense needs an outbound call that can call back before state is written. In this crate the only outbound call helper is `cross_contract_safety::safe_invoke_contract`, which wraps `try_invoke_contract` and translates failures into errors. The calculation, config, governance and `migrate()` paths do not call other contracts, so no guard is required. Any future outbound call must go through `safe_invoke_contract` and write state before calling out.

## Access control of the upgrade path (#744)

`migrate(caller)` calls `caller.require_auth()`, loads `ADMIN_KEY` (`NotInitialized` if absent) and rejects any other caller with `Unauthorized`. It is idempotent at the current version, rejects a stored version newer than the binary (`VersionMismatch`), and publishes a `migrate_done` event with `(stored, current)`. Operators and strangers cannot migrate (see `auth_matrix_tests.rs`).

## Gas / resource use (#741, #752)

- History is stored as per-entry sub-keys with a per-outage index (v3), so appends do not rewrite every retained entry.
- History length and config count are cached counters (`HISTLEN`, `CFGCNT`) instead of being recomputed on each read.
- `migrate()` derives the cached values once from the source of truth; read paths use the counters.
- Migration arms run in sequence in one invocation; each arm only touches the keys it introduces.

