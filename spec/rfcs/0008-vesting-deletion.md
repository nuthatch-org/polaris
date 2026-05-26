# RFC 0008: Token Vesting Contracts Deletion

| Field | Value |
|-------|-------|
| RFC | 0008 |
| Title | Token Vesting Contracts Deletion |
| Status | Draft |
| Author | polarity |
| Created | 2026-05-26 |
| Depends on | RFC 0007 (GraphToken deletion) |
| Closes issue | #8 |

---

## Summary

Delete `TokenLockWallet.sol` and `TokenLockManager.sol`. There is no token to vest.

---

## Specification

### Deleted

- `TokenLockWallet.sol`
- `TokenLockManager.sol`
- Associated interfaces and deployment scripts

No other contracts reference these. Deletion is clean.

---

## Implementation notes

- Confirm no references to `TokenLockWallet` or `TokenLockManager` remain in other contracts or deployment scripts after deletion.
- Trivial MR. Can be merged alongside RFC 0007.
