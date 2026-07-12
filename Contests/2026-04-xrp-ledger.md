# [L-01] MPT mulRatio Overflow Causes Payment Failure (tecINTERNAL) on Large Balances with Transfer Fee

## Finding Metadata

| Property | Detail |
|:---------|:-------|
| **Severity** | 🔵 Low |
| **Category** | Logic Error — Unhandled Overflow |
| **File** | `include/xrpl/protocol/MPTAmount.h` |
| **Function** | `mulRatio()` |
| **Contest** | XRP Ledger (April 2026) |
| **Platform** | Sherlock |
| **Classification** | 🏆 Solo finding — unique, no duplicates |

---

## Summary

When an MPT issuance has a non-zero transfer fee and a payment involves large MPT amounts (approaching `maxMPTokenAmount`), the `mulRatio` function in `MPTEndpointStep` throws a `std::overflow_error`. This exception is caught by `RippleCalc` and converted to `tecINTERNAL`, causing the payment to fail unexpectedly. The sender loses their transaction fee while the payment is not executed.

---

## Vulnerability Details

### Root Cause

The `mulRatio` function throws an unhandled overflow exception instead of gracefully capping the result when a large MPT amount is multiplied by the transfer rate.

### Affected Execution Flow

```
payment(alice → bob, large MPT amount)
  └─► MPTEndpointStep (reverse pass)
        └─► mulRatio(srcToDst, srcQOut, QUALITY_ONE, roundUp=true)
              └─► srcToDst × srcQOut / QUALITY_ONE > INT64_MAX
                    └─► Throw<std::overflow_error>("MPT mulRatio overflow")  // ❌
  └─► RippleCalc.cpp:83-90 catches std::exception
        └─► Returns tecINTERNAL                                              // Opaque error
```

### Technical Explanation

In [`MPTAmount.h:140-161`](https://github.com/sherlock-audit/2026-04-xrp-ledger-april-2026/blob/main/rippled/include/xrpl/protocol/MPTAmount.h#L139-L161), the `mulRatio` function computes `value × num / den` and throws if the result exceeds `INT64_MAX`:

```cpp
if (r > std::numeric_limits<MPTAmount::value_type>::max())
    Throw<std::overflow_error>("MPT mulRatio overflow");
```

This is called during the reverse pass of `MPTEndpointStep` at [`MPTEndpointStep.cpp:484`](https://github.com/sherlock-audit/2026-04-xrp-ledger-april-2026/blob/main/rippled/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L484) when computing the input amount needed after applying the transfer fee.

When `srcQOut` is the MPT transfer rate (up to `2 × QUALITY_ONE` for a 100% fee) and `srcToDst` is close to `maxMPTokenAmount` (`INT64_MAX = 9,223,372,036,854,775,807`), the computation overflows.

Additional affected call sites: [`MPTEndpointStep.cpp:506`](https://github.com/sherlock-audit/2026-04-xrp-ledger-april-2026/blob/main/rippled/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L506) and [`MPTEndpointStep.cpp:633`](https://github.com/sherlock-audit/2026-04-xrp-ledger-april-2026/blob/main/rippled/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L633).

---

## Impact

**Severity: LOW**

- **Payment failure** — Legitimate payments of large MPT amounts fail with `tecINTERNAL` when the issuance has any non-zero transfer fee. The sender loses their transaction fee.
- **Breaks core functionality** — The MPT DEX cannot process large payments for issuances with transfer fees, effectively limiting the usable range below `maxMPTokenAmount / (1 + transferFeeRate)`.
- **Uninformative error** — `tecINTERNAL` gives users and applications no actionable information about why the payment failed.
- **Fee burned** — The sender's transaction fee is consumed even though the payment is not executed.

---

## Likelihood

**Likelihood: LOW**

- Requires MPT amounts approaching `INT64_MAX` (9.22 × 10^18), which is valid but uncommon.
- Requires a non-zero transfer fee on the MPT issuance.
- No attacker required — triggers mechanically under normal usage.

### Overflow Thresholds

| Transfer Fee | srcQOut | Overflow when srcToDst exceeds |
|:-------------|:--------|:-------------------------------|
| 1% | 1,010,000,000 | 9.13 × 10^18 |
| 5% | 1,050,000,000 | 8.78 × 10^18 |
| 10% | 1,100,000,000 | 8.38 × 10^18 |
| 50% | 1,500,000,000 | 6.15 × 10^18 |
| 100% (max) | 2,000,000,000 | 4.61 × 10^18 |

---

## Proof of Concept

Added to `src/test/app/FlowMPT_test.cpp`:

```cpp
void testMulRatioOverflow(FeatureBitset features)
{
    testcase("mulRatio overflow with large MPT and transfer fee");
    using namespace jtx;

    Env env(*this, features);
    auto const gw = Account("gateway");
    auto const alice = Account("alice");
    auto const bob = Account("bob");
    env.fund(XRP(100'000), gw, alice, bob);
    env.close();

    std::int64_t const largeMax = 9'223'372'036'854'775'807;  // INT64_MAX
    MPT const USD = MPTTester(
        {.env = env,
         .issuer = gw,
         .holders = {alice, bob},
         .transferFee = 50'000,  // 5% transfer fee
         .maxAmt = static_cast<std::uint64_t>(largeMax)});

    std::int64_t const largeAmount = 9'000'000'000'000'000'000;  // 9 × 10^18
    env(pay(gw, alice, USD(largeAmount)));
    env.close();

    // With 5% transfer fee, the flow engine computes:
    // in = mulRatio(srcToDst, srcQOut=1,050,000,000, QUALITY_ONE, roundUp=true)
    // 9e18 × 1.05 = 9.45e18 > INT64_MAX (9.22e18) → overflow
    env(pay(alice, bob, USD(largeAmount)),
        sendmax(USD(largeAmount)),
        ter(tecINTERNAL));  // BUG: should not throw, should handle gracefully
    env.close();
}
```

**Test Output:**
```
xrpl.app.FlowMPT mulRatio overflow with large MPT and transfer fee
ERR:Flow Exception from flow: MPT mulRatio overflow
ERR:Flow Exception from flow: MPT mulRatio overflow
ERR:Flow Exception from flow: MPT mulRatio overflow
xrpl.app.FlowMPT had 0 failures.
```

The overflow exception triggers three times (once per path attempt). The payment fails with `tecINTERNAL`, confirming the bug.

---

## Recommended Mitigation

Cap the `mulRatio` result instead of throwing when it exceeds `INT64_MAX`:

```cpp
// Before (throws):
if (r > std::numeric_limits<MPTAmount::value_type>::max())
    Throw<std::overflow_error>("MPT mulRatio overflow");

// After (clamp):
if (r > std::numeric_limits<MPTAmount::value_type>::max())
    return MPTAmount(std::numeric_limits<MPTAmount::value_type>::max());
```

Alternatively, add an explicit check in `MPTEndpointStep` before calling `mulRatio` to detect when `amount × rate` would overflow and handle it as an insufficient funds condition (`tecINSUFFICIENT_FUNDS` or `tecPATH_PARTIAL`) rather than an opaque `tecINTERNAL`.

---

*Finding by **Um158057** | XRP Ledger Contest · Sherlock · April 2026*
