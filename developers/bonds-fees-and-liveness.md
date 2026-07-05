# Bonds, Fees & Liveness

This page collects the exact economics of `pythiaoorcle`: how bonds are sized, how fees are taken, and how liveness bounds work — with worked examples.

## Final fee

Each whitelisted collateral has a flat **final fee**, configured per token in `pythiastore1` via `setfinalfee` and read cross-contract with `store::get_final_fee`. The final fee is the base unit that bonds and minimum bonds are derived from. It defends against spam: every serious answer must post at least a final fee's worth of value.

## Burned-bond rate

On a **disputed** settlement, a fraction of the losing bond is taken as an **oracle fee** and sent to the fee vault. That fraction is the burned-bond rate:

* Set via `pythiaoorcle::setburnedbps(bps)`; `0` falls back to `DEFAULT_BURNED_BOND_BPS` = **5000 (50%)**.
* Bounded to ≤ 10000 (100%).

$$
\text{oracleFee} = \text{bond} \times \frac{\text{burnedBondBps}}{10000}
$$

## OOv2 bonds

* **Default bond** = `DEFAULT_BOND_MULTIPLIER` × final fee = **2 × finalFee**. Override with `setbond` while the request is `REQUESTED`.
* **Each side posts bond + final fee.** The proposer deposits `bond + finalFee` (memo `bond:<id>`); a disputer deposits the same.

### Payouts

Let **B** = bond, **F** = final fee, **f** = burnedBondBps/10000, **R** = reward.

**Undisputed (proposer wins):**

$$
\text{proposer receives } = B + F + R
$$

(They simply get their own stake back plus the reward.)

**Disputed (winner = whoever matches the DVM price):**

$$
\begin{aligned}
\text{oracleFee} &= B \times f \\
\text{fee vault} &= F_\text{loser} + \text{oracleFee} \\
\text{winner}    &= 2B + F_\text{winner} - \text{oracleFee} + R
\end{aligned}
$$

The winner recovers both bonds minus the oracle fee, keeps their own final fee, and takes the reward; the loser forfeits their bond and final fee.

## OOv3 bonds

* **Minimum bond** is derived from the final fee and the burn rate:

$$
\text{minBond} = \text{finalFee} \times \frac{10000}{\text{burnedBondBps}}
$$

At the default 50% burn rate, `minBond = 2 × finalFee`. Read it via the static getter `oracle::getminbond(oracle_acct, currency)`.

* The asserter deposits at least `minBond` (memo `assert`); a disputer deposits an equal bond.

### Payouts

**Undisputed (asserter wins):**

$$
\text{asserter receives } = B \quad(\text{full bond back, no fee})
$$

**Disputed (winner = asserter if DVM votes TRUE, else disputer):**

$$
\begin{aligned}
\text{oracleFee} &= B \times f \\
\text{fee vault} &= \text{oracleFee} \\
\text{winner}    &= 2B - \text{oracleFee}
\end{aligned}
$$

At 50% burn, the winner nets **1.5 × B** — their own bond back plus the loser's bond minus the fee.

## Worked example

Assume final fee **F = 1.0000 PYTHIA**, default bond **B = 2.0000 PYTHIA**, burn rate **f = 50%**, reward **R = 5.0000 PYTHIA**.

| Scenario | Winner receives | Fee vault | Loser loses |
|---|---|---|---|
| OOv2 undisputed | `B + F + R` = 2 + 1 + 5 = **8.0000** | 0 | — |
| OOv2 disputed | `2B + F − 0.5B + R` = 4 + 1 − 1 + 5 = **9.0000** | `F + 0.5B` = 1 + 1 = **2.0000** | `B + F` = **3.0000** |
| OOv3 undisputed | `B` = **2.0000** | 0 | — |
| OOv3 disputed | `2B − 0.5B` = 4 − 1 = **3.0000** | `0.5B` = **1.0000** | `B` = **2.0000** |

(OOv2 disputed: the winner posted `B + F` and receives `2B + F − oracleFee + R`; net gain over their own stake is the loser's `B − oracleFee` plus `R`.)

## Liveness

Liveness is the challenge window during which an answer can be disputed.

| Constant | Value | Meaning |
|---|---|---|
| `DEFAULT_LIVENESS` | 7200 s (2 h) | Default OOv2/OOv3 liveness. |
| `MIN_LIVENESS` | 30 s | Absolute minimum (production should use ≥ 600 s). |
| `MAX_LIVENESS` | 604800 s (7 d) | Maximum. |

* **OOv2:** set per request with `setliveness` (while `REQUESTED`), or use the oracle default.
* **OOv3:** pass `liveness` directly to `asserttruth` (or the default via `assertwdflt`).

{% hint style="info" %}
The 30-second minimum exists so test environments can cycle quickly. On mainnet, choose a liveness long enough that honest disputers can realistically notice and challenge a wrong answer for the value at stake — a few hours or more for high-value data.
{% endhint %}

## Ancillary data / claim size

The `ancillary_data` (OOv2) and `claim` (OOv3) byte length is capped by the oracle's `ancillary_bytes_limit` (default **8192 bytes**, adjustable with `setlimit`). Keep claims concise but complete.
