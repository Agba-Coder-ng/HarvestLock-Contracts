# Hash Type Consistency Audit

## Overview
This document audits the usage of fixed-size hash types (`BytesN<32>`) versus variable-length byte types (`Bytes`) for all hash-shaped fields in the HarvestLock escrow contract.

## Findings

### 1. AllocationMember struct (line 328-334)
```rust
#[contracttype]
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct AllocationMember {
    pub member_hash: BytesN<32>,  // Line 329
    /// Basis points of the commitment's total payout this member is
    /// entitled to, off-chain. Record-only in v1 — see `set_allocation`'s
    /// doc comment for why `settle` doesn't pro-rate against this.
    pub share_bps: u32,
}
```
- **Field**: `member_hash`
- **Purpose**: Per-member salted hash (HMAC-SHA256 of phone number with random salt)
- **Type Used**: `BytesN<32>` ✓ Correct
- **Expected Type**: Fixed-size 32-byte hash
- **Consistency**: ✓ Consistent

### 2. Other Structs Audited

#### OracleConfig (lines 269-294)
- Fields: `oracle_contract` (Address), `price_asset` (Symbol), `max_age_secs` (u64), `denominated_amount` (i128)
- No hash-shaped fields

#### OracleRate (lines 302-314)
- Fields: `price` (i128), `timestamp` (u64)
- No hash-shaped fields

#### Commitment (lines 377-488)
- All fields are addresses, amounts (i128), timestamps (u64), basis points (u32), quantities (u32), or enums/booleans
- No hash-shaped fields

#### StatusChanged (lines 513-518)
- Fields: `from` (Status), `to` (Status)
- No hash-shaped fields

### 3. Type Usage Survey

#### BytesN<32> Usage
- **Location**: Line 329 (`AllocationMember.member_hash`)
- **Count**: 1 occurrence
- **Appropriateness**: ✓ Correctly used for fixed-size 32-byte hash

#### Bytes Usage
- **Search Results**: 0 occurrences of plain `Bytes` type
- **Conclusion**: No variable-length byte types are used in the contract

## Conclusion

The HarvestLock escrow contract demonstrates **consistent and correct usage** of hash-related types:

1. **All hash-shaped fields use `BytesN<32>`**: The sole hash-shaped field (`member_hash` in `AllocationMember`) correctly uses the fixed-size `BytesN<32>` type appropriate for its 32-byte HMAC-SHA256 output.

2. **No inappropriate use of variable-length `Bytes`**: The contract contains zero usages of the variable-length `Bytes` type, eliminating the risk of fields that are "supposed to always be a fixed size" being incorrectly typed.

3. **Security Implication**: By using `BytesN<32>` for the member hash, the contract gains compile-time guarantees that:
   - Values are exactly 32 bytes
   - No length validation is needed at runtime
   - The type system prevents insertion of incorrectly-sized hash values

This consistency eliminates a class of latent bugs where wrong-length input would not be rejected by the type system, ensuring that all hash-shaped data maintains its expected fixed-size property.

**Status**: ✅ PASS - All hash-shaped fields consistently use `BytesN<32>` with no inappropriate `Bytes` usage detected.