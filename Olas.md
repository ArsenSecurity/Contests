# Incorrect callValueRefundAddress in createRetryableTicket function leads to cross-chain message not delivered

## Lines of code
https://github.com/code-423n4/2024-05-olas/blob/3ce502ec8b475885b90668e617f3983cea3ae29f/tokenomics/contracts/staking/ArbitrumDepositProcessorL1.sol#L189-L191


## Vulnerability Detail
The createRetryableTicket delivers/propagate the Information to the Arbitrum L2 about targets, stakingIncentives. However, incorrect callValueRefundAddress is set, which could result in the information not to be delivered and lost.

According to the Technical documentation we see that during the createRetryableTicket following params should hold:

excessFeeRefundAddress: The L2 address to which the excess fee is
credited. This is going to be set as the Alias Timelock address on Arbitrum
callValueRefundAddress: The L2 address to which the l2CallValue is
credited if the ticket times out or gets cancelled. This is going to be set as
the Alias Timelock address on Arbitrum
However, both these params are controlled by the user, who initiated the tx. These params are encoded in the bridgePayload and right before cross-chain transfer they are decoded to be passed into the the createRetryableTicket function. Let’s take a look

// Decode the staking contract supplemental payload required for bridging tokens
        (address refundAccount, uint256 gasPriceBid, uint256 maxSubmissionCostToken, uint256 gasLimitMessage,
            uint256 maxSubmissionCostMessage) = abi.decode(bridgePayload, (address, uint256, uint256, uint256, uint256));

        // If refundAccount is zero, default to msg.sender
        if (refundAccount == address(0)) {
            refundAccount = msg.sender;
        }
So, from the bridgePayload, the refundAccount could be any address, if it is set to address(0) , it is set to msg.sender.

According to the Arbitrum docs, the callValueRefundAddress (which is set to any arbitrary address in our case) got a critical permission to cancel the ticket.

## Proof of Concept
Malicious user calls claimStakingIncentives fn in the Dispenser.sol with malicious address encoded into the bridgePayload parameter.

calculation of staking incentives happens + Refund returned amount back to tokenomics inflation

The internal fn _distributeStakingIncentives is called which will execute the cross-chain tx.

Following the tx flow we reach the ArbitrumDepositProcessorL1, the _sendMessage function. At this step the bridgePayload is decoded with malicious refundAccount

createRetryableTicket function doesn’t differentiate between excessFeeRefundAddress and callValueRefundAddress and simply use refundAccount for both.

sequence = IBridge(l1MessageRelayer).createRetryableTicket{value: cost[1]}(l2TargetDispenser, 0,
            maxSubmissionCostMessage, refundAccount, refundAccount, gasLimitMessage, gasPriceBid, data);
Once the ticket is created, the malicious “refundAccount” manually cancel the ticket and on Arbitrum the Ticket is deleted event is emmited. It means that receiveMessage function will never be called on the L2.

## Impact
On the Arbitrum L2, the receiveMessage won’t be called. And crucial synchronisation step will not be executed. Precisely, _processData function will not be triggered, which is crucial step to sync the withheldAmount which “failed to be delivered” on L2. And when the syncWithheldTokens will be called on L2 (to propagate the withheldAmount on L1) , it will revert if withheldAmount would be 0, or proceed with incorrect params.

Eventually, the correct synchronisation step of withheldAmount could be failed, due to ticket cancellation.

Additionally, If a ticket with a callvalue is eventually discarded (cancelled or expired), having never successfully run, the escrowed callvalue will be paid out to a callValueRefundAddress account that was specified in the initial submission, so the cost of such attack is almost miserable

## Recommendation
Ensure, that callValueRefundAddress is set to trusted address, as it is stated in the technical documentation.

-----------
# StakingBase.sol can’t handle fee-on-transfer token #81

## Lines of code
https://github.com/code-423n4/2024-05-olas/blob/3ce502ec8b475885b90668e617f3983cea3ae29f/registries/contracts/staking/StakingToken.sol#L115-L125

## Vulnerability details
Olas Protocol allows to utilise “fee-on-transfer” (PAXG, STA) tokens as stakingToken. However, the deposit function isn’t implemented in such way, so it could handle the “fee-on-transfer” token.

function deposit(uint256 amount) external {
        // Add to the contract and available rewards balances
        uint256 newBalance = balance + amount;
        uint256 newAvailableRewards = availableRewards + amount;

        // Record the new actual balance and available rewards
        balance = newBalance;
        availableRewards = newAvailableRewards;
        //@audit
        // Add to the overall balance
        SafeTransferLib.safeTransferFrom(stakingToken, msg.sender, address(this), amount);

        emit Deposit(msg.sender, amount, newBalance, newAvailableRewards);
    }
## Proof of Concept
Assume user deposit 10 PAXG tokens. The newBalance is incremented with passed amount exactly, while in reality the safeTransferFrom amount will be less. Let’s assume that 9.5 PAXG tokens will be transferred. It means, that availableRewards will be incremented incorrectly
When the owner of the StakingBase contract claim the reward, the rewards will be based on the total availableRewards . It means reward calculation will be based on the 10 PAXG tokens instead of actual amount transferred which is 9.5 PAXG
The claim function, as well as unstake will fail, because it will try to _withdraw more than the contract hold.
Impact
Owner can’t claim / unstake the rewards.

## Recommendation
In deposit function , add only the actual transferred amounts (computed by PostTransferBalance - PreTransferBalance

------------
# BlackListed multisig will not be able to withdraw the funds. 

## Lines of code
https://github.com/code-423n4/2024-05-olas/blob/3ce502ec8b475885b90668e617f3983cea3ae29f/registries/contracts/staking/StakingToken.sol#L104-L110

## Vulnerability Detail
User could create the staking contract with the USDC || USDT as a stakingToken. However after some time, if the user's multisig wallet will be blacklisted by USDC || USDT, he will not be able to claim/unstake the staking reward, which will be accumulated and stuck forever in the Staking contract.

## Proof of Concept
https://github.com/code-423n4/2024-05-olas/blob/3ce502ec8b475885b90668e617f3983cea3ae29f/registries/contracts/staking/StakingToken.sol#L104-L110

## Impact
USDC or USDT funds of the black listed account will be stuck forever in the Olas Staking contract.

## Recommendation
There are 2 options. Ensure that owner could withdraw the funds to the any address that he pass into the unstake/claim params. Or, create the withdraw mechanism which would allow to withdraw the blacklisted funds and put it into the Olas DAO “vault.”

## Assessed type
Context
