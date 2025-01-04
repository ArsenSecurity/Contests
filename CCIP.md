# The s_bucket.capacity mismatch on the Ethereum-Arbitrum could result in tx revert, without being catched.


## Summary

One of the main goal of the CCIP protocol is to gracefully catch the errors, so the failed tx could be retried via the manual execution. If the tx will not be catched, the node/DON will retry the tx in intervals, during the 8 hours duration. The main goal is to find a flow, where the tx would  always revert, so the error can't be catched and manually executed. Firstly, set's take a look how exactly it could happens if the tx would be sent to the Arbitrum if the sequencer fails to post a batches. Secondly, even if the sequencer works properly there is still a risk on the bucket capacity mismatch because the block.timestmaps are updated differently on Ethereum - Arbitrum

## Vulnerability Details

There is a lane from Ethereum Mainnet -> Arbitrum

For example, assume that the bucket capacity of the token sent is 150e18 tokens.

* User A sends 100e18 tokens from Ethereum to Arbitrum. On Ethereum the capacity is decreased by -100e18 tokens, so the **50e18** tokens left in the bucket on Ethereum.
* Users tx reach the Arbitrum L2, where the tokens amount successfuly deducted from the bucket, so on Arbitrum we also have (150e18 - 100e18 = **50e18**) tokens left in the bucket

  This condition is updated, so it  marks the last time stamp of the bucket updating.

  ```Solidity
  s_bucket.lastUpdated = uint32(block.timestamp);
  ```

However, accidentaly the sequencer fail to post a batches, for example due to the following reasons.

* Not enough gas to pay for the tx
* Server Downtime
* Network Congestion

Here is the main point: Some time has passed, the bucket on Ethereum is fully refilled, so the **150e18** tokens already avaliable. While on Arbitrum, the block.timestamp isn't updated, so the bucket capacity on Arbitrum would remain the same -> **50e18** tokens, because the block.timestamp remain the same

Here we have the situation:

* User see that 150e18 tokens avaliable on Ethereum(because they are refilled), so user initiate the tx. The refilled bucket on Ethereum 150e18, deduct the user amount that he sent to Arbitrum (150e18).
* Transaction finally reach the Arbitrum. However, since the block.timestamp isn't updated due to the sequencer failure, the following peace of code will not be executed, it means that bucket isn't updated, so there is still 50e18 tokens in the bucket.
* The function `_releaseOrMintTokens` would try to deduct user transffered funds (150e18) from the bucket capacity on Arbitrum (50e18), which would result in the `TokenRateLimitReached` error.

```Solidity
TokenRateLimitReacheduint256 timeDiff = block.timestamp - s_bucket.lastUpdated;

    if (timeDiff != 0) {
      if (tokens > capacity) revert BucketOverfilled();

      // Refill tokens when arriving at a new block time
      tokens = _calculateRefill(capacity, tokens, timeDiff, s_bucket.rate);

      s_bucket.lastUpdated = uint32(block.timestamp);
    }
```

The real problem could happens if the sequencer  fails to post a batches for more than 8 hours (current time-range, according the CCIP docs, during what the DON will try to re-execute the failed tx). Because in such case the tx wouldn't be catched and manually re-executed which could potentially resulted in the loss of user funds.

> ### Timestamp boundaries of the sequencer[​](https://docs.arbitrum.io/build-decentralized-apps/arbitrum-vs-ethereum/block-numbers-and-time#timestamp-boundaries-of-the-sequencer "Direct link to Timestamp boundaries of the sequencer")
>
> As mentioned, block timestamps are usually set based on the sequencer's clock. Because there's a possibility that the sequencer fails to post batches on the parent chain (for example, Ethereum) for a period of time, it should have the ability to slightly adjust the timestamp of the block to account for those delays and prevent any potential reorganisations of the chain. To limit the degree to which the sequencer can adjust timestamps, some boundaries are set, currently to 24 hours earlier than the current time, and 1 hour in the future.
>
> <https://docs.arbitrum.io/build-decentralized-apps/arbitrum-vs-ethereum/block-numbers-and-time>

***

**#2**

**Additionaly**, there is a second issue that could occur. It worth to add that the block.timestamp on Arbitrum is updated pased on the chain usability, not like it is done on Ethereum (every 12 sec), so it also could be a potential vector if the usability of the Arbitrum is very low, the block.timestamp wouldn't be updated in accordance with the Ethereum block.timestmap, so it could potentially create the mismatch with the bucket capacity on the Ethereum - Arbitrum lane. Where the bucket is fully refilled on Ethereum, while on Arbitrum it is refilled only partially.

> Block timestamps on Arbitrum are not linked to the timestamp of the L1 block. They are updated every L2 block based on the sequencer's clock. These timestamps must follow these two rules:
>
> 1. Must be always equal or greater than the previous L2 block timestamp
> 2. Must fall within the established boundaries (24 hours earlier than the current time or 1 hour in the future). More on this below.
>
> Furthermore, for transactions that are force-included from L1 (bypassing the sequencer), the block timestamp will be equal to either the L1 timestamp when the transaction was put in the delayed inbox on L1 (not when it was force-included), or the L2 timestamp of the previous L2 block, whichever of the two timestamps is greater.

Such assumption, that the bucket would be updated properly and in accordance on the both chains poses the risk that the error wouldn't be catched, due to the bucket amount mismatch, and the tx would revert and user would need to get the ChainLink support.

## Impact

The tx could fail -> revert(), without being catched and manually re-executed, which could result in the loss of user's funds.

Impact: High

Likelihood: Low

Result: Medium Severity

## Tools Used

Manual review

## Recommendations

There is a possible solution:

1. The most simple one, just catch the `TokenRateLimitReached` error in the `_trialExecute`
