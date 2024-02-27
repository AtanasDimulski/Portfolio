# [The Wildcat Protocol Report](https://code4rena.com/reports/2023-10-wildcat)

## Findings by Grey Hawk Reach
| Severity | Title | 
|:--:|:--:|
| [H-01](#H-01) | Error in LRTOracle#getRSETHPrice leads to users minting significantly less rsETH than they should | 
| [H-02](#H-02) | Price calculation is vulnerable to the inflation attack | 

## <a id='H-01'></a>[H-01] Error in LRTOracle#getRSETHPrice leads to users minting significantly less rsETH than they should   
## Lines of code
https://github.com/code-423n4/2023-11-kelp/blob/f751d7594051c0766c7ecd1e68daeb0661e43ee3/src/LRTDepositPool.sol#L136-L141

## Vulnerability details
First time rsETH is minted, its price is set to the price of 1 ETH at the time of mint.
After that, every time rsETH is minted, to get the price of rsETH, LRTOracle#getRSETHPrice calculates the total value of LSTs in Kelp contracts:
```solidity
    function getTotalAssetDeposits(address asset) public view override returns (uint256 totalAssetDeposit) {
        (uint256 assetLyingInDepositPool, uint256 assetLyingInNDCs, uint256 assetStakedInEigenLayer) =
            getAssetDistributionData(asset);
        return (assetLyingInDepositPool + assetLyingInNDCs + assetStakedInEigenLayer);
    }
```
and divides it by the supply of rsETH.
Because LRTDepositPool transfers the LST from the user first, and then calculates the price from it, the subsequent depositors will get significantly less rsETH per unit of LST than the first one.

## Proof of Concept
1. Alice, the first depositor, deposits 1e18 stETH and gets 1e18 rsETH.
2. Bob deposits 1e18 stETH and receives:

```solidity
rsethAmountToMint = (amount * lrtOracle.getAssetPrice(asset)) / lrtOracle.getRSETHPrice();

amount = 1e18

getAssetPrice = 1e18

getRSETHPrice = totalETHInPool / rsEthSupply

    totalETHInPool = stETH deposits * stETH/ETH price = 2e18  * 1e18

    rsETHSupply = 1e18

    getRSETHPrice = 2e18

rsethAmountToMint = (1e18 * 1e18) / 2e18 = 0.5e18
```
As you can see, despite depositing the same amount of stETH and at the same stETH/ETH price, Bob receives only a half of rsETH that Alice received.
[Foundry PoC](https://gist.github.com/aslanbekaibimov/20e8632342544c072bac5f33e705175f)
## Impact
Malicious users can claim significant rewards while providing liquidity only for one block per week.

In the extreme scenario, whales will use this strategy to collect nearly all CANTO rewards from every market.

## Tools Used
Foundry  

## Recommended Mitigation Steps
Compute the price first, and then transfer funds into DepositPool.
```diff
+       uint256 rsethAmountMinted = _mintRsETH(asset, depositAmount);
        if (
            !IERC20(asset).transferFrom(
                msg.sender,
                address(this),
                depositAmount
            )
        ) {
            revert TokenTransferFailed();
        }
-       uint256 rsethAmountMinted = _mintRsETH(asset, depositAmount);
```
## Assessed type
Token-Transfer

## <a id='H-02'></a>[H-02] Price calculation is vulnerable to the inflation attack 
## Lines of code
[https://github.com/code-423n4/2023-08-verwa/blob/main/src/GaugeController.sol#L211-L278](https://github.com/code-423n4/2023-11-kelp/blob/f751d7594051c0766c7ecd1e68daeb0661e43ee3/src/LRTDepositPool.sol#L95-L110)

## Vulnerability details
## Impact
This issue persists regardless if the "Error in price calculation" finding is fixed or not. For this issue, we found it reasonable to assume that the mitigation is implemented.

Because rsETH's price depends on the balances of LSTs that DepositPool/NodeDelegators hold, rsETH's price calculation is vulnerable to the classic [ERC4626 inflation attack](https://docs.openzeppelin.com/contracts/5.x/erc4626).

After rsETH have just been deployed, whenever there's a pending deposit of X of LST tokens in the mempool,

Adversary may frontrun the victim: deposit 1 wei of LST, get 1 wei of rsETH, and then donate the same X of LST tokens into DepositPool.

As a result, the victim sends X of LST, gets 0 rsETH; Adversary's 1 wei of rsETH now represents two times more LSTs that the Adversary donated.s.

## Proof of Concept
1. Kelp is deployed
2. Alice deposits 1 wei of stETH and mints 1 wei of rsETH
3. Bob tries to deposit 1e18 of stETH
4. Alice frontruns Bob and donates 1e18 of stETH into DepositPool.
5. Bob's transaction is minted.

```solidity
rsethAmountToMint = (amount * lrtOracle.getAssetPrice(asset)) / lrtOracle.getRSETHPrice();

    getRSETHPrice = totalETHInPool / rsEthSupply

    totalETHInPool = stETH deposits * stETH/ETH price = (1e18 + 1) * 1e18

    rsETHSupply = 1

rsethAmountToMint = (1e18 * 1e18) / ((1e18 + 1) * 1e18) = 0
```
As a result, Bob receives 0 shares. Alice owes the only 1 wei of rsETH in existence. If another user tries to mint rsETH, they will need to provide at least x >= 2e18 + 1 stETH. And they may receive become an another victim, if Alice donates x - 2e18 stETH into the pool.

Assuming the withdrawal amounts will be derived from rsETH/(stETH + cbETH + rETH) ratio, Alice's 1 wei of rsETH will be redeemable for all 2e18 + 1 stETH in the pool, while she spent only a half of that amount.
[Foundry PoC](https://gist.github.com/aslanbekaibimov/22bce3d692c64be8d0aeb350da9479b8)
## Tools Used
Foundry

## Recommended Mitigation Steps
Implement Virtual offset described in https://docs.openzeppelin.com/contracts/5.x/erc4626#defending_with_a_virtual_offset

## Assessed type
ERC4626.
