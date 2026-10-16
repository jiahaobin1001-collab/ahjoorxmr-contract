# Token Whitelist: Guardian Role and Quotas/Tiers

This document describes the emergency guardian role and the volume control system (quotas, tiers, and per-token overrides) in the `ahjoor-token-whitelist` contract.

## 1. Guardian Role

The guardian is an emergency-only role designed to pause suspicious token activity without waiting for a full governance vote. It is a restricted, safety-first role with far less power than the admin.

### 1.1 Guardian vs Admin

| Capability | Admin | Guardian |
|------------|-------|----------|
| Add/remove whitelisted tokens | ✅ | ❌ |
| Deprecate/revive tokens | ✅ | ❌ |
| Set token quotas | ✅ | ❌ |
| Assign risk tiers | ✅ | ❌ |
| Set per-token limit overrides | ✅ | ❌ |
| Suspend a token (emergency pause) | ✅ | ✅ |
| Extend a suspension | ✅ | ❌ |
| Un-suspend a token | ✅ | ❌ |
| Set the guardian address itself | ✅ | ❌ |
| Set the guardian suspension cap | ✅ | ❌ |

### 1.2 Guardian Functions

#### `set_guardian(guardian: Address)`
- **Caller:** Admin only.
- **Purpose:** Assigns (or reassigns) the emergency guardian address.
- **Note:** Once set, only the admin can change the guardian.

#### `guardian_suspend_token(token: Address, reason_hash: BytesN<32>)`
- **Caller:** Guardian only.
- **Purpose:** Immediately suspends a token as an emergency measure.
- **Effect:** The token cannot be used in any payment or transfer until it is unsuspended by the admin.
- **Suspension cap:** The guardian can only suspend tokens up to a configured aggregate cap (see `set_guardian_suspension_cap`). If the cap is reached, the guardian cannot suspend any more tokens until the admin unsuspends some.

#### `set_guardian_suspension_cap(max_suspended_value: i128)`
- **Caller:** Admin only.
- **Purpose:** Sets the maximum aggregate value (in whitelisted token units) that the guardian is allowed to suspend at any one time.
- **Purpose (cont.):** This prevents the guardian from shutting down the entire contract in a worst-case compromise scenario.

### 1.3 Suspension History
Every suspension (whether applied by admin or guardian) is recorded in the suspension history, up to `SUSPENSION_HISTORY_LIMIT` (default 10 entries). Each entry records:
- `start_ledger` – when the suspension began
- `expiry_ledger` – when it auto-expires
- `reason_hash` – hash of the off-chain reason
- `actor` – who applied it (`Admin` or `Guardian`)
- `suspended_by` – the actual address that called the function

---

## 2. Volume Controls: Quotas, Tiers, and Overrides

The contract enforces three layers of volume limits. The **effective limit** for a token is the **most restrictive** value among these three layers.

### 2.1 Token Quota (Per-Token Period Limit)

A token quota defines a rolling-volume cap for a specific token.

#### Functions
- `set_token_quota(token, max_volume_per_period, period_ledgers)` – creates a quota for a token
- `update_token_quota(token, new_max_volume, new_period)` – updates an existing quota
- `remove_token_quota(token)` – removes the quota entirely

#### `TokenQuota` struct
```rust
pub struct TokenQuota {
    pub max_volume_per_period: i128,  // max volume allowed in the period
        pub period_ledgers: u32,           // length of the rolling period (in ledgers)
        }
        ```

        #### Constraints
        - `period_ledgers` must be ≤ `MAX_QUOTA_PERIOD_LEDGERS` (~30 days at 5s ledgers).
        - Volume is tracked via a fixed-size rolling window of `VOLUME_AGG_BUCKET_COUNT` (24) buckets, so the storage cost is constant regardless of the period length.

        ### 2.2 Risk Tier (Global Tier Limit)

        A risk tier defines default limits that apply to all tokens assigned to that tier.

        #### Functions
        - `assign_token_tier(token, tier_name)` – assigns a token to a tier
        - `get_risk_tier(token)` – returns the tier name for a token

        #### `TierLimits` struct
        ```rust
        pub struct TierLimits {
            pub name: soroban_sdk::String,     // e.g. "low", "medium", "high"
                pub max_single_tx_amount: i128,   // max per-transaction amount
                    pub max_daily_volume: i128,        // max daily volume for all tokens in this tier
                    }
                    ```

                    ### 2.3 Per-Token Limit Override

                    A per-token override sets an explicit limit for a specific token, independent of its tier and quota.

                    #### Function
                    - `set_token_limit_override(token, max_single_tx, max_daily)` – sets (or removes by passing 0) an override

                    ### 2.4 How the Three Layers Combine

                    The effective limit for a token is computed as follows:

                    ```
                    effective_max_single_tx = min(
                        quota.max_volume_per_period (if set),
                            tier.max_single_tx_amount (if tier assigned),
                                override.max_single_tx (if override set),
                                )

                                effective_max_daily_volume = min(
                                    quota.max_volume_per_period (if set),
                                        tier.max_daily_volume (if tier assigned),
                                            override.max_daily (if override set),
                                            )
                                            ```

                                            **Rule:** The **most restrictive** (smallest) value wins. If any layer sets a limit, that limit caps the token; if a layer is not set, it is not considered (i.e., it does not relax the other layers).

                                            ### 2.5 Volume Recording
                                            Every successful transfer of a whitelisted token is recorded via `record_token_volume(token, amount)`. The volume is bucketed into the rolling window so that `get_token_volume(token, from_ledger, to_ledger)` can compute the total volume in any window of up to `MAX_VOLUME_QUERY_RANGE` ledgers.

                                            ---

                                            ## 3. Summary

                                            - **Guardian** is an emergency-only role that can suspend tokens within an aggregate cap. It cannot add/remove tokens, set quotas, or modify tiers.
                                            - **Admin** retains full control: adding tokens, setting quotas, assigning tiers, setting overrides, managing the guardian, and all un-suspensions.
                                            - **Volume limits** are the intersection (minimum) of three layers: per-token quota, risk tier, and per-token override. The most restrictive layer always wins.

                                            This design balances speed (guardian can pause immediately in an emergency) with safety (guardian's power is capped, and only the admin can restore service or change the rule set).
