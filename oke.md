o

# High

## [HIGH-01] Unauthorized Token Claim Issue in SecondSwap_StepVesting contract
A critical vulnerability in the `claimable()` function of the SecondSwap_StepVesting contract allows users to claim more tokens than their remaining vesting balance, potentially leading to unauthorized token extraction. The `claimable()` function calculates token claims based on the original total vesting amount without properly adjusting for partial claims or token transfers. This creates a discrepancy between the calculated claimable amount and the actual remaining vesting balance.

```solidity
function claimable(address _beneficiary) public view returns (uint256, uint256) {
    Vesting memory vesting = _vestings[_beneficiary];
    if (vesting.totalAmount == 0) {
        return (0, 0);
    }

    uint256 currentTime = Math.min(block.timestamp, endTime);
    if (currentTime < startTime) { 
        return (0, 0);
    }

    uint256 elapsedTime = currentTime - startTime;
    uint256 currentStep = elapsedTime / stepDuration; 
    uint256 claimableSteps = currentStep - vesting.stepsClaimed;

    uint256 claimableAmount;

    if (vesting.stepsClaimed + claimableSteps >= numOfSteps) {
        claimableAmount = vesting.totalAmount - vesting.amountClaimed;
        return (claimableAmount, claimableSteps);
    }
    claimableAmount = vesting.releaseRate * claimableSteps; // @audit-issue: doesn't account for already claimed
    return (claimableAmount, claimableSteps);
}
```

**Impact:** Loss of users' fund
**Proposed mitigation:** Modify the `claimable()` function to properly account for already claimed amounts by using `(vesting.totalAmount - vesting.amountClaimed)/ (numOfSteps - vesting.stepsClaimed)` instead of `vesting.releaseRate * claimableSteps`.

## [HIGH-02] Incorrect Token Transfer Destination in _forceRepay Function
The `_forceRepay` function incorrectly transfers yield tokens to the contract itself (`address(this)`) instead of the transmuter. In the `_forceRepay` function, yield tokens are transferred to `address(this)` instead of the `transmuter`, meaning that tokens intended to go to the transmuter are instead stuck in the contract, preventing proper protocol functioning.

```solidity
function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
    // ...
    // Transfer the repaid tokens from the account to the transmuter.
    TokenUtils.safeTransfer(yieldToken, address(this), creditToYield); // @audit-issue: incorrect address
    return creditToYield;
}
```

**Impact:** This bug prevents the transmuter from receiving repaid yield tokens, which breaks core protocol functionality. Any debt that should be repaid through forced repayments during liquidations will not actually be processed by the transmuter.
**Proposed mitigation:** Fix the token transfer destination in the `_forceRepay` function to transfer tokens to `transmuter` instead of `address(this)`.

## [HIGH-03] Unauthorized Override Assignment Enables Arbitrary Fund Theft from Unclaimed Rewards
The `overrideReceiver()` function lacks access control validation, allowing any address to designate another address as their override and subsequently drain that address's unclaimed rewards through the migration mechanism in `removeOverrideAddress()`. The overrideReceiver() function in RewardManager.sol fails to validate that the caller (msg.sender) has legitimate authority to set an override and that the target override address (overrideAddress) has consented to the relationship.

```solidity
function overrideReceiver(address overrideAddress, bool migrateExistingRewards) external whenNotPaused nonReentrant {
    if (migrateExistingRewards) { _migrateRewards(msg.sender, overrideAddress); }
    require(overrideAddress != address(0) && overrideAddress != msg.sender, InvalidAddress());
    overrideAddresses[msg.sender] = overrideAddress; // @audit-issue: No authorization check
    emit OverrideAddressSet(msg.sender, overrideAddress);
}
```

**Impact:** Any attacker can steal the complete unclaimed reward balance from any address in the system without requiring any legitimate relationship or authorization from the victim.
**Proposed mitigation:** Implement access control validation by adding a helper function `_isLegitimateReceiver()` to validate that the caller is a legitimate receiver for at least one pubkey before allowing override settings.

## [HIGH-04] Unadjusted Pyth Oracle Prices Cause Misaligned Token Valuations and Inaccurate Position Calculations
The protocol incorrectly uses raw price values returned by the Pyth oracle without aligning them with the corresponding token decimals. Since Pyth expresses prices using a base value and an exponent, and each token has its decimal configuration, failing to normalise these values leads to inaccurate asset pricing throughout the protocol. This misalignment affects all token valuation calculations, including position and debt tracking in the `ShadowRangeVault`. The protocol retrieves price data from the Pyth oracle but directly uses the raw price field without applying the `expo` (exponent) or adjusting it to match the token's decimals.

```solidity
function getPythPrice(address token) public view returns (uint256) {
    bytes32 priceId = pythPriceIds[token];
    if (priceId == bytes32(0)) {
        revert("PriceId not set");
    }
    PythStructs.Price memory priceStruct = pyth.getPriceUnsafe(priceId);
   
    require(priceStruct.publishTime + maxPriceAge > block.timestamp, "Price is too old");
    uint256 price = uint256(uint64(priceStruct.price)); // @audit-issue: Ignores expo, no decimal adjustment
    return price;
}
```

**Impact:** Using unadjusted Pyth prices that don't match token decimal formats results in significant valuation errors leading to incorrect position value and debt assessments, mispricing of collateral and liabilities, unfair liquidations or protocol insolvency risks, and inaccurate health metrics for vaults and users.
**Proposed mitigation:** Modify the getPythPrice function to correctly apply the Pyth exponent and normalize the result to match a consistent decimal format (e.g., 8 decimals) by calculating `adjustedPrice = uint256(price) * (10 ** uint256(8)) / (10 ** uint256(-expo))` for negative exponents.

## [HIGH-05] Inconsistent highTick Processing Between Liquidity Modification and Settlement
The protocol implements inconsistent handling of the upper tick boundary (`highTick`) between the liquidity tree modification phase and settlement phase. In `WalkerLib.modify()`, the high tick is converted using `pInfo.treeTick(highTick) - 1`, while in `PoolWalker.settle()`, it uses `pInfo.treeTick(highTick)` without the subtraction. This discrepancy means the two critical operations that should maintain identical tree state are operating on different tick ranges.

```solidity
// WalkerLib.modify - subtracts 1
uint24 high = pInfo.treeTick(highTick) - 1;

// PoolWalker.settle - no subtraction
uint24 high = pInfo.treeTick(highTick); // @audit-issue: Inconsistent with modify()
```

**Impact:** This bug can cause permanent liquidity tree corruption, leading to incorrect fee distributions, inaccurate price calculations, and potential protocol insolvency as liquidity accounting becomes fundamentally broken.
**Proposed mitigation:** Protocol specific - ensure consistent tick processing between modification and settlement phases.

## [HIGH-06] Incompatible with Native ETH Pools Due to ERC20 Assumptions
The `collectFees` function assumes all pool currencies are ERC20 tokens, making it incompatible with Uniswap V4 pools that contain native ETH. Uniswap V4 introduces native ETH support where Currency can represent either ERC20 tokens or native ETH (address(0)). The `collectFees` function incorrectly assumes all currencies are ERC20 tokens by using `IERC20(Currency.unwrap(token)).balanceOf()`, which will fail for native ETH since address(0) is not a valid ERC20 contract.

```solidity
function collectFees(uint256 nfpId, address recipient) external override {
    _checkOwner();
    
    if (nfpId == 0) revert SuperDCAListing__UniswapTokenNotSet();
    if (recipient == address(0)) revert SuperDCAListing__InvalidAddress();
    
    (PoolKey memory key,) = POSITION_MANAGER_V4.getPoolAndPositionInfo(nfpId);
    Currency token0 = key.currency0;
    Currency token1 = key.currency1;
    
    // @audit-issue: Fails for native ETH
    uint256 balance0Before = IERC20(Currency.unwrap(token0)).balanceOf(recipient);
    uint256 balance1Before = IERC20(Currency.unwrap(token1)).balanceOf(recipient);
    // ...
}
```

**Impact:** Contract cannot collect fees from pools containing native ETH, limiting protocol compatibility with major trading pairs and potentially locking fee revenue.
**Proposed mitigation:** Implement proper native ETH handling using Uniswap V4's CurrencyLibrary for balance checking: `CurrencyLibrary.balanceOf(token, recipient)` instead of `IERC20(Currency.unwrap(token)).balanceOf(recipient)`.

## [HIGH-07] Reward Loss Vulnerability via Index Reset During Stake/Unstake
The `accrueReward` function suffers from a reward calculation flaw where staking or unstaking resets the token's reward index, causing loss of accrued rewards for all stakers in that token bucket when rewards are claimed immediately after position changes. When users stake or unstake, the token's `lastRewardIndex` is updated to the current global rewardIndex. If `accrueReward` is called shortly after, the delta calculation `(rewardIndex - info.lastRewardIndex)` becomes near-zero, resulting in minimal or zero rewards despite significant time elapsed since the last actual reward accrual.

```solidity
function stake(address token, uint256 amount) external override {
    _updateRewardIndex();
    // ...
    TokenRewardInfo storage info = tokenRewardInfoOf[token];
    info.lastRewardIndex = rewardIndex; // @audit-issue: Resets index, loses accrued rewards
    // ...
}

function accrueReward(address token) external override onlyGauge returns (uint256 rewardAmount) {
    _updateRewardIndex();
    TokenRewardInfo storage info = tokenRewardInfoOf[token];
    if (info.stakedAmount == 0) return 0;
    
    uint256 delta = rewardIndex - info.lastRewardIndex; // @audit-issue: Delta becomes ~0 after stake/unstake
    if (delta == 0) return 0;
    
    rewardAmount = Math.mulDiv(info.stakedAmount, delta, 1e18);
    info.lastRewardIndex = rewardIndex;
    return rewardAmount;
}
```

**Impact:** Rewards accumulated between the last `accrueReward` call and any stake/unstake operation are permanently lost, unfairly penalizing stakers and reducing the protocol's reward distribution efficiency.
**Proposed mitigation:** Remove the premature index reset from stake/unstake functions (`info.lastRewardIndex = rewardIndex;`) and only update lastRewardIndex after reward calculation in the accrueReward function.

## [HIGH-08] Same-block earmark early-exit leaves stale transmuter balance, causing under-earmarking
Same-block `earmark()` early-exit leaves a stale transmuter balance, inflating "cover" next block and under‑earmarking debt, letting borrowers overborrow, delaying redemptions, and raising bad‑debt/insolvency risk. In `AlchemistV3.sol::_earmark()`, an early-return guard `if (block.number <= lastEarmarkBlock) return;` exits on subsequent calls within the same block. When the transmuter's MYT balance changes between two same-block calls (e.g., via repay, direct MYT transfer, or equivalent inflow), the second `_earmark()` doesn't update `lastTransmuterTokenBalance`.

```solidity
function _earmark() internal {
    if (totalDebt == 0) return;
    if (block.number <= lastEarmarkBlock) return; // @audit-issue: Early-exit on same block (stale balance risk)

    uint256 transmuterCurrentBalance = TokenUtils.safeBalanceOf(myt, address(transmuter));
    uint256 transmuterDifference =
        transmuterCurrentBalance > lastTransmuterTokenBalance
            ? transmuterCurrentBalance - lastTransmuterTokenBalance
            : 0;
    uint256 amount = ITransmuter(transmuter).queryGraph(lastEarmarkBlock + 1, block.number);

    uint256 coverInDebt = convertYieldTokensToDebt(transmuterDifference);
    amount = amount > coverInDebt ? amount - coverInDebt : 0;

    lastTransmuterTokenBalance = transmuterCurrentBalance; // @audit-issue: Only updated when not early-exiting
    // ...
}
```

**Impact:** The stale `lastTransmuterTokenBalance` causes the next `_earmark()` to over-count cover and under-earmark system debt, allowing borrowers to retain more unearmarked debt and therefore more borrowing headroom than intended, distorting redemption pressure and global accounting in favor of borrowers and against redeemers.
**Proposed mitigation:** Protocol specific.

## [HIGH-09] Buyer fee-rate used for sell trades, causing sellers under/over‑paid and treasury misaccounted
The contract calculates trade fees using the buyer's effective trade-fee rate but subtracts that fee from the seller's proceeds, causing sellers to be charged according to the buyer's tier. In `_executeTokenSwap` and the buy-branches of `_executeAgainstMatcher`, the code computes `paymentAmount`, then `tradeFee = paymentAmount * _getEffectiveTradeFeeRate(buyer) / 10000`, and `netPayment = paymentAmount - tradeFee`, then transfers tradeFee to treasury and netPayment to seller. For a seller-exit the seller should bear the trade fee, but using the buyer's fee-rate misattributes the fee and ties seller receipts to buyer-tier settings.

```solidity
function _executeTokenSwap(...) internal {
    // ...
    uint256 paymentAmount = (fillAmount * sellPrice) / 10000;
    // @audit-issue: Uses buyer's fee rate for seller's trade
    uint256 tradeFee = paymentAmount * _getEffectiveTradeFeeRate(buyer) / 10000;
    uint256 netPayment = paymentAmount - tradeFee;
    
    // Transfer tradeFee to treasury and netPayment to seller
    // ...
}
```

**Impact:** Sellers can receive materially incorrect proceeds and the treasury's fee accounting is unreliable. At scale (e.g., 1,000,000 payment with 150 bpm mismatch), this results in $15,000 incorrect movement.
**Proposed mitigation:** Fix code to charge the seller's rate for sell executions by using `uint256 sellerFeeRate = _getEffectiveTradeFeeRate(sellOrder.user);` and `uint256 tradeFee = (paymentAmount * sellerFeeRate) / 10000;`

# Medium

## [MEDIUM-01] Lack of Discount Percentage Validation
The current implementation of the _getDiscountedPrice() function lacks explicit validation for the discount percentage, allowing potentially invalid or excessive discount percentages to be processed without proper checks. This vulnerability enables the input of discount percentages that could lead to unexpected or mathematically incorrect pricing calculations. The function processes discounts without validating the discount percentage bounds.

```solidity
function _getDiscountedPrice(Listing storage listing, uint256 _amount) private view returns (uint256) {
    uint256 discountedPrice = listing.pricePerUnit;

    if (listing.discountType == DiscountType.LINEAR) {
        // @audit-issue: No validation on discountPct
        discountedPrice = (discountedPrice * (BASE - ((_amount * listing.discountPct) / listing.total))) / BASE;
    } else if (listing.discountType == DiscountType.FIX) {
        // @audit-issue: No validation on discountPct
        discountedPrice = (discountedPrice * (BASE - listing.discountPct)) / BASE;
    }
    return discountedPrice;
}
```

**Impact:** Unchecked discount percentages could result in arbitrary price manipulations, potentially causing financial inconsistencies or unintended pricing behaviors. Malicious actors could manipulate pricing through extreme discount values, potentially setting discount percentages beyond 100% of the base value or producing negative or zero prices.
**Proposed mitigation:** Protocol Specific - add explicit validation for discount percentages to ensure they are within valid bounds (e.g., 0-100%).

## [MEDIUM-02] Incorrectly Flagging Users as Bridged Token Holders
The `_credit` function in the TITN contract is responsible for crediting tokens when received via LayerZero's Omnichain Fungible Token (OFT) mechanism. However, this function incorrectly flags all credited addresses as bridged token holders, regardless of whether the transaction originated from another chain or occurred locally on BASE. According to the protocol's design, non-bridged TITN Tokens on BASE should be freely transferable while bridged TITN Tokens should be restricted until explicitly unlocked. This issue violates the intended behavior by incorrectly restricting local TITN holders on BASE.

```solidity
function _credit(
    address _to,
    uint256 _amountLD,
    uint32 _srcEid
) internal virtual override returns (uint256 amountReceivedLD) { 
    if (_to == address(0x0)) _to = address(0xdead);
    _mint(_to, _amountLD);

    // @audit-issue: Flags all credited addresses as bridged, even local BASE transfers
    if (!isBridgedTokenHolder[_to]) { 
        isBridgedTokenHolder[_to] = true;
    }

    return _amountLD;
}
```

**Impact:** This breaks a key feature by preventing non-bridged TITN holders on BASE from freely transferring their tokens as intended, incorrectly restricting them.
**Proposed mitigation:** Modify _credit to ensure that bridging restrictions only apply to tokens that actually came from another chain by checking `if (_srcEid != baseChainId && !isBridgedTokenHolder[_to])` before flagging users as bridged token holders.

## [MEDIUM-03] JackpotBridgeManager overcharges cross-chain users when ticket price is updated mid-drawing
The `JackpotBridgeManager.buyTickets()` function incorrectly uses the global `ticketPrice` instead of the current drawing's ticket price when charging users. This causes cross-chain users to be overcharged when the owner updates the ticket price during an active drawing, with excess funds becoming stuck in the bridge manager contract. The function fetches the global `ticketPrice` variable from the Jackpot contract (the pending price for the next drawing), but the correct price for the current drawing is stored in `drawingState[currentDrawingId].ticketPrice`.

```solidity
function buyTickets(
    Ticket[] memory tickets,
    address receipient,
    address[] calldata referrers,
    uint256[] calldata referrerSharesOfWinnings,
    bytes32 nonce
) external {
    uint256 currentDrawingId = jackpot.currentDrawingId();
    uint256 ticketPrice = jackpot.ticketPrice(); // @audit-issue: Uses global pending price
    uint256 totalCost = ticketPrice * tickets.length;

    // Line 181: Bridge manager transfers using global price
    usdc.transferFrom(msg.sender, address(this), totalCost);
    // Line 183: Bridge manager approves the same amount to Jackpot
    usdc.approve(address(jackpot), totalCost);
    // Line 184: Jackpot charges using currentDrawingState.ticketPrice (different!)
    jackpot.buyTickets(tickets, receipient, referrers, referrerSharesOfWinnings);
}
```

**Impact:** Direct fund loss as cross-chain users are systematically overcharged when ticket price increases. Excess USDC accumulates in bridge manager with no withdrawal mechanism. Cross-chain ticket purchases completely fail when ticket price decreases (DoS condition). Cross-chain users pay significantly more than direct buyers for identical tickets. Lost funds are permanently inaccessible to users.
**Proposed mitigation:** Modify `JackpotBridgeManager.buyTickets()` to fetch the ticket price from the current drawing state instead of the global variable: `uint256 ticketPrice = jackpot.getDrawingState(currentDrawingId).ticketPrice;`