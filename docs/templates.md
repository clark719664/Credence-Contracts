# Contract Template

Canonical Soroban contract template for the Credence workspace. Copy this crate when starting a new contract to get the correct structure, patterns, and test harness out of the box.

## Overview

`TemplateContract` is a minimal but complete Soroban contract that demonstrates every pattern required by Credence contracts:

- Admin-gated initialisation with a double-init guard
- Typed `DataKey` enum for all storage slots
- `#[contracttype]` structs for on-chain data
- `require_auth()` on every mutating entry point
- `Symbol`-keyed event emission
- Ledger-timestamp recording on writes
- `#![no_std]` + `soroban_sdk` imports

## Types

### `DataKey`

| Variant          | Description                              |
|------------------|------------------------------------------|
| `Admin`          | Stores the contract administrator        |
| `Record(Address)`| Stores a `Record` keyed by owner address |

### `Record`

| Field        | Type  | Description                                  |
|--------------|-------|----------------------------------------------|
| `value`      | i128  | Arbitrary numeric value set by the admin     |
| `updated_at` | u64   | Ledger timestamp of the last write           |
| `expires_at` | u64   | Ledger timestamp when the record expires (0 = never expires) |

## Contract Functions

### `initialize(admin: Address)`

Set the contract admin. Requires `admin.require_auth()`. Panics with `ContractError::AlreadyInitialized` (`#2`) if called more than once. Emits an `initialized` event.

### `set_record(owner: Address, value: i128, expires_at: u64)`

Store or overwrite a `Record` for `owner`. Requires admin authorization (`ContractError::NotInitialized` if uninitialised). Rejects `expires_at != 0 && expires_at <= now` with `ContractError::ExpiryInPast` (`#500`). Records the current ledger timestamp in `updated_at`. Emits a `record_set` event.

### `remove_record(owner: Address)`

Delete the record for `owner`. Requires admin authorization (`ContractError::NotInitialized` if uninitialised). No-op if the record does not exist. Emits a `record_removed` event.

### `purge_expired(owner: Address)`

Explicitly remove an expired record for `owner`. Permissionless by design (removing already-expired state is harmless). Panics with `ContractError::AttestationNotFound` (`#301`) if no record exists or `ContractError::DelegationNotExpired` (`#509`) if the record has not yet expired. Emits a `record_purged` event.

### `get_record(owner: Address) -> Record`

Pure read-only view returning the record for `owner`. Panics with `ContractError::AttestationNotFound` (`#301`) if none exists, or `ContractError::SignatureExpired` (`#222`) if `e.ledger().timestamp() >= expires_at`. Does not mutate storage (since Soroban rolls back writes on any panicking invocation).

### `has_record(owner: Address) -> bool`

Pure read-only view returning `true` if a non-expired record exists for `owner`, `false` otherwise.

### `is_expired(owner: Address) -> bool`

Return `true` if a record exists for `owner` in storage and is currently expired.

## How to handle TTL/expiry

Because Soroban rolls back all ledger writes whenever a contract invocation panics, a `get_record` function that calls `.remove(...)` followed by `panic_with_error!` never persists its deletion. Instead:
1. Validate `expires_at == 0 || expires_at > now` at write time (`set_record`).
2. Keep `get_record`, `has_record`, and `is_expired` strictly read-only.
3. Provide an explicit, non-panicking `purge_expired(owner)` entry point that removes expired records and emits `record_purged`.

### `get_admin() -> Address` / `set_admin(new_admin: Address)`

Exported once through `interfaces::governable::Governable`. Panics with `ContractError::NotInitialized` (`#1`) if the contract has not been initialised.

## Events

| Event               | Topic key              | Data                | Emitted when                     |
|---------------------|------------------------|---------------------|----------------------------------|
| `initialized`       | `initialized`          | `admin`             | Contract is initialised          |
| `record_set`        | `record_set, owner`    | `value`             | A record is created or updated   |
| `record_removed`    | `record_removed, owner`| `()`                | A record is deleted by admin     |
| `record_purged`     | `record_purged, owner` | `()`                | An expired record is purged      |
| `admin_transferred` | `admin_transferred`    | `(old_admin, new)`  | Admin role is transferred        |

## Security

- Double initialisation is rejected (`"already initialized"`).
- All mutating entry points require admin `require_auth()`.
- `get_record` panics rather than returning a default, preventing silent misuse.
- `get_admin` panics on uninitialised contracts, preventing silent misuse.

## Using this template

1. Copy `contracts/templates/` to `contracts/<your_contract>/`.
2. Rename the package in `Cargo.toml` and update the workspace `members` list in the root `Cargo.toml`.
3. Rename `TemplateContract` and `TemplateContractClient` throughout.
4. Replace `DataKey`, `Record`, and entry points with your contract's logic.
5. Keep the test harness structure — add tests for every new entry point.

## Build & test

```bash
# Native test build
cargo test -p templates

# WASM build
cargo build --target wasm32-unknown-unknown --release -p templates
```
