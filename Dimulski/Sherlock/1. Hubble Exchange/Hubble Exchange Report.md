# [Hubble Exchange](https://audits.sherlock.xyz/contests/72/report)

## Findings by Dimulski
| Severity | Title | 
|:--:|:---|
| [M-01](#M-01) | Oracle#getUnderlyingPrice may return stale data | 

## <a id='M-01'></a>[M-01] Oracle#getUnderlyingPrice may return stale data
## Summary
getUnderlyingPrice makes the AggregatorV3Interface(chainLinkAggregatorMap[underlying]).latestRoundData() subcall but only validates the price. It doesn't validate when price was last updated. Chainlink oracle have a built in heartbeat and data that has been updated outside of this window is stale and cannot be trusted.

## Vulnerability Detail
[Oracle.sol#L24-L36](https://github.com/hubble-exchange/hubble-protocol/blob/d89714101dd3494b132a3e3f9fed9aca4e19aef6/contracts/Oracle.sol#L24-L36)
```solidity
function getUnderlyingPrice(address underlying)
    virtual
    external
    view
    returns(int256 answer)
{
    if (stablePrice[underlying] != 0) {
        return stablePrice[underlying];
    }
    (,answer,,,) = AggregatorV3Interface(chainLinkAggregatorMap[underlying]).latestRoundData();
    require(answer > 0, "Oracle.getUnderlyingPrice.non_positive");
    answer /= 100;
}
```
The above lines never validate when the price was last updated allowing for it to use stale data. Using outdated price data can lead to over borrowing or unfair liquidation.

## Impact
Contract can consume outdated information leading to incorrect prices

## Code Snippet
[Oracle.sol#L24-L36](https://github.com/hubble-exchange/hubble-protocol/blob/d89714101dd3494b132a3e3f9fed9aca4e19aef6/contracts/Oracle.sol#L24-L36)

## Tool used
Manual Review

## Recommendation
Validate the update time to make sure it is within the heartbeat period.
