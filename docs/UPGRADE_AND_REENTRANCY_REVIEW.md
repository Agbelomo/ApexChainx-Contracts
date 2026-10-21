# Upgrade, reentrancy and gas review

The contract is a Soroban crate (`apexchainx_calculator`); there is no Solidity `AccessControl` or `ProxyUpgrade` contract. The equivalents are:

- **AccessControl** - the role checks `require_admin` and `require_operator` in `apexchainx_calculator/src/lib.rs`, documented in `docs/AUTH_MODEL.md` and pinned by `auth_matrix_tests.rs`.
- **ProxyUpgrade** - `migrate()` and the `STORAGE_VERSION` steps (v0 -> v1 -> v2 -> v3), documented in `docs/MIGRATION_STATE_CONSUMPTION.md`.

## Reentrancy and OpenZeppelin primitives (ProxyUpgrade #738, ReentrancyGuard #752, #761, #762)

Reentrancy in the Solidity sense needs an outbound call that can call back before state is written. In this crate the only outbound call helper is `cross_contract_safety::safe_invoke_contract`, which wraps `try_invoke_contract` and translates failures into errors. The calculation, config, governance and `migrate()` paths do not call other contracts, so no guard is required. Any future outbound call must go through `safe_invoke_contract` and write state before calling out.

OpenZeppelin is a Solidity library. On Soroban the equivalent primitives are `Address::require_auth()`, typed `SLAError` enums, and `CrossContractSafety` call stack tracking. Outbound calls follow the Checks-Effects-Interactions pattern, and `auth_matrix_tests.rs` (`reentrancy_guard_access_control_tests`) and `cross_contract_safety.rs` (`reentrancy_guard_primitives_tests`) verify that modifiers block unauthorized invocation and maintain rollback isolation (#761, #762).

## Access control of the upgrade path and storage gaps (#744, #764)

`migrate(caller)` calls `caller.require_auth()`, loads `ADMIN_KEY` (`NotInitialized` if absent) and rejects any other caller with `Unauthorized`. It is idempotent at the current version, rejects a stored version newer than the binary (`VersionMismatch`), and publishes a `migrate_done` event with `(stored, current)`. Operators and strangers cannot migrate (see `auth_matrix_tests.rs`).

Regarding storage gaps (#764): Solidity upgradeable contracts reserve `uint256[50] __gap` slots to avoid variable collision across inheritance hierarchies. Soroban instance storage is addressed by symbolic keys (`Symbol`) rather than contiguous numerical slots. Layout collision cannot occur, and upgrade safety is enforced by `STORAGE_VERSION`, migration arms in `migrate()`, and the snapshot in `storage_key_invariant_tests.rs` (pinned in `proxy_upgrade_storage_gap_safety_tests`).

## Gas / resource use and Pausable optimization (#741, #752, #771)

- History is stored as per-entry sub-keys with a per-outage index (v3), so appends do not rewrite every retained entry.
- History length and config count are cached counters (`HISTLEN`, `CFGCNT`) instead of being recomputed on each read.
- `migrate()` derives the cached values once from the source of truth; read paths use the counters.
- Migration arms run in sequence in one invocation; each arm only touches the keys it introduces.
- **Pausable gas optimization (#771)**: `pause` validates input length bounds (`MAX_REASON_LEN`) immediately to fail fast without state lookups on invalid input, and reuses the instance storage handle rather than re-requesting it multiple times. Both `pause` and `unpause` execute well within the 160k instruction budget (verified by `test_pause_invalid_reason_fails_fast_gas` and `test_is_paused_read_gas_efficiency`).


