# During the withdrawAndCall of ERC20 from ZetaChain, user could loose funds
## Description
The user calls withdrawAndCall on the ZEVM chain to withdraw the tokens to EVM compatible chains and make a call to the contract.

Expecting, that the withdrawAndCall will be used to communicate with the contracts on the EVM, it is clear that this function will be heavily used by the protocols that want to execute a cross-chain communication between ZEVM-EVM.

The problem is, that the final step of this cross-chain communication will be executeWithERC20 function, which will

Approve the dst contract to use the tokens
The call will be made to the dst contract
From the function logic it is clear that not all the tokens transferred to the dst contract, and it is purely depends on the execute logic that will be performed in the dst contract

function _execute(address destination, bytes calldata data) internal returns (bytes memory) {
        
        (bool success, bytes memory result) = destination.call{ value: msg.value }(data); 
        if (!success) revert ExecutionFailed();

        return result;
    }
The problem is that if the dst contract will not consume all the amount, the approval will be reset, and remaining balance that is left will be transferred back to the ASSET HANDLER, it means that the left over amount will be lost, instead of being credited back to the user account.

We haven’t found any off chain logic that would add the remainingBalance to the user balance, additionally, from the event emitted it is clear that remainingBalance isn’t passed into the event, so it strength the theory that the remainingBalance will play no role…

It basically means if the user call some contract which implement some strategy in some external protocols, the left over amount would be lost

## Impact Explanation
The left over amount will be lost for the user account.

## Likelihood Explanation
High

## Proof of Concept
Alice calls withdrawAndCall on the ZEVM. The main purpose is call some external protocol on the EVM for different purposes (swaps, lending, liq. staking, options). She deposits 100e18

Once the tx will reach the final step on the EVM, the executeWithERC20 will be triggered, to approve the tokens to the dst and make an external call

It happens that external protocol that is being called hasn’t consumed all amount, and there is a left over of 5e18 tokens.

This tokens are transferred to the ASSET HANDLER back via transferToAssetHandler function instead being credited to the user account.
```
function transferToAssetHandler(address token, uint256 amount) private {
        //check if the token is zeta one, to proceed (burn/transfer) instead
        if (token == zetaToken) { 
            // transfer to connector
            // approve connector to handle tokens depending on connector version (eg. lock or burn)
            if (!IERC20(token).approve(zetaConnector, amount)) revert ApprovalFailed();
            // connector whether burn or transferFrom them.
            ZetaConnectorBase(zetaConnector).receiveTokens(amount);
        } else {
            // transfer to custody
            //check if the token is whitelisted one, prevent from the execution further if it returns false
            if (!IERC20Custody(custody).whitelisted(token)) revert NotWhitelistedInCustody();
            //transfer the tokens to the custody contract.
            IERC20(token).safeTransfer(custody, amount);
        }
    }
```
Result: user loose 5e18 tokens

## Recommendation
Instead of simply transfer the assets back to the handler, implement the right logic, so that user doesn’t loose the tokens, and could claim the left over amount.

-------------

# Attacker can transfer zERC20 from ZEVM to EVM, bypassing gasFee and gasLimit payment
## Relevant Context
There 2 types of withdrawal on the GatewayZEVM:

withdraw → which simply withdraw ZRC20 tokens to an external chain.
withdrawAndCall → which withdraw ZRC20 tokens and call a smart contract on an external chain.
The main difference between them is that during simple withdraw user doesn’t provide any manual gasLimit, because it is calculated automatically based on the type of zrc20 being withdrawn. Meanwhile, during the withdrawAndCall user could provide manual gasLimit, based on what, the final gasFee will be calculated via this line of code.

uint256 gasFee = _withdrawZRC20WithGasLimit(amount, zrc20, gasLimit);
The main problem, that there is no check imposed on the gasLimit, it means that user could provide 0 value, and contract is okay with it.

## Description
The main problem here, is that passing the 0 value as a gasLimit, the user could avoid paying gasFee, and the cross chain call will still proceed.

To calculate the gasFee the following function will be invoked.
```solidity
uint256 gasFee = _withdrawZRC20WithGasLimit(amount, zrc20, gasLimit);
What it basically does is retrieves the gasToken and gasFee, and then it transfer the gasFee to the FUNGIBLE_MODULE_ADDRESS and burns the zrc20 token that user want to withdraw via cross-chain call.

function _withdrawZRC20WithGasLimit(uint256 amount, address zrc20, uint256 gasLimit) internal returns (uint256) {
        (address gasZRC20, uint256 gasFee) = IZRC20(zrc20).withdrawGasFeeWithGasLimit(gasLimit);
        if (!IZRC20(gasZRC20).transferFrom(msg.sender, FUNGIBLE_MODULE_ADDRESS, gasFee)) {
            revert GasFeeTransferFailed();
        }
        if (!IZRC20(zrc20).transferFrom(msg.sender, address(this), amount)) {
            revert ZRC20TransferFailed();
        }
        if (!IZRC20(zrc20).burn(amount)) revert ZRC20BurnFailed();
        return gasFee;
    }
```
The critical point here, is that our “0 value” gasLimit is passed to the withdrawGasFeeWithGasLimit in the first line.
```solidity
function withdrawGasFeeWithGasLimit(uint256 gasLimit) public view override returns (address, uint256) {
        address gasZRC20 = ISystem(SYSTEM_CONTRACT_ADDRESS).gasCoinZRC20ByChainId(CHAIN_ID); 
        if (gasZRC20 == address(0)) revert ZeroGasCoin(); //check that it isn't 0
        uint256 gasPrice = ISystem(SYSTEM_CONTRACT_ADDRESS).gasPriceByChainId(CHAIN_ID); //it retrieves the gasPrice @audit hardcoded one
        if (gasPrice == 0) { //ensure it is not 0
            revert ZeroGasPrice();
        }   
        uint256 gasFee = gasPrice * gasLimit + PROTOCOL_FLAT_FEE; //PROTOCOL_FLAT_FEE is 0. 
        return (gasZRC20, gasFee);
    }
```
As we can see that during the final calculation of the gasFee the gasPrice that is retrieved from the SYSTEM_CONTRACT_ADDRESS will be multiplied by our (user provided) gasLimit, which in our case is 0. It also worth to mention that sponsor confirmed that PROTOCOL_FLAT_FEE will remain 0.

*Lucas: This value is set to 0 and will be removed in the future to just keep the regular fee. from discord chat

So, we end up with gasFee being 0! What next? The 0 value of the gasZRC20 will be transferFrom the user to the FUNGIBLE_MODULE_ADDRESS , it will proceed with no problem.

Eventually, the Withdrawn even will be emited with the gasLimit being 0 as well as gasFee being 0.

Let’s take a look on the off-chain node, to be make sure that this attack is valid, especially at the v2_zevm_inbound.go at the func (k Keeper) newWithdrawalInbound(…) , here we could see very interesting thing.
```go
gasLimit := event.GasLimit.Uint64()
	if gasLimit == 0 {
		gasLimitQueried, err := k.fungibleKeeper.QueryGasLimit(
			ctx,
			ethcommon.HexToAddress(foreignCoin.Zrc20ContractAddress),
		)
		if err != nil {
			return nil, errors.Wrap(err, "cannot query gas limit")
		}
		gasLimit = gasLimitQueried.Uint64()
	}
```
What it basically does, is say “if the gasLimit is 0, take the default value for this token and move forward with this value”.

If we take a look on this table, we could see that for most of the tokens the default gasLimit is 100k https://zetachain.blockpi.network/lcd/v1/public/zeta-chain/fungible/foreign_coins , based on this number we will play right now.

Our initial approach fails around this vulnerability fails, because if the function will use 100k gas for USDC token on Ethereum for example, it would still be not enough to execute the withdrawAndCall from the ERC20Custody , since 2 token transfers, one external call, 3 approvals consume significantly more gas.

That’s why we we decided to change the strategy.

Actual Bug

When we initiate a withdrawAndCall on the ZEVM what we can do, is to provide

gasLimit = 0 (we have explained it above)
message.length = 0
The problem is, that if we provide message.length 0 (basically no message), in the v2_deposits.go during the ProcessV2Deposit the following will be executed
```go
func (k Keeper) ProcessV2Deposit(
	ctx sdk.Context,
	from []byte,
	senderChainID int64,
	zrc20Addr ethcommon.Address,
	to ethcommon.Address,
	amount *big.Int,
	message []byte,
	coinType coin.CoinType,
) (*evmtypes.MsgEthereumTxResponse, bool, error) {
	context := systemcontract.ZContext{
		...
	}
	if len(message) == 0 {
		// simple deposit
		res, err := k.DepositZRC20(ctx, zrc20Addr, to, amount)
		return res, false, err
If the len(message) == 0, the simple DepositZRC20 on the EVM chain will be executed, which would allow us to make a deposit bypassing paying the gasLimit/gasFee. Since we make the pure DepositZRC20 the 100k gasLimit will be enough to make a safeTransfer.
```
## Impact Explanation
User can withdraw ZRC20 tokens from the ZEVM and pay no fees and gasLimit, which would result in the protocol loss.

## Likelihood Explanation
High. User can simply call withdrawAndCall and provide gasLimit as 0 , as well as message as .length 0.

## Proof of Concept
Malicious user want to withdraw the tokens and without paying the fees and gas on the dst chain.

he calls withdrawAndCall on the zevm and provide gasLimit as 0 and bytes calldata message as 0.

the offchain node will grap the tx and see that gasLimit is 0, here the node will use the default gasLimit for the foreignToken → if it is USDC the gasLimit used will be 100_000 according to it (https://zetachain.blockpi.network/lcd/v1/public/zeta-chain/fungible/foreign_coins)

the offchain node during the outbound processing will see that the len(message) == 0 , thus he will route the tx to the function withdraw in the ERC20Custody, instead of function withdrawAndCall

The 100k will be enough for IERC20.transfer, since the average gas consumption during the transfer is 60k, you could prove it by taking a look on this tx or simple PoC

https://etherscan.io/tx/0x307666c9b4c5b0c66f7d7f44eb0e97446a5dc84e5868b7c7baeb011e1adf5062

//forge test --match-path test/ERC20Custody.t.sol --match-test test_EastEuropeHatsAudit -vvvv
    function test_EastEuropeHatsAudit() public{
        uint256 amount = 100;
        vm.prank(tssAddress);
        custody.withdraw(address(receiver), address(token), amount);
    }
## Recommendation
Make the check that during the withdrawAndCall user can’t provide 0 value as a gasLimit, as well as message.length = 0

--------------
# WithdrawAndCall / Call methods can be initiated for Solana/Bitcoin chains, always leading to reverts/refunds#621

## Summary
Since Zeta supports / will support deposits/withdrawals from/to Bitcoin and Solana, besides EVM-compatible chains, there is a problem which involves the newly-added arbitrary call option.

Since only EVM-compatible chains can support arbitrary calls, initiating a call / withdrawAndCall() from Zeta → connected chain, is plausible and will be successfully emitted.

The problem is that this call would revert an offchain level and lead to reverts/refunds that could’ve been prevented immediately with chainId whitelist / checks.

Finding Description
With V2 contracts, Zeta are introducing arbitrary calls to smart contracts. Those arbitrary calls can be initiated from an external/connected chain with a destination on Zeta OR they can be initiated from Zeta, with the destination set as the external/connected chain.

The problem is, that when calls are initiated from Zeta, they can only be performed on EVM-compatible chains (e.g. BSC, Polygon, etc.), while Zeta supports deposits from/to BTC and SOL as well.

When we initiate a withdrawAndCall() or just call() with either BTC or SOL / SOL-based token:
```solidity
    function withdrawAndCall(
        bytes memory receiver,
        uint256 amount,
        address zrc20,
        bytes calldata message,
        uint256 gasLimit,
        RevertOptions calldata revertOptions
    )
        external
        nonReentrant
        whenNotPaused
    {
        if (receiver.length == 0) revert ZeroAddress();
        if (amount == 0) revert InsufficientZRC20Amount();

        uint256 gasFee = _withdrawZRC20WithGasLimit(amount, zrc20, gasLimit);
        emit Withdrawn(
            msg.sender,
            0,
            receiver,
            zrc20,
            amount,
            gasFee,
            IZRC20(zrc20).PROTOCOL_FLAT_FEE(),
            message,
            gasLimit, 
            revertOptions
        );
    }
```
The event will be emitted with a message/payload, caught by the observer. The problem is that this call will inevitably revert due to the SOL gateway not supporting arbitrary calls, or in the case of BTC, this not being an option at all.

## Impact Explanation
withdrawAndCall() / call() methods should allow only EVM-based chains for withdrawal (ZRC20 tokens), as SOL-based and BTC would always revert. This would lead to additional costs in terms of gas for reverts/refunds.

## Likelihood Explanation
High, as anyone can initiate withdraw / withdraw + arbitrary call transactions with BTC / SOL.

## Proof of Concept (if required)
User initiates a withdrawAndCall transaction with a SOL-based token.
Inserts arbitrary call data in the message field so that the offchain node recognizes this transaction as a withdrawAndCall instead of a plain withdraw.
Once the transaction is finalized on the inbound side, it will revert when it reaches the outbound as arbitrary calls on Solana or Bitcoin are not plausible.
This will create an additional need / cost for the Zeta protocol to revert and refund the assets to the user.
This can also be abused by malicious user to create unnecessary revert costs.
## Recommendation (optional)
Make sure that the chainId is validated when call / withdrawAndCall are invoked so that these kind of situations can be resolved in the beginning.


-------------

# Already deployed ZRC20 tokens are incompatible with V2 Gateways
## Finding Description
Already deployed ZRC20 tokens by the ZetaChain team which can be found here: https://www.zetachain.com/docs/developers/tokens/zrc20/ will be incompatible with the V2 Gateways due to:

withdrawGasFeeWithGasLimit() function invoked in two instances on the gateway and is non-existent in the ZRC20 which are currently deployed.
And due to a condition in the already deployed ZRC20 tokens’ deposit() function which is access-controlled and can only be called by either the system contract OR the fungible module address, but not the gateway.
First and foremost, it should be noted that ZRC20 tokens are not upgradeable, and there are already 13 deployed tokens, like USDC.BSC, USDC.ETH, ETH.ETH, etc.

As an example here’s the contract for USDC.ETH deployed on ZetaChain:

0x0cbe0dF132a6c6B4a2974Fa1b7Fb953CF0Cc798a

The first problem is that the V1 version of the ZRC20 contract doesn’t contain the withdrawGasFeeWithGasLimit() function which can be seen in the V2 version of the ZRC20 contract which will be used in the contracts that will be used by the to-be deployed ZRC20 tokens:
```solidity
function withdrawGasFeeWithGasLimit(uint256 gasLimit) public view override returns (address, uint256) {
        address gasZRC20 = ISystem(SYSTEM_CONTRACT_ADDRESS).gasCoinZRC20ByChainId(CHAIN_ID);
        if (gasZRC20 == address(0)) revert ZeroGasCoin();

        uint256 gasPrice = ISystem(SYSTEM_CONTRACT_ADDRESS).gasPriceByChainId(CHAIN_ID);
        if (gasPrice == 0) {
            revert ZeroGasPrice();
        }
        uint256 gasFee = gasPrice * gasLimit + PROTOCOL_FLAT_FEE;
        return (gasZRC20, gasFee);
    }
```
There are two instances in the V2 ZEVM Gateways that invoke this function. The first one is during the call() method, invoked directly:

```solidity
 function call(
        bytes memory receiver,
        address zrc20,
        bytes calldata message,
        uint256 gasLimit,
        RevertOptions calldata revertOptions
    )
        external
        nonReentrant
        whenNotPaused
    {
        if (receiver.length == 0) revert ZeroAddress();
        if (message.length == 0) revert EmptyMessage();

        (address gasZRC20, uint256 gasFee) = IZRC20(zrc20).withdrawGasFeeWithGasLimit(gasLimit);
        if (!IZRC20(gasZRC20).transferFrom(msg.sender, FUNGIBLE_MODULE_ADDRESS, gasFee)) {
            revert GasFeeTransferFailed();
        }

        emit Called(msg.sender, zrc20, receiver, message, gasLimit, revertOptions);
    }
```
And the second one, whenever _withdrawZRC20WithGasLimit() is called:

```solidity
    function _withdrawZRC20WithGasLimit(uint256 amount, address zrc20, uint256 gasLimit) internal returns (uint256) {
        (address gasZRC20, uint256 gasFee) = IZRC20(zrc20).withdrawGasFeeWithGasLimit(gasLimit);
        if (!IZRC20(gasZRC20).transferFrom(msg.sender, FUNGIBLE_MODULE_ADDRESS, gasFee)) {
            revert GasFeeTransferFailed();
        }

        if (!IZRC20(zrc20).transferFrom(msg.sender, address(this), amount)) {
            revert ZRC20TransferFailed();
        }

        if (!IZRC20(zrc20).burn(amount)) revert ZRC20BurnFailed();

        return gasFee;
    }
```
The above mentioned internal function is called in both withdrawAndCall() and in withdraw() through the _withdrawZRC20():
```solidity
    function _withdrawZRC20(uint256 amount, address zrc20) internal returns (uint256) {
        // Use gas limit from zrc20
        return _withdrawZRC20WithGasLimit(amount, zrc20, IZRC20(zrc20).GAS_LIMIT());
    }
```

The second instance which will cause the already deployed ZRC20 tokens to be incompatible with the new V2 gateways is the deposit, the difference is that now it’s planned for the gateways to call the ZRC20 tokens’ deposit() function directly, thus we have the following condition in the new contracts:
```solidity
 function deposit(address to, uint256 amount) external override returns (bool) {
        if (
            msg.sender != FUNGIBLE_MODULE_ADDRESS && msg.sender != SYSTEM_CONTRACT_ADDRESS
                && msg.sender != gatewayAddress
        ) revert InvalidSender();
```
While the already deployed ZRC20 contracts have the following access control condition:
```solidity
function deposit(address to, uint256 amount) external override returns (bool) {
        if (msg.sender != FUNGIBLE_MODULE_ADDRESS && msg.sender != SYSTEM_CONTRACT_ADDRESS) revert InvalidSender();
Since they can only be called by either the system contract or the fungible module address, if the gateway tries to call the deposit function (whenever a new ZRC20 token is deposited from a spoke chain to Zeta), this function will revert.
```
## Impact Explanation
All 13 already-deployed ZRC20 tokens (the list for which) can be found here:

https://www.zetachain.com/docs/developers/tokens/zrc20/ will be incompatible with the new V2 Gateways.

## Likelihood Explanation
This will occur every time when Gateways try to interact with the already deployed ZRC20 tokens either due to the missing withdrawGasFeeWithGasLimit() function on the V1 ZRC20 contracts or due to access-control restrictions in the deposit() function.

## Proof of Concept (if required)
A user initiates a deposit from a spoke/connected chain (BNB, ETH, Polygon, etc.)
The offchain node processes the inbound transaction, then the outbound transaction is picked up by Zeta and processed.
When deposit or depositAndCall is called on the ZEVM Gateway, the transaction would inevitably revert due to the above-mentioned inconsistencies.
Recommendation (optional)
Implement a different way to interact with the already deployed ZRC20 tokens through the gateways which will be compatible.


