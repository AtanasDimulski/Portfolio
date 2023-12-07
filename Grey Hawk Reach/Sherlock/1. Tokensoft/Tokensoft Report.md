# [Tokensoft Report](https://audits.sherlock.xyz/contests/100/leaderboard)

## Findings by 0xnevi
| Severity | Title | 
|:--:|:---|
| [M-01](#crosschain-token-airdrop-always-fails-due-to-insufficient-feeCrosschain token airdrop always fails due to insufficient fee)| Crosschain token airdrop always fails due to insufficient fee | 

# Crosschain token airdrop always fails due to insufficient fee
# Summary
xcall in CrosschainDistributor._settleClaim always fails due to insufficient fee.

# Vulnerability Detail
Function _settleClaim() from CrosschainDistributor.sol sends tokens to another chain using connext.xcall(). According to Connext Docs, fee for a crosschain txn is paid either in:

Native token - using xcall{value: ...}().
The asset being transferred - using optional _relayerFee parameter.
https://docs.connext.network/developers/guides/estimating-fees

Because _settleClaim uses neither of these, every xcall will fail.

```solidity
id = connext.xcall(
    _recipientDomain, // destination domain
    _recipient, // to
    address(token), // asset
    _recipient, // delegate, only required for self-execution + slippage
    _amount, // amount
    0, // slippage -- assumes no pools on connext
    bytes('') // calldata
  );
```
# Impact
The protocol is unable to airdrop tokens to beneficiaries on another chain

# Code Snippet
https://github.com/sherlock-audit/2023-06-tokensoft/blob/1f58ddb066ab383c416cfcbf95c9902683506e96/contracts/contracts/claim/abstract/CrosschainDistributor.sol#L68C1-L90C2

# Tool used
Manual Review

# Recommendation
Implement one of the two fee payment mechanisms