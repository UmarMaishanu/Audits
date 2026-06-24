# [H-01] Spot Price Used in Seize Calculation While EMA Used for Liquidation Eligibility Allows Over-Seizure During Price Divergence

## Finding Metadata

| Property | Detail |
|:---------|:-------|
| **Severity** | 🔴 High |
| **Category** | Logic Error — Oracle Price Inconsistency |
| **Contract** | `market.move` |
| **Function** | `liquidate_calculate_seize_ctokens()` |
| **Contest** | Current Finance (March 2026) |
| **Platform** | Sherlock |
| **Ranking** | #33 of 509 auditors |

---

## Summary

The `liquidate_calculate_seize_ctokens` function uses `get_spot_price` for both debt and collateral prices when computing how much collateral to seize from a borrower. However, the upstream eligibility check `ensure_liquidate_borrow_allowed` uses `get_price` (EMA) to determine whether a position is liquidatable. These two price sources can diverge significantly during volatile market conditions, allowing a liquidator to seize more collateral than the protocol intends — causing direct and unwarranted financial loss to borrowers.

---

## Vulnerability Details

### Root Cause

The liquidation flow uses **two different oracle price types** for two sequential steps in the same operation — EMA for eligibility, spot for seize calculation — without any consistency enforcement or divergence check in the seize path.

### Affected Execution Flow

```
liquidate<MarketType, TOKEN_A, TOKEN_B>()
  └─► ensure_liquidate_borrow_allowed()
        └─► debts_value_usd_for_liquidation()
              └─► get_price()                        // ✅ Uses EMA price
  └─► liquidate_calculate_seize_ctokens()
        └─► get_spot_price()                         // ❌ Uses spot price — inconsistent
  └─► seize_collaterals()                            // Seizes inflated amount
```

### Technical Explanation

In `contracts/protocol/sources/internal/market/market.move`:

1. **Eligibility check** (`market.move:722` → `market.move:927`): `ensure_liquidate_borrow_allowed` calls `debts_value_usd_for_liquidation` (`market.move:1129`), which calls `get_price` at `market.move:1155`. This function, defined at `x_oracle/sources/entry_points/user.move:26`, returns the **EMA price**.

2. **Seize calculation** (`market.move:1045-1046`): `liquidate_calculate_seize_ctokens` calls `get_spot_price`, defined at `x_oracle/sources/entry_points/user.move:34`, which returns the **spot price**.

3. **Existing safeguard not applied**: The protocol already has `get_price_with_check` (imported at `market.move:13`) which enforces EMA/spot bounds via a per-asset `ema_spot_tolerance`. This function is used in `is_obligation_safe` for borrow/withdraw health checks — but is **never called in the liquidation seize path**.

When the spot price of the debt asset exceeds its EMA price (a natural occurrence during short-term volatility spikes), the seize calculation inflates the USD value of the debt being repaid, causing more collateral to be seized than the protocol's EMA-based valuation would justify.

---

## Impact

**Severity: HIGH**

- **Direct Financial Loss** — The borrower suffers excess collateral loss equal to `(spot/EMA - 1) × repay_amount × (1 + liquidation_incentive)` in collateral value. On a $100,000 liquidation with a 10% spot/EMA divergence, this results in approximately $11,000 in excess collateral seized beyond the intended penalty.
- **Liquidator Profit at Borrower Expense** — The liquidator captures the entire excess as profit, creating an economic incentive to time liquidations during price divergence.
- **Core Invariant Violation** — The protocol's design intent is that all valuation functions use consistent oracle prices. The seize path silently deviates from this by using spot while every other health and valuation function uses EMA.
- **No Attacker Required** — The vulnerability fires mechanically under normal volatile market conditions. Any natural spot/EMA divergence of 5–15% is sufficient.

---

## Likelihood

**Likelihood: HIGH**

- Spot/EMA divergence of 5–15% occurs regularly during normal market volatility — no manipulation required.
- Liquidations inherently cluster during volatile periods, which is precisely when spot/EMA divergence is greatest.
- The vulnerability triggers automatically for every liquidation during a divergence event — no special attacker setup needed.

### External Pre-conditions

1. Pyth spot price of the debt asset needs to be higher than its EMA price (natural short-term volatility spike, e.g. 5–15% divergence).

### Internal Pre-conditions

None.

---

## Attack Path

| Step | Action | Detail |
|:-----|:-------|:-------|
| 1 | Price divergence occurs | Debt asset TOKEN_A spikes: EMA = $1.00, Spot = $1.10 |
| 2 | Position becomes liquidatable | Borrower's obligation is at or past liquidation threshold |
| 3 | Liquidator calls `liquidate` | `liquidate<MarketType, TOKEN_A, TOKEN_B>` |
| 4 | Eligibility check passes | `ensure_liquidate_borrow_allowed` uses EMA ($1.00) — passes |
| 5 | Seize uses inflated price | `liquidate_calculate_seize_ctokens` uses spot ($1.10) — computes 10% more |
| 6 | Excess collateral seized | Liquidator receives 1,210 TOKEN_B instead of intended 1,100 TOKEN_B for repaying 1,000 TOKEN_A |

---

## Recommended Mitigation

Replace `get_spot_price` in `liquidate_calculate_seize_ctokens` with `get_price` (EMA) to be consistent with all other health and valuation functions in the protocol.

Alternatively, use `get_price_with_check` with the per-asset `ema_spot_tolerance` to revert the liquidation when prices diverge beyond the configured threshold. This would preserve the existing safeguard pattern already used in `is_obligation_safe` for borrow/withdraw operations.

```move
// Before (vulnerable):
let debt_price  = get_spot_price(debt_token);
let coll_price  = get_spot_price(collateral_token);

// After (consistent with protocol design):
let debt_price  = get_price(debt_token);      // EMA — matches eligibility check
let coll_price  = get_price(collateral_token); // EMA — matches eligibility check
```

This ensures the seize calculation uses the same price basis as the eligibility check, eliminating the divergence-based over-seizure vector.

---

*Finding by **Um158057** | Current Finance Contest · Sherlock · March 2026*
