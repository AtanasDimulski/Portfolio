# [The Wildcat Protocol QA Report](https://code4rena.com/reports/2023-10-wildcat)

Wrong accounting in transferFromer   
## Lines of code
https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketToken.sol#L41-L576

## Impact
When a user approves another user to spend a certain amount of his funds, he has to execute the approve() function in the WildcatMarketToken.sol and specify the desired amount. The wildcat token is a rebasing token, that utilizes a scaleFactor to apply interest to the balances of lenders. The value that we are interested in is scaledAmount and that is the value that the user is supposed to be approved for. The problems arises if the approved user executes transferFrom() at a later date. As it can be seen from the provided POC below the difference is almost 20e18 tokens. As the time between approve() is called and the approved account executes a transferFrom() function the difference will be bigger. As mentioned by the protocol in the The Pitch "Wildcat is a tool for sophisticated entities who wish to bring credit agreements on-chain". So we can expect that big amounts are on the line, resulting in big differences on the amount a user is approved for, and supposed to be able to transfer, than the actual amount he can transfer.

## Proof of Concept
Add the following test to WildcatMarketController.t.sol

```solidity
function test_WrongAccountingApprove() public {
    setUpContracts();

    /// INFO: Approve Lenders
    vm.startPrank(borrower);
    address[] memory lenders = new address[](2);
    lenders[0] = alice;
    lenders[1] = bob;
    wildcatMarketController.authorizeLenders(lenders);
    address[] memory markets = new address[](1);
    markets[0] = marketAddress;
    wildcatMarketController.updateLenderAuthorization(alice, markets);
    wildcatMarketController.updateLenderAuthorization(bob, markets);
    vm.stopPrank();

    /// INFO: Alice deposits
    vm.startPrank(alice);
    mockERC20.approve(marketAddress, 10000e18);
    wildcatMarket.depositUpTo(10000e18);
    console.log("This is the scale factor after alice deposit: ", wildcatMarket.scaleFactor());
    wildcatMarket.approve(bob, 10000e18);
    console.log("Alice approves bob for: ", wildcatMarket.allowance(alice, bob));
    vm.stopPrank();

    /// INFO: Bob calls transferFrom
    vm.startPrank(bob);
    uint32 skipSeconds = 7 days;
    skip(skipSeconds);
    console.log("Scaled balance of bob before transfer: ", wildcatMarket.scaledBalanceOf(bob));
    wildcatMarket.transferFrom(alice, bob, 10000e18);
    console.log("Scaled balance of bob after transfer: ", wildcatMarket.scaledBalanceOf(bob));
    console.log("Scaled balance of alice after bob transfer: ", wildcatMarket.scaledBalanceOf(alice));
    console.log("Alice approves bob for: ", wildcatMarket.allowance(alice, bob));
    vm.stopPrank();
  }
```
```solidity
Logs:
  This is the scale factor after alice deposit:  1000000000000000000000000000
  Alice approves bob for:  10000000000000000000000
  Scaled balance of bob before transfer:  0
  Scaled balance of bob after transfer:  9980858627290128520645
  Scaled balance of alice after bob transfer:  19141372709871479355
  Alice approves bob for:  0
```
In order to run the above test add the following imports to WildcatMarketController.t.sol:

```solidity
import {WildcatArchController} from 'src/WildcatArchController.sol';
import {WildcatMarketControllerFactory} from 'src/WildcatMarketControllerFactory.sol';
import {
  MinimumDelinquencyGracePeriod, MaximumDelinquencyGracePeriod, 
  MinimumReserveRatioBips, MaximumReserveRatioBips, MinimumDelinquencyFeeBips, 
  MaximumDelinquencyFeeBips, MinimumWithdrawalBatchDuration, MaximumWithdrawalBatchDuration, 
  MinimumAnnualInterestBips, MaximumAnnualInterestBips } from './shared/TestConstants.sol';
import {WildcatMarketController} from 'src/WildcatMarketController.sol';
import { WildcatSanctionsSentinel, IChainalysisSanctionsList, IWildcatArchController } from 'src/WildcatSanctionsSentinel.sol';
import { WildcatSanctionsEscrow, IWildcatSanctionsEscrow } from 'src/WildcatSanctionsEscrow.sol';
import { SanctionsList } from 'src/libraries/Chainalysis.sol';
import { MockChainalysis, deployMockChainalysis } from './helpers/MockChainalysis.sol';
```

Add the following variables to WildcatMarketController.t.sol:

```solidity
 address public alice = address(123);
  address public bob = address(124);
  address public hacker = address(125);
  address public borrower = address(126);
  address public archOwner = address(127);
  WildcatArchController public wildcatArchController;
  WildcatMarketControllerFactory public wildcatMarketControllerFactory;
  MarketParameterConstraints public constraintsWMC;

  WildcatMarket public wildcatMarket;
  address public marketAddress;

  WildcatMarketController public wildcatMarketController;
  address public marketControllerAddress;
  MockERC20 public mockERC20;
  WildcatSanctionsSentinel internal sentinel;
```

Add the following functions to WildcatMarketController.t.sol:

```solidity
function _resetConstraints() internal {
    constraintsWMC = MarketParameterConstraints({
      minimumDelinquencyGracePeriod: MinimumDelinquencyGracePeriod,
      maximumDelinquencyGracePeriod: MaximumDelinquencyGracePeriod,
      minimumReserveRatioBips: MinimumReserveRatioBips,
      maximumReserveRatioBips: MaximumReserveRatioBips,
      minimumDelinquencyFeeBips: MinimumDelinquencyFeeBips,
      maximumDelinquencyFeeBips: MaximumDelinquencyFeeBips,
      minimumWithdrawalBatchDuration: MinimumWithdrawalBatchDuration,
      maximumWithdrawalBatchDuration: MaximumWithdrawalBatchDuration,
      minimumAnnualInterestBips: MinimumAnnualInterestBips,
      maximumAnnualInterestBips: MaximumAnnualInterestBips
    });
  }

  function setUpContracts() public {
    /// INFO: Deploy MockErc20 token and mint tokens
    mockERC20 = new MockERC20('TokenR', 'TKNR', 18);
    // mockERC20.mint(bob, 10000e18);
    mockERC20.mint(alice, 10000e18);

    vm.startPrank(archOwner);
    /// INFO: Deploy & set up ArchController
    wildcatArchController = new WildcatArchController();
    wildcatArchController.registerBorrower(borrower);

    /// INFO: Set up sentinel
    sentinel = new WildcatSanctionsSentinel(address(wildcatArchController), address(SanctionsList));

    /// INFO: Deploy Factory
    _resetConstraints();
    wildcatMarketControllerFactory = new WildcatMarketControllerFactory(
      address(wildcatArchController),
      address(sentinel),
      constraintsWMC
    );
    wildcatArchController.registerControllerFactory(address(wildcatMarketControllerFactory));
    vm.stopPrank();

    /// INFO: Deploy MarketController and Market
    vm.startPrank(borrower);
    uint128 maxTotalSupply = 100_000e18;
    uint16 annualInterestBips = 1000; // 10%
    uint16 delinquencyFeeBips = 1000; // 10%
    uint32 withdrawalBatchDuration = 3600; // 1 hour
    uint16 reserveRatioBips = 1000; // 10%
    uint32 delinquencyGracePeriod = 3600; // 1 hour

    (marketControllerAddress, marketAddress) = wildcatMarketControllerFactory.deployControllerAndMarket(
      "WildcatTokenR",
      "WCTKNR",
      address(mockERC20),
      maxTotalSupply,
      annualInterestBips,
      delinquencyFeeBips,
      withdrawalBatchDuration,
      reserveRatioBips,
      delinquencyGracePeriod
    );
    wildcatMarket = WildcatMarket(marketAddress);
    wildcatMarketController = WildcatMarketController(marketControllerAddress);
    vm.stopPrank();
  }
```
To run the test use the following command: forge test -vvv --mt test_WrongAccountingApprove

## Tools Used
Manual Review & Foundry 

## Recommended Mitigation Steps
In WildcatMarketToken.sols

This is the correct order:
```solidity
  function transferFrom(
    address from,
    address to,
    uint256 amount
  ) external virtual nonReentrant returns (bool) {
    uint256 allowed = allowance[from][msg.sender];
    /// INFO: if scaleFactor grows state.scaleAmount(amount) will return less than amount, if a user has allowance for 10 tokens but scale factor is 1.1e27, 
    /// succesfully he will transfer 9 tokens, and all his approval will be removed
    // Saves gas for unlimited approvals.
    if (allowed != type(uint256).max) {
      uint256 newAllowance = allowed - amount;
      _approve(from, msg.sender, newAllowance);
    }

    _transfer(from, to, amount);

    return true;
  }
```
before amount is subbed from allowed convert it to scaledAmount
Example:
amount = state.scaleAmount(amount).toUint104();

## Assessed type
Math
