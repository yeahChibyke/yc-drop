## High -- [Issue S-1611 in GTE Perps and Launchpad on Code4rena](https://code4rena.com/audits/2025-08-gte-perps-and-launchpad/submissions/S-1611)

- **Title:
Salted Pair Creation in Factory Mismatches Unsalted `pairFor` Computation in Launchpad, Causing `endRewards` to Revert on Wrong Address.**

- **Description:**

In the GTE Launchpad contract, the `endRewards` function computes the Uniswap V2 pair address using `pairFor`, which assumes a standard (unsalted) CREATE2 deployment: `keccak256(abi.encodePacked(hex"ff", factory, keccak256(abi.encodePacked(token0, token1)), uniV2InitCodeHash))`.

However, the `GTELaunchpadV2PairFactory` uses a custom salted CREATE2: `salt = keccak256(abi.encodePacked(token0, token1, _launchpadLp, _launchpadFeeDistributor))`, resulting in a different pair address. When `endRewards` casts the wrong computed address to `IGTELaunchpadV2Pair` and calls `distributor.endRewards(pair)`, it targets a non-existent or incorrect contract, causing a revert (e.g., no code at address or interface mismatch). This prevents ending rewards accrual after graduation, leaving the rewards pool active indefinitely.
- **Impact:**
  - Denial of Service as The rewards pool cannot be deactivated post-graduation, preventing proper closure and potential fee distributions. Rewards may continue accruing incorrectly.
  - Funds are stuck If rewards pool remains active, fees are misallocated or undistributed, leading to lost revenue for bonders. In high-volume pools, this could accumulate to significant unclaimable funds.

- **Proposed mitigation:**

The protocol should update `pairFor` to account for the salt.


## Medium -- [Issue S-352 in GTE Perps and Launchpad on Code4rena](https://code4rena.com/audits/2025-08-gte-perps-and-launchpad/submissions/S-352)

- **Title:
Rounding down in Quote calculation allows underpriced `LaunchToken` purchases by Malicious user, compounding protocol loss over multiple buys.**

- **Description:**

The `_getQuoteAmount` function in the `SimpleBondingCurve` contract uses integer division for calculating the quote amount required for a given base amount during token buys on the Launchpad.

Specifically, the last line of the function `return (quoteReserve * baseAmount) / baseReserveAfter;` will truncate any fractional remainder towards zero due to Solidity's integer arithmetic. This results in a floored quote value, allowing users to pay less than the mathematically ideal amount for tokens.

- **Impact:**
  - The underpayment directly reduces the quoteReserve added to the curve.
  - Subsequent calculations use this understated reserve, causing the error to compound: Future buys are also underpriced relative to the ideal constant product invariant.
  - Attackers can strategically select baseAmount values (as long as they meet the minimum) to maximize the truncation effect, repeatedly buying at a discount until the curve is drained or the bonding phase ends.

- **Proposed mitigation:**

  - This can be mitigated by using a higher minimumBase amount, however, this is not a complete solution as the attacker can still buy with a calculated base amount such that the quote returned is rounded down.
  - A better solution would be to implement a mechanism to handle the rounding error, such as a precision factor or fixed point arithmetic library that can handle decimals more accurately.


## Medium -- [Issue S-562 in GTE Perps and Launchpad on Code4rena](https://code4rena.com/audits/2025-08-gte-perps-and-launchpad/submissions/S-562)

- **Title:
Loss of user rewards to Launchpad whenever stake is adjusted and reward is distributed.**

- **Description:**

When the Launchpad calls `distributor.increaseStake` or `decreaseStake` during token buys/sells via the LaunchToken's transfer hooks, the Distributor computes the user's pending rewards but transfers them to the wrong recipient which is the Launchpad contract itself instead of the intended user account. This occurs because `_distributeAssets` always sends the rewards to `msg.sender` (which is the Launchpad when it proxies the call), effectively stealing the rewards from users.

- **Impact:**
  - Users permanently lose all their accured rewards (base and quote tokens) on stake adjustments. There seems to be no recovery mechanism in the codebase.

- **Proposed mitigation:**

Add a recipient param to `_distributeAssets`. In `increaseStake`/`decreaseStake`, pass `account` as the recipient. Ensure that the Launchpad also checks that calls pass in a valid user as recipient.
