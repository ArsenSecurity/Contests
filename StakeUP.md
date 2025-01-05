# In _closeMatchOrders, if the amount to remove is half of the collateral, the match would be deleted which will lead to fund loss#

### Relevant Context

When a lender decides to kill a portion of their “matched” orders, they can specify an amount for which they would want their orders un-matched and capital returned to them. The problem arises in a specific case in which the amount to remove/kill is exactly half that of the collateral in-question, this will cause the whole match to be “unrightly” killed and half of the amount lost.

### Risks Involved
When a lender decides to “kill” an order already matched by a borrower, they can do so via killMatchOrder , the problem arises, if the amount that a lender wants to “kill” is exactly half that of their collateral.
```solidity
 function killMatchOrder(uint256 amount) external returns (uint256 totalRemoved) {
        _amountZeroCheck(amount);
        // if the order is already matched we have to account for the borrower's who filled the order.
        // If you kill a match order and there are multiple borrowers, the order will be closed in a LIFO manner.
        totalRemoved = _closeMatchOrders(msg.sender, amount);
        IERC20(_asset).safeTransfer(msg.sender, totalRemoved);
    }
This is because in _closeMatchOrders :

 function _closeMatchOrders(address account, uint256 amount) internal returns (uint256 totalRemoved) {
        MatchOrder[] storage matches = _userMatchedOrders[account];
        uint256 remainingAmount = amount;

        uint256 length = matches.length;
        for (uint256 i = length; i != 0; --i) {
            uint256 index = i - 1;

         ...

            if (remainingAmount != 0) {
                uint256 amountToRemove = Math.min(remainingAmount, matches[index].lCollateral);
                uint256 borrowAmount = uint256(matches[index].bCollateral);

                if (amountToRemove != matches[index].lCollateral) {
                    borrowAmount = amountToRemove.divWad(_leverage);
                    matches[index].lCollateral -= uint128(amountToRemove);
                    matches[index].bCollateral -= uint128(borrowAmount);
                }
                remainingAmount -= amountToRemove;
                _idleCapital[matches[index].borrower] += borrowAmount;

                emit MatchOrderKilled(account, matches[index].borrower, amountToRemove);
                if (matches[index].lCollateral == amountToRemove) matches.pop();
            } else {
                break;
            }
        }
        totalRemoved = amount - remainingAmount;
        _matchedDepth -= totalRemoved;
    }
```
The following line of code will unjustly remove the whole match, if the lCollateral is equal to the amountToRemove but in a scenario where the amountToRemove is exactly half of the collateral this will hold true.

This is because prior to this in the if-block which executes logic if the amountToRemove is not equal to the matches[index].lCollateral , the matches[index].lCollateral will get decreased for the said amount.

If the amount we want removed is exactly half that of the collateral, and after the collateral is decreased for that amount, the following condition will become true, and the match would be unjustly removed without a possibility for both the lender and the borrower to access their funds afterwards:
```
if (matches[index].lCollateral == amountToRemove) matches.pop();
```
## Impact Explanation
High - As it will cause fund loss for both the lender/borrower without an alternative way to recover the lost funds.

## Likelihood Explanation
High - This can be triggered anytime the amount the user wants to remove is exactly half of the collateral.

## Proof of Concept
Imagine the following scenario:

Alice is a lender with 1 matched order with the following stats:
matches[index].lCollateral= 10_000 USDC
Bob is the borrower which matched Alice’s order with a bCollateral amount of exactly 2_000 USDC due to the 5x leverage.
Alice wants to kill half of their matched order and calls killMatchOrder with an amount of 5_000;
There is only 1 matched order, and the following logic will take place:
```solidity
if (amountToRemove != matches[index].lCollateral) {
                    borrowAmount = amountToRemove.divWad(_leverage);
                    matches[index].lCollateral -= uint128(amountToRemove);
                    matches[index].bCollateral -= uint128(borrowAmount);
                }
```
The above will hold true as the amountToRemove which is 5_000, does not equal the collateral of the matches in this case.
The above logic will decrease the lCollateral for the amount of 5_000, and the bCollateral for the amount of 1_000, considering the leverage.
Going further, the following condition will become true:
if (matches[index].lCollateral == amountToRemove) matches.pop();
This is because the lCollateral which is now 5_000 (after it was decreased), will equal the amountToRemove , and the match will be permanently deleted, causing fund loss for both the lender and the borrower.
## Recommendation
Only “pop” the match if both the collateral and borrow amount equal 0.

------------------


# Ordered/Sequential LayerZero delivery can be maliciously manipulated leading to an outdated USD per share / timestamp
## Finding Description
There are two pre-conditions that make this attack vector plausible:

The poke() function can be called by anyone with arbitrary LZSettings;
They’re bridging the information in an ordered manner, i.e. the messages have to follow a sequential nonce and they can’t be executed out-of-order;
There are multiple ways in which a malicious user can cause the message to get stuck before its executed on the destination, and some of those include:

Calling poke with less gas than needed and the admin having to retry the message later;
ExecutorOrderedExecutionOption not being passed in options leading to contract level reverts on the destination and a message which can’t be executed

poke() is a crucial functionality within the stUSDC ecosystem as the usd per share information is dependent on its timely updates.

If the usd per share information is not timely updated, and this information is outdated the share calculation when minting/redeeming stUsdc will be outdated leading to potential losses for either the users or the protocol.

According to LayerZero documentation: https://docs.layerzero.network/v2/developers/evm/oapp/message-design-patterns

“If you do not pass an ExecutorOrderedExecutionOption in your _lzSend call, the Executor will attempt to execute the message despite your application-level nonce enforcement, leading to a message revert.”

There are no conditions which will cause this to revert on the source-chain side, so the revert will occur on the destination, i.e. after the message was sent from the source chain.

Current logic doesn’t enforce the ExecutorOrderedExecutionOption in the _lzSend call, but its up to the user which calls poke() to specify LZ options:
```solidity
 function poke(LzSettings calldata settings) external payable nonReentrant {
 
 ...
 
 require(msg.value >= lzFee, Errors.InvalidMsgValue());
            _keeper.sync{value: lzFee}(
                newUsdPerShare, currentTimestamp, _peerEids, settings.options, settings.refundRecipient
            );
            
            ...
            
      // Harvest matured TBYs and distribute rewards
        _harvest();
        _distributePokeRewards();
    }
```
The problem is since ordered execution is enforced, and the destination side execution logic has to follow a strict sequential nonce, this message will become non-executable.

The owner/admin then has to use skip - “The skip function should be used only in instances where either message verification fails or must be stopped, not message execution. LayerZero provides separate handling for retrying or removing messages that have successfully been verified, but fail to execute.”

This could cause delays in the usd per share information on other chains.

Another plausible malicious meddling into how messages are executed, could be passing a lower gas amount than needed to be executed on destination, causing again the need for admins/owners to have to debug messages as per:

https://docs.layerzero.network/v2/developers/evm/troubleshooting/debugging-messages

Since this is easily manipulable by whoever calls the poke() method once every 24 hours, if the usd share per message update is significant, malicious users can purposefully delay its update to leverage and exploit the outdated share price on destination for their gain.

There is a DoS element to this as well, since if the admin team doesn’t retry/skip/clear the payload within 24 hours, and another poke() call is executed, the next poke() call will be DoS’d as well due to the enforced sequential nonce order.
## Proof of Concept
Malicious user purposefully doesn’t pass the required/needed options in the poke() method;
This will cause for the usd per share update to be delayed and DoS’d until the admin team releases/skips the message and if that’s not done within 24 hours for any subsequent poke() calls (which can be legitimate) to be DoS’d as well.
The malicious user can then benefit from the outdated usd per share price on destination.

## Impact Explanation
Either limit poke() to be a “permissioned” method callable by a protocol-controlled bot or enforce the required options in the lzSend call (hardcode them).



------------------

# Users which have deposited assets for stUSDC can be DoSd from redeeming, allowing Tby holders to bypass maturity time
## Finding Description
Both users which deposit USDC(asset) and Tbys, mint stUSDC in exchange which can be redeemed immediately:
```solidity
 function depositAsset(uint256 amount) external nonReentrant returns (uint256 amountMinted) {
        ...
        _deposit(amountMinted);
        ...
        _openLendOrder(amount);
    }
```
As well as the Tby deposit functionality:
```solidity
function depositTby(uint256 tbyId, uint256 amount) external nonReentrant returns (uint256 amountMinted) {
        amountMinted = _calculateTbyMintAmount(pool, tbyId, amount);
        _deposit(amountMinted);
        ...
        _tby.safeTransferFrom(msg.sender, address(this), tbyId, amount, "");
    }
```
The stUsdc balance is minted during the _deposit function logic:
```solidity
function _deposit(uint256 amount) internal {
        uint256 sharesAmount = sharesByUsd(amount);
        if (sharesAmount == 0) revert Errors.ZeroAmount();

        _mintShares(msg.sender, sharesAmount);
        _globalShares += sharesAmount;
        _setTotalUsdFloor(_totalUsdFloor + amount);
    }
```
Since the user’s shares were minted in both cases in the form of stUsdc, they can be immediately redeemed for USDC.
```solidity
 function redeemStUsdc(uint256 amount) external nonReentrant returns (uint256 assetAmount) {
        require(amount > 0, Errors.ZeroAmount());
        require(balanceOf(msg.sender) >= amount, Errors.InsufficientBalance());

        uint256 shares = sharesByUsd(amount);
        assetAmount = amount / _scalingFactor;

        uint256 assetBalance = _asset.balanceOf(address(this));
        if (assetBalance < assetAmount) {
            uint256 amountNeeded = assetAmount - assetBalance;
            _tryOrderCancellation(amountNeeded);
            uint256 newAssetBalance = _asset.balanceOf(address(this));
            require(newAssetBalance >= assetAmount, Errors.InsufficientBalance());
        }

        _burnShares(msg.sender, shares);
        _setTotalUsdFloor(_totalUsdFloor - amount);
        _globalShares -= shares;

        emit Redeemed(msg.sender, shares, assetAmount);
        _asset.safeTransfer(msg.sender, assetAmount);
    }
```
The first problem which this causes is once the Tbys are minted to the users, they can be immediately redeemed for USDC (while they receive stakeUp rewards), by depositing Tby and then redeeming the stUSDC for USDC. Which basically beats defeats the protocol’s core invariable of having to wait for the maturity period to end.

This can even be more profitable then waiting for the RWA’s price to appreciate due to the StakeUp rewards (depending on the stakeUp token’s price).

The second issue is that Tby holders can DoS the asset holders from redeeming their stUsdc for USDC due to “emptying out” the USDC reserves.

## Proof of Concept
depositAsset → user_1 deposits 1000e6 usdc, which are converted into 1000e18;

_openLendOrder is created. *for simplicity assume it isn’t filled for now.
depositTby → user_2 deposits 1000e6 tby, which are converted into 1000e18; (Assume a 1:1 conversion ratio as well, for simplicity);

redeemStUsdc → user_2 decides to redeem all their stUsdc, after depositing the Tby so it is 1000e6 usdc, however, the contract doesn’t have enough usdc to redeem, so the if condition is entered:
```solidity
if (assetBalance < assetAmount) { 
            uint256 amountNeeded = assetAmount - assetBalance;
            _tryOrderCancellation(amountNeeded);
            uint256 newAssetBalance = _asset.balanceOf(address(this));
            require(newAssetBalance >= assetAmount, Errors.InsufficientBalance());
        }
After killing the open orders, it will transfer the 1000 USDC to the Tby depositor.
```
Now if user_1 once to redeem their stUsdc for USDC, it won’t be possible
## Impact Explanation
Users which have deposited assets, can’t redeem their stUsdc back for the asset.



------------------

# Malicious users can depositTby at outdated prices utilizing cross-contract reentrancy
## Finding Description
Malicious users can utilize the ERC1155 callback to maliciously re-enter the StakeUp contract (cross-contract reentrancy) and deposit Tbys at outdated prices (i.e. non-normalized prices).
## Proof of Concept
A user which opened a lend order (and was later matched) could perform the following cross-contract reentrancy attack, which would allow them to basically deposit (and/or redeem) Tbys at outdated and non-normalized prices.

When a swapIn call is being performed, it will convert the matched orders into Tbys:
```solidity
  for (uint256 i = 0; i != len; ++i) {
            uint256 amountUsed = _convertMatchOrders(id, accounts[i], assetAmount);
            assetAmount -= amountUsed;
            amountSwapped += amountUsed;
            if (assetAmount == 0) break;
        }
```
When they include the order from the malicious lender, it will call the _checkOnERC1155Received hook when minting:
```soliity
    if (amountUsed != 0) {
            _idToTotalBorrowed[id] += borrowerAmountConverted;
            uint256 totalLenderFunds = amountUsed - borrowerAmountConverted;
            _matchedDepth -= totalLenderFunds;
            _tby.mint(id, account, totalLenderFunds);
        }
```
This will allow the malicious contract to re-enter the system by entering the StakeUp contract (since the StakeUp contract’s non-reentrancy modifiers are only valid for same-contract calls);

Once they’ve “entered” the system, they can call depositTby :
```soliity
 function depositTby(uint256 tbyId, uint256 amount) external nonReentrant returns (uint256 amountMinted) {
        IBloomPool pool = _bloomPool;
        require(amount > 0, Errors.ZeroAmount());
        require(!pool.isTbyRedeemable(tbyId), Errors.RedeemableTbyNotAllowed());

        amountMinted = _calculateTbyMintAmount(pool, tbyId, amount);

        _deposit(amountMinted);
        // Calculate & mint SUP mint rewards to users.
        _mintRewards(pool, tbyId, amount);

        emit TbyDeposited(msg.sender, tbyId, amount, amountMinted);
        _tby.safeTransferFrom(msg.sender, address(this), tbyId, amount, "");
    }
```
The amount of stUsdc which would need to be minted is calculated using the _calculateTbyMintAmount :
```soliity
    function _calculateTbyMintAmount(IBloomPool pool, uint256 tbyId, uint256 amount) internal view returns (uint256) {
        uint256 tbyStart = pool.tbyMaturity(tbyId).start;
        uint256 rate = pool.getRate(tbyId); 
        uint256 timeElapsed = block.timestamp - tbyStart;
        if (timeElapsed > Constants.ONE_DAY) {
            if (rate > Math.WAD) {
                uint256 adjustedRate =
                    Math.WAD + ((rate - Math.WAD).mulWad(timeElapsed - Constants.ONE_DAY).divWad(timeElapsed));
                return amount.mulWad(adjustedRate) * _scalingFactor;
            }
            return amount.mulWad(rate) * _scalingFactor;
        }
        return (rate >= Math.WAD) ? amount * _scalingFactor : amount.mulWad(rate) * _scalingFactor;
    }
```
Since the amount minted is highly dependent on the rate, the rate is highly dependent on the price:
```soliity
    function getRate(uint256 id) public view override returns (uint256) {
        TbyMaturity memory maturity = _idToMaturity[id];
        RwaPrice memory rwaPrice = _tbyIdToRwaPrice[id];

        if (rwaPrice.startPrice == 0) {
            revert Errors.InvalidTby();
        }
        if (block.timestamp <= maturity.start) {
            return Math.WAD;
        }
    
        uint256 price = rwaPrice.endPrice != 0 ? rwaPrice.endPrice : _rwaPrice();
        uint256 rate = (uint256(price).divWad(uint256(rwaPrice.startPrice))); 
        return _takeSpread(rate, rwaPrice.spread);
    }
```
And the price is normalized, after the orders are converted in the swapIn call, this would allow the lender to capitalize on the non-normalized prices:
```soliity
 function _normalizePrice(uint256 startPrice, uint256 currentPrice, uint256 amount, uint256 existingCollateral)
        private
        pure
        returns (uint128)
    {
        uint256 totalValue = (existingCollateral.mulWad(startPrice) + amount.mulWad(currentPrice));
        uint256 totalCollateral = existingCollateral + amount; 
        return uint128(totalValue.divWad(totalCollateral)); 
    }
```
The “malicious” user could also redeem the stUsdc for USDC immediately (in the same transaction) while harvesting the rewards, depending on what their goal is, effectively with this harvesting the rewards, and gaining all of their USDC back.
## Impact Explanation
They are able to deposit the Tbys at non-normalized prices, as well as redeem stUsdc for USDC in the same transaction.

------------------

# DoS on the swapIn execution via _checkOnERC1155Received
## Finding Description
swapIn could be DoS’d by malicious lender who could waste gas / revert when the Tbys are being minted to them.

This is plausible because once the ordered is filled we would mint the Tby token to the lender, which is ERC1155, and upon the minting, the _checkOnERC1155Received ”hook” is checked.

This means, if the receiver is the smart contract which doesn’t implement the OnERC1155Received the whole batch swapIn of the orders will revert, causing the DoS or a malicious contract which could intentionally waste gas;
```solidity
function swapIn(address[] memory accounts, uint256 assetAmount)
        external
        KycMarketMaker
        nonReentrant
        returns (uint256 id, uint256 amountSwapped)
    {
        id = _handleTbyId();

        // Iterate through the accounts and convert the matched orders to live TBYs.
        uint256 len = accounts.length;
        for (uint256 i = 0; i != len; ++i) {
            uint256 amountUsed = _convertMatchOrders(id, accounts[i], assetAmount);
            assetAmount -= amountUsed;
            amountSwapped += amountUsed;
            if (assetAmount == 0) break;
        }
```
The _convertMatchOrders function will further call _tby.mint which will check the _checkOnERC1155Received if the address is a smart contract (i.e. if the address provided contains code).

This allows malicious or just unbeknownst users to perform a DoS of the swapIn contract either intentionally or unintentionally, or just “waste gas”, i.e. gas bombs from a large return data.

## Impact Explanation
The swapIn call can be DoS’d or it will/can lead to gas theft of the market maker by wasting gas when the mint is performed.



------------------

# In case of a leverage increase/decrease multiple problems can arise leading to stuck funds
## Finding Description
The owner has the ability to set and modify the leverage, depending on the “discount” that they want to give to borrowers. This can be done through the setLeverage method.

Admin/owner can adjust the leverage depending on economical incentives, capital availability, locked funds, etc. depending on the current situation within the protocol, as well as the wider macroeconomic situation.

The problem which will arise is if said leverage is either increased or decreased. This could/would lead to stuck funds and orders which can’t be killed.

Leverage is a key concept in the protocol logic, that allows borrower to fill up the order by providing less funds then the lender, i.e. they don’t have to fully match the lender’s order.

The initial leverage can be anything up to 100x (99.9e18) but likely will be in the range from 20e18 - 50e18.

Problems could arise once admin team decides to change the initial leverage (due to external economic factors for example) through setLeverage to a lesser value (from 20e18 to 10e18 for example), this would cause a portion of matched orders to not be able to be closed.

```solidity
function _setLeverage(uint256 leverage) internal {
        require(leverage >= Math.WAD && leverage < MAX_LEVERAGE, Errors.InvalidLeverage());
        _leverage = leverage;
        emit LeverageSet(leverage);
    }
```
This is due to how the calculations in the _closeMatchOrders are performed:

```solidity
function _closeMatchOrders(...) public{
	if (amountToRemove != matches[index].lCollateral) { //if the amountToRemove isn't equal to the collateral of the specific order
                    borrowAmount = amountToRemove.divWad(_leverage); //calc the borrow amount based on the provided amountToRemove
                    matches[index].lCollateral -= uint128(amountToRemove); //subs. the amountToRemove
                    matches[index].bCollateral -= uint128(borrowAmount); //subs. the newly calculated borrowAmount
                }
}
```
So, assuming the initial leverage value was 20e18.

Taking as an example a collateral value of 100 USDC:

lCollateral = 100e6 (100 USDC)
bCollateral = 100e6 * 1e18 / 20e18 = 5e6 (5 USDC)
However, once leverage is changed to 10e18 for example , if the user or the stUsdc contract(due to users redeeming stUsdc) would try to killMatchOrder the following scenario would occur
```solidity
if (amountToRemove != matches[index].lCollateral) { 
      borrowAmount = amountToRemove.divWad(_leverage); 
      matches[index].lCollateral -= uint128(amountToRemove); 
      matches[index].bCollateral -= uint128(borrowAmount);
}
```
borrowAmount would be calculated as following → 100e6 * 1e18 / 10e18 = 10e6
lCollateral -= 100e6
bCollateral -= 10e6 → This would revert, because our bCollateral was 5e6;
This effectively DoS’s the matched order so it can’t be killed.

A scenario in which the leverage was increased, for example:

borrowAmount would be calculated as following → 100e6 * 1e18 / 40e18 = 2.5e6
lCollateral -= 100e6
bCollateral -= 2.5e6 → This would cause for the borrower to release only half the funds, while the other half remains “stuck”.


## Impact Explanation
Leverage can’t be changed once its initially set as this would cause stuck funds and users will be unable to functionally kill their matched orders;




