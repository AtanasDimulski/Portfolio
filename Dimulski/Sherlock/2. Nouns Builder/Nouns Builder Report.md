# [Nouns Builder](https://audits.sherlock.xyz/contests/111/report)

## Findings by Dimulski
| Severity | Title | 
|:--:|:---|
| [H-01](#H-01) | The implementation of updateFounders() introduces different vulnerabilities based on the scenario it is called in. | 

## <a id='H-01'></a>[H-01] The implementation of updateFounders() introduces different vulnerabilities based on the scenario it is called in.
## Summary
The [``updateFounders()``](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L375-L437) function may be called by the owner of the Token.sol contract in different scenarios.
When a DAO is created there are n number of tokens that can be reserved for a private sale, if the founders have decided to bootstrap the project in such a way, 
or mint NFTs to certain people on discount. When setting up the parameters for the DAO there is the option for different founders to be set whom in turn will 
receive certain percentage of all the minted NFTs, until their vesting period is over. NFTs grant voting power. For example [Nouns DAO](https://nouns.wtf/) mints 1 of each 10 NFTs to the treasury, 
a single auction continues for a day. Auctions can also be held every 15 mins for example [Lil Nouns](https://lilnouns.wtf/). NFTs are distributed to founders based on their ownership percentage.
For example if the total ownership of all the founders is 10%, and regular users mint 100 NFTs, the total minted NFTs will be 110 and 10 NFTs will be distributed to the 
founders based on their percentage share. As mentioned previously there are several scenarios where the owner of the protocol be it the initial owner, 
or the DAO governance once auctions start may need to utilize the updateFounders() function, some of them are as follows:

1. The initial founders vesting period is ending in a month and a decision to extend their period has been taken
2. One of the founders addresses has been compromised, and a change of the address is required
3. Founders may have to be removed or their percentage of NFTs received adjusted
4. A new founder may have to be added
There are other scenarios as well.

Note: This issue doesn't describe a malicious owner imputing parameters that will benefit him, or supplying a specific wrong parameter that triggers an edge case which in turns leads 
to a loss of value for the protocol or founders. This issue describes a scenario where an admin or governance enters a parameter that would be expected under normal conditions, 
and something is broken.

## Vulnerability Detail
First add the following to Token.t.sol
```solidity
import { console2 } from "forge-std/Test.sol";
```
In NounsBuilderTest.sol modify the following:

- At line 96 change [``percents[0] = 10;``](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/test/utils/NounsBuilderTest.sol#L96) to percents[0] = 5;
- At line 97 change [``percents[1] = 5;``](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/test/utils/NounsBuilderTest.sol#L97) to percents[1] = 3;
- At line 131 change [``0``](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/test/utils/NounsBuilderTest.sol#L131) to 90

To better illustrate what is happening the the updateFounders() and _addFounders() functions add the following to the Token.sol contract:
```solidity
import { console2 } from "forge-std/Test.sol";
```
- At line 170 in the _addFounders() function add console2.log("From Token.sol _addFounders: ", baseTokenId);
- At line 422 in the updateFounders() function add console2.log("From Token.sol updateFounders: ", baseTokenId);

Add the following test to Token.t.sol

```solidity
function test_UpdateFoundersIncorrectFounderRemoval() public {
        deployMock();

        address founder1 = vm.addr(0x1);
        address founder2 = vm.addr(0x2);

        address[] memory founders = new address[](2);
        uint256[] memory percents = new uint256[](2);
        uint256[] memory vestingEnds = new uint256[](2);

        founders[0] = founder1;
        founders[1] = founder2;

        percents[0] = 5;
        percents[1] = 3;

        vestingEnds[0] = 4 weeks;
        vestingEnds[1] = 4 weeks;

        console2.log("Number of reserved tokens: ", token.reservedUntilTokenId());
        console2.log("Total ownership percentage of the founders before the update: ", token.totalFounderOwnership());
        console2.log("Percentage of the already set first founder: ", token.getFounder(0).ownershipPct);
        console2.log("Percentage of the already set second founder: ", token.getFounder(1).ownershipPct);
        address walletFounder1 = token.getFounder(0).wallet;
        address walletFounder2 = token.getFounder(1).wallet;
        console2.log("Address of the first compromised founder: ", walletFounder1);
        console2.log("Address of the second compromised founder: ", walletFounder2);
        console2.log("Address of the new founder1: ", founder1);
        console2.log("Address of founder2: ", founder2);

        /// INFO: Compromised addresses are changed before minting of any tokens starts
        setFounderParams(founders, percents, vestingEnds);

        vm.prank(address(founder));
        token.updateFounders(foundersArr);
        console2.log("Total percentage ownership of the founders after the update: ", token.totalFounderOwnership());

        // Mint 100 tokens
        for (uint256 i = 0; i < 100; i++) {
            vm.prank(address(auction));
            token.mint();
        }

        console2.log("Balance of the first compromised founder: ", token.balanceOf(walletFounder1));
        console2.log("Balance of the second compromised founder: ", token.balanceOf(walletFounder2));
        console2.log("Balance of the correct first founder: ", token.balanceOf(founder1));
        console2.log("Balance of the correct second founder: ", token.balanceOf(founder2));
        console2.log("Here is the total supply: ", token.totalSupply());
    }
```

```solidity
Logs:
  From Token.sol _addFounders:  90
  From Token.sol _addFounders:  10
  From Token.sol _addFounders:  30
  From Token.sol _addFounders:  50
  From Token.sol _addFounders:  70
  From Token.sol _addFounders:  91
  From Token.sol _addFounders:  24
  From Token.sol _addFounders:  57
  Number of reserved tokens:  90
  Total ownership percentage of the founders before the update:  8
  Percentage of the already set first founder:  5
  Percentage of the already set second founder:  3
  Address of the first compromised founder:  0xd3562Fd10840f6bA56112927f7996B7c16edFCc1
  Address of the second compromised founder:  0xA7cBf566E80C4A1Df2C4aE965c79FB087f25E4bF
  Address of the new founder1:  0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf
  Address of founder2:  0x2B5AD5c4795c026514f8317c7a215E218DcCD6cF
  From Token.sol updateFounders:  0
  From Token.sol updateFounders:  20
  From Token.sol updateFounders:  40
  From Token.sol updateFounders:  60
  From Token.sol updateFounders:  80
  From Token.sol updateFounders:  1
  From Token.sol updateFounders:  34
  From Token.sol updateFounders:  67
  From Token.sol _addFounders:  92
  From Token.sol _addFounders:  12
  From Token.sol _addFounders:  32
  From Token.sol _addFounders:  52
  From Token.sol _addFounders:  72
  From Token.sol _addFounders:  93
  From Token.sol _addFounders:  26
  From Token.sol _addFounders:  59
  Total percentage ownership of the founders after the update:  8
  Balance of the first compromised founder:  6
  Balance of the second compromised founder:  4
  Balance of the correct first founder:  6
  Balance of the correct second founder:  4
  Here is the total supply:  120
```

To run the test use: ``forge test -vvv --mt test_UpdateFoundersIncorrectFounderRemoval``

## Impact
In the above POC I have demonstrated a scenario where when setting up a DAO there are two founder addresses that are supplied 
and their percentage of the minted NFTs that they are entitled to is 5 and 3 respectively. Their wallets gets compromised,
before any minting is done, and the owner of the protocol decides to keep the same percentages for each founder but change the addresses. 
As can be seen from the above test scenario the newly updated addresses will receive their correct share, but so will the compromised addresses which may result in an address compromised 
by a malicious actor receiving voting power. As can be seen from the above Logs when the _addFounders() 
function in the Token.sol contract is called upon initialization of the contract, and the reserved ids for the founders are:

```solidity
  From Token.sol _addFounders:  90
  From Token.sol _addFounders:  10
  From Token.sol _addFounders:  30
  From Token.sol _addFounders:  50
  From Token.sol _addFounders:  70
  From Token.sol _addFounders:  91
  From Token.sol _addFounders:  24
  From Token.sol _addFounders:  57
```
correctly reserving 8% of the minted NFTs to be distributed to the founders. 
The problem arises when updateFounders() function is called. It's purpose is to delete all reserved token ids for the previous founders and update the tokenRecipient mapping 
with the new founders parameters by calling _addFounders() function.
When it comes to deleting the previous founders we see that it actually deletes the wrong ids from the tokenRecipient mapping as seen from the Logs:

```solidity
  From Token.sol updateFounders:  0
  From Token.sol updateFounders:  20
  From Token.sol updateFounders:  40
  From Token.sol updateFounders:  60
  From Token.sol updateFounders:  80
  From Token.sol updateFounders:  1
  From Token.sol updateFounders:  34
  From Token.sol updateFounders:  67
```

After the deletion is complete the updateFounders() function calls _addFounders() function

```solidity
    function updateFounders(IManager.FounderParams[] calldata newFounders) external onlyOwner {
        // ...

        // Clear values from storage before adding new founders
        settings.numFounders = 0;
        settings.totalOwnership = 0;
        emit FounderAllocationsCleared(newFounders);

        _addFounders(newFounders, reservedUntilTokenId);
    }
```

As we can see from the Logs:

```solidity
  From Token.sol _addFounders:  92
  From Token.sol _addFounders:  12
  From Token.sol _addFounders:  32
  From Token.sol _addFounders:  52
  From Token.sol _addFounders:  72
  From Token.sol _addFounders:  93
  From Token.sol _addFounders:  26
  From Token.sol _addFounders:  59
```

the _addFounders() function updates the tokenRecipient mapping with the new founders information for selected token ids, correctly representing their ownership percentage. 
After the updateFounders() function is executed the expected correct state of the contract would be the total ownership % to be 8, 
and only the newly updated founders to be entitled to a percent of the NFTs minted. But as we can see from the above POC the total ownership % is 16, and the previous addresses of founders 
are still entitled to a share of the NFTs minted. If we consider that these addresses are compromised the protocol will be minting governace power to a malicious user. 
In some scenario this may result in the ownership percent reaching 100% so all NFTs will be distributed to founders, which makes the whole protocol obsolete as normal 
users won't have any incentive to participate in auctions. In some scenarios some founders may get their ownership percentage increased after updateFounders() function 
is executed, when this was not the intention of the governace. Overall the updateFounders() function is not working correctly, introducing different vulnerabilities 
where either founders receive a wrong percentage of the minted NFTs, or NFTs are minted to compromised addresses, or founders that the DAO want to remove for some reason. 
In most cases in order to fix that the protocol would have to be redeployed. In most of the cases where the reservedUntilTokenId is not a number that can be divided by 100 without a remainder 
(e.g 200 % 100 = 0) this issue will exist thus the High severity.

Note: reservedUntilTokenId being equal to 100 or more introduces another separate issue. The compromised addresses scenario is demonstrated in the POC, as it clearly illustrates the impact. 
I have clearly described several other cases where the consequences for the protocol would be severe. Do not judge this issue as compromised admins keys or malicious owner, 
as this is not the case this issue presents.
## Code Snippet
```solidity
    function updateFounders(IManager.FounderParams[] calldata newFounders) external onlyOwner {
        // ...

        unchecked {
            //  ...

                // using the ownership percentage, get reserved token percentages
                uint256 schedule = 100 / cachedFounder.ownershipPct;

                // Used to reverse engineer the indices the founder has reserved tokens in.
                uint256 baseTokenId;

                for (uint256 j; j < cachedFounder.ownershipPct; ++j) {
                    // Get the next index that hasn't already been cleared
                    while (clearedTokenIds[baseTokenId] != false) {
                        baseTokenId = (++baseTokenId) % 100;
                    }

                    delete tokenRecipient[baseTokenId];
                    clearedTokenIds[baseTokenId] = true;

                    emit MintUnscheduled(baseTokenId, i, cachedFounder);

                    // Update the base token id
                    baseTokenId = (baseTokenId + schedule) % 100;
                }
            }
        }
```

## Tool used
Manual Review & Foundry

## Recommendation
Instead of uint256 baseTokenId; in the updateFounders() function use uint256 baseTokenId = reservedUntilTokenId;.
But be careful because if reservedUntilTokenId is updated before that it may still introduce the vulnerabilities described above. 
Consider a way to keep track of the reservedUntilTokenId that was used last time to add founders.
