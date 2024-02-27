# [The Wildcat Protocol Report](https://code4rena.com/reports/2023-10-wildcat)

## Findings by Grey Hawk Reach
| Severity | Title | 
|:--:|:--:|
| [H-01](#H-01) | Wrong order of parameters leads to a permanent loss of funds for the lender | 
| [M-01](#M-01) | Incompatibility with Rebase tokens | 

## <a id='H-01'></a>[H-01] Wrong order of parameters leads to a permanent loss of funds for the lender   
## Lines of code
https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketWithdrawals.sol#L164-L180
https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketBase.sol#L170-L186

## Impact
There are two instances of this issue which are provided in the links to affected code. The first instance is in WildcatMarketBase.sol in the _blockAccount() function. The second instance is in WildcatMarketWithdrawals.sol in the executeWithdrawal() function. Both instances have a similar impact. If a user is sanctioned by the Chainalysis oracle, he can either get nukeFromOrbit resulting in his account getting the Blocked role, and all of his scaledBalance will be sent to an escrow account. In the second instance if a user is sanctioned, and tries to withdraw the underlying asset of the market, the underlying asset will be sent to an escrow account. The problem arises when the escrow contract is created. As can be seen in the below code snippet from the executeWithdrawal() function. The first parameter supplied to the createEscrow() fucntion is the account address and the second is the borrower address.

```solidity
// Snippet1
address escrow = IWildcatSanctionsSentinel(sentinel).createEscrow(
          accountAddress,
          borrower,
          address(asset)
        );
```
The WildcatSanctionsSentinel.sol createEscrow() function expects parameters in the following order:

```solidity
// Snippet2
function createEscrow(
    address borrower,
    address account,
    address asset
  )
```
When the WildcatSanctionsEscrow.sol is created the borrower variable will be set to accountAddress, and the account variable will be set to the borrower supplied in Snippet1. In the case that the sanctioned account get unsanctioned from the Chainalysis oracle, or from the borrower (in the current implementation overrideSanction() won't work as expected as borrower and account are switched), and the user calls the releaseEscrow() function in WildcatSanctionsEscrow.sol the funds will be sent to the borrower address, and the lender will lose all of his funds. In case _blockAccount() is called then all of the sanctioned user scaledBalance will be sent to an escrow contract, which can then only be withdrawn to the borrower account. If the borrower decides, he can immediately call the releaseEscrow() function and get the funds send to him, because the isSactioned() check will return false for his address, as he is not sanctioned. Chainanalysis uses OFAC to decide which accounts should be sanctioned https://go.chainalysis.com/chainalysis-oracle-docs.html. OFAC lists can be appealed https://ofac.treasury.gov/specially-designated-nationals-list-sdn-list/filing-a-petition-for-removal-from-an-ofac-list. The below provided POC is when a sanctioned user calls executeWithdrawal().

## Proof of Concept
Add this test to WildcatMarketController.t.sol

```solidity
function test_WrongOrderOfFuncParams() public {
    setUpContracts();

    /// INFO: Approve Lenders
    vm.startPrank(borrower);
    address[] memory lenders = new address[](1);
    lenders[0] = alice;
    wildcatMarketController.authorizeLenders(lenders);
    address[] memory markets = new address[](1);
    markets[0] = marketAddress;
    wildcatMarketController.updateLenderAuthorization(alice, markets);
    vm.stopPrank();

    /// INFO: Alice deposits
    vm.startPrank(alice);
    mockERC20.approve(marketAddress, 10e18);
    wildcatMarket.depositUpTo(10e18);
    console.log("Alice scale token balance: ", wildcatMarket.balanceOf(alice));
    console.log("Alice mockERC20 token balance: ", mockERC20.balanceOf(alice));
    console.log("Borrower scale token balance before alice tries to withdraw: ", wildcatMarket.balanceOf(borrower));
    console.log("Borrower mockERC20 token balance before alice tries to withdraw: ", mockERC20.balanceOf(borrower));

    /// INFO: Alice withdraws
    wildcatMarket.queueWithdrawal(10e18);
    uint32 expiry = wildcatMarket.currentState().pendingWithdrawalExpiry;

    // skip 1 hour
    skip(3601);
    
    /// INFO: Alice gets sanctioned by Chainalysis
    MockChainalysis(address(SanctionsList)).sanction(alice);
    
    wildcatMarket.executeWithdrawal(alice, expiry);
    address aliceEscrow = sentinel.getEscrowAddress(alice, borrower, address(mockERC20));
    console.log("Balance of esrow account that alice's mockERC20 tokens were sent to: ", mockERC20.balanceOf(aliceEscrow));
    console.log("Alice scale token balance: ", wildcatMarket.balanceOf(alice));
    console.log("Alice mockERC20 token balance: ", mockERC20.balanceOf(alice));

    /// INFO: Alice gets unsactiond by the Chainalysis or the borrower unsactions her
    MockChainalysis(address(SanctionsList)).unsanction(alice);
    WildcatSanctionsEscrow(aliceEscrow).releaseEscrow();
    console.log("MockERC20 balance of esrow account that alice's funds were sent to: ", mockERC20.balanceOf(aliceEscrow));
    console.log("Alice scale token balance: ", wildcatMarket.balanceOf(alice));
    console.log("Alice mockERC20 token balance: ", mockERC20.balanceOf(alice));
    console.log("Borrower mockERC20 balance: ", mockERC20.balanceOf(borrower));
    vm.stopPrank();
  }
```
```solidity
Logs:
  Alice scale token balance:  10000000000000000000
  Alice mockERC20 token balance:  0
  Borrower scale token balance before alice tries to withdraw:  0
  Borrower mockERC20 token balance before alice tries to withdraw:  0
  Balance of esrow account that alice's mockERC20 tokens were sent to:  10000000000000000000
  Alice scale token balance:  0
  Alice mockERC20 token balance:  0
  MockERC20 balance of esrow account that alice's funds were sent to:  0
  Alice scale token balance:  0
  Alice mockERC20 token balance:  0
  Borrower mockERC20 balance:  10000000000000000000
```
In order to run the test you have to import the following to WildcatMarketController.t.so

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

Add the following variables to WildcatMarketController.t.sol

```solidity
address public alice = address(123);
  address public bob = address(124);
  address public hacker = address(125);
  address public borrower = address(126);
  address public archOwner = address(127);
  WildcatArchController public wildcatArchController;
  WildcatMarketControllerFactory public wildcatMarketControllerFactory;
  MarketParameterConstraints public constraints;

  WildcatMarket public wildcatMarket;
  address public marketAddress;

  WildcatMarketController public wildcatMarketController;
  address public marketControllerAddress;
  MockERC20 public mockERC20;
  WildcatSanctionsSentinel internal sentin
```

Add the following functions to WildcatMarketController.t.sol

```solidity
function _resetConstraints() internal {
    constraints = MarketParameterConstraints({
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
    mockERC20.mint(alice, 10e18);

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
      constraints
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
## Impact
Malicious users can claim significant rewards while providing liquidity only for one block per week.

In the extreme scenario, whales will use this strategy to collect nearly all CANTO rewards from every market.

## Tools Used
Manual Review & Foundry 

## Recommended Mitigation Steps
There are two instances of the affected code in both of them you have to reorder the parameters

This is the correct order:
```solidity
address escrow = IWildcatSanctionsSentinel(sentinel).createEscrow(
        borrower,
        accountAddress,
        address(asset)
      );
```
## Assessed type
Context

## <a id='M-01'></a>[M-01] Incompatibility with Rebase tokens 
## Lines of code
https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatMarketController.sol#L291-L360

## Vulnerability details
## Impact
Borrowers can choose whatever token they want to be the underlying token for a market. The problem comes when those tokens are Rebasing tokens such as Ampleforth. The balances of those tokens are changed (rebased) by a certain algorithm depending on the purpose of the Rebasing token and the market conditions. When lenders initially deposit Rebase tokens in the Wildcat protocol, the amount of that deposit will be stored within the protocol. Based on the internal rebasing of Wildcat the scaledAmound will change based on scaledFactor which increases in order to add APR to the lender deposits, but that doesn't account for rebasing in the underlying token. At a later stage this may result in wrong user balance accounting. In the case of Ampleforth if the price_exchange_rate of AMPL > 1 2019 USD the market is indicating there is more demand than supply. The Ampleforth protocol automatically and proportionally increases the quantity of tokens in user wallets. This will result in users being able to withdraw more tokens that they are supposed to and eventually the last user to withdraw won't be able to, as there won't be enough tokens left in the Wildcat protocol. Similar scenario is presented in the POC

## Proof of Concept
Add the following test to WildcatMarketController.t.sol

```solidity
function test_rebaseToken() public {
    setUpContracts();

    /// INFO: Mint Rebase token to Alice and Bob
    vm.startPrank(rebaseTokenOwner);
    rebaseToken.transfer(alice, 10e9);
    rebaseToken.transfer(bob, 10e9);
    console.log("");
    console.log("Mint Rebase token to Alice and Bob");
    console.log("Alice rebase token balance: ", rebaseToken.balanceOf(alice));
    console.log("Alice rebase token balance: ", rebaseToken.balanceOf(bob));
    console.log("");
    vm.stopPrank();

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

    vm.startPrank(alice);
    rebaseToken.approve(marketAddress, 10e9);
    wildcatMarket.depositUpTo(10e9);
    vm.stopPrank();

    vm.startPrank(bob);
    rebaseToken.approve(marketAddress, 10e9);
    wildcatMarket.depositUpTo(10e9);
    vm.stopPrank();

    /// INFO: Rebase the token
    vm.startPrank(rebaseTokenOwner);
    rebaseToken.rebase(-49_000_000e8);
    vm.stopPrank();

    vm.startPrank(alice);
    wildcatMarket.queueWithdrawal(10e9);
    uint32 expiry = wildcatMarket.currentState().pendingWithdrawalExpiry;
    skip(3601);
    wildcatMarket.executeWithdrawal(alice, expiry);
    vm.stopPrank();

    vm.startPrank(bob);
    wildcatMarket.queueWithdrawal(10e9);
    uint32 expiryNew = wildcatMarket.currentState().pendingWithdrawalExpiry;
    skip(3601);
    wildcatMarket.executeWithdrawal(bob, expiryNew);
    console.log("");
    console.log("Balance of Bob, Alice and Market after Bob and Alice withdrew their rebase token");
    console.log("Alice rebase token balance after withdraw: ", rebaseToken.balanceOf(alice));
    console.log("Bob rebase token balance after withdraw: ", rebaseToken.balanceOf(bob));
    console.log("Market rebase token balance after rebase: ", rebaseToken.balanceOf(marketAddress));
    vm.stopPrank();
  }
```
```solidity
Logs:
  Rebase token got deployed:  0xc7183455a4C133Ae270771860664b6B7ec320bB1

  Mint Rebase token to Alice and Bob
  Alice rebase token balance:  10000000000
  Alice rebase token balance:  10000000000


  Balance of Bob, Alice and Market after Bob and Alice withdrew their rebase token
  Alice rebase token balance after withdraw:  10000000000
  Bob rebase token balance after withdraw:  8040000000
  Market rebase token balance after rebase:  0
```
When bob quarries for withdraw the 10e9 rebase tokens he deposited and later withdraws them, he receives less than he is supposed to, and Alice effectively stole portion of Bob's deposit due to wrong accounting. In the above test both Alice and Bob should have been allowed to withdraw a maximum of 9020000000 tokens - usually minus the protocol fees, which in the POC are set to 0.

In order to run the test you have to import the following to WildcatMarketController.t.sol
```solidity
import {WildcatArchController} from 'src/WildcatArchController.sol';
import {WildcatMarketControllerFactory} from 'src/WildcatMarketControllerFactory.sol';
import {
  MinimumDelinquencyGracePeriod, MaximumDelinquencyGracePeriod, 
  MinimumReserveRatioBips, MaximumReserveRatioBips, MinimumDelinquencyFeeBips, 
  MaximumDelinquencyFeeBips, MinimumWithdrawalBatchDuration, MaximumWithdrawalBatchDuration, 
  MinimumAnnualInterestBips, MaximumAnnualInterestBips } from './shared/TestConstants.sol';
import {WildcatMarketController} from 'src/WildcatMarketController.sol';
import {WildcatSanctionsSentinel, IChainalysisSanctionsList, IWildcatArchController } from 'src/WildcatSanctionsSentinel.sol';
import {WildcatSanctionsEscrow, IWildcatSanctionsEscrow } from 'src/WildcatSanctionsEscrow.sol';
import {SanctionsList} from 'src/libraries/Chainalysis.sol';
import {MockChainalysis, deployMockChainalysis } from './helpers/MockChainalysis.sol';
import {RebaseToken} from './helpers/RebaseToken.sol';
```
Add the following variables to WildcatMarketController.t.sol
```solidity
  address public alice = address(123);
  address public bob = address(124);
  address public hacker = address(125);
  address public borrower = address(126);
  address public archOwner = address(127);
  address public jon = address(128);
  address public rebaseTokenOwner = address(129);
  WildcatArchController public wildcatArchController;
  WildcatMarketControllerFactory public wildcatMarketControllerFactory;
  MarketParameterConstraints public constraintsWMC;

  WildcatMarket public wildcatMarket;
  address public marketAddress;

  WildcatMarketController public wildcatMarketController;
  address public marketControllerAddress;
  WildcatSanctionsSentinel internal sentinel;
  RebaseToken public rebaseToken;
```

Add the following functions to WildcatMarketController.t.sol
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
    rebaseToken = new RebaseToken();
    console.log("Rebase token got deployed: ", address(rebaseToken));
    vm.startPrank(rebaseTokenOwner);
    rebaseToken.initialize(rebaseTokenOwner);
    vm.stopPrank();

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
      address(rebaseToken),
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

Add RebaseToken.sol to test/helpers

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import "./SafeMathInt.sol";
import "./SafeMath.sol";

contract RebaseToken {
    using SafeMath for uint256;
    using SafeMathInt for int256;
    uint256 public constant DECIMALS = 9;
    uint256 public constant MAX_UINT256 = type(uint256).max;
    uint256 public constant INITIAL_FRAGMENTS_SUPPLY = 50 * 10**6 * 10**DECIMALS;
    string public _name;
    string public _symbol;
    uint8 public _decimals;
    // TOTAL_GONS is a multiple of INITIAL_FRAGMENTS_SUPPLY so that _gonsPerFragment is an integer.
    // Use the highest value that fits in a uint256 for max granularity.
    uint256 public constant TOTAL_GONS = MAX_UINT256 - (MAX_UINT256 % INITIAL_FRAGMENTS_SUPPLY);

    // MAX_SUPPLY = maximum integer < (sqrt(4*TOTAL_GONS + 1) - 1) / 2
    uint256 public constant MAX_SUPPLY = type(uint128).max; // (2^128) - 1

    uint256 public _totalSupply;
    uint256 public _gonsPerFragment;
    mapping(address => uint256) public _gonBalances;

    // This is denominated in Fragments, because the gons-fragments conversion might change before
    // it's fully paid.
    mapping(address => mapping(address => uint256)) public _allowedFragments;

    function initialize(address owner_) public {
        _name = "Ampleforth";
        _symbol = "AMPL";
        _decimals = uint8(DECIMALS);
        _totalSupply = INITIAL_FRAGMENTS_SUPPLY;
        _gonBalances[owner_] = TOTAL_GONS;
        _gonsPerFragment = TOTAL_GONS.div(_totalSupply);
    }

    function rebase(int256 supplyDelta) external returns (uint256)
    {
        if (supplyDelta == 0) {
            return _totalSupply;
        }

        if (supplyDelta < 0) {
            _totalSupply = _totalSupply.sub(uint256(supplyDelta.abs()));
        } else {
            _totalSupply = _totalSupply.add(uint256(supplyDelta));
        }

        if (_totalSupply > MAX_SUPPLY) {
            _totalSupply = MAX_SUPPLY;
        }

        _gonsPerFragment = TOTAL_GONS.div(_totalSupply);
        return _totalSupply;
    }

    /**
     * @return The total number of fragments.
     */
    function totalSupply() external view returns (uint256) {
        return _totalSupply;
    }

    function decimals() external view returns (uint256) {
        return _decimals;
    }

    function name() external view returns (string memory) {
        return _name;
    }

    function symbol() external view returns (string memory) {
        return _symbol;
    }
     /**
     * @param who The address to query.
     * @return The balance of the specified address.
     */
    function balanceOf(address who) external view returns (uint256) {
        return _gonBalances[who].div(_gonsPerFragment);
    }

    /**
     * @param who The address to query.
     * @return The gon balance of the specified address.
     */
    function scaledBalanceOf(address who) external view returns (uint256) {
        return _gonBalances[who];
    }

    /**
     * @return the total number of gons.
     */
    function scaledTotalSupply() external pure returns (uint256) {
        return TOTAL_GONS;
    }

    function transfer(address to, uint256 value) external returns (bool) {
        uint256 gonValue = value.mul(_gonsPerFragment);
        _gonBalances[msg.sender] = _gonBalances[msg.sender].sub(gonValue);
        _gonBalances[to] = _gonBalances[to].add(gonValue);
        return true;
    }

    function allowance(address owner_, address spender) external view returns (uint256) {
        return _allowedFragments[owner_][spender];
    }

    function transferFrom(address from, address to, uint256 value) external returns (bool) {
        _allowedFragments[from][msg.sender] = _allowedFragments[from][msg.sender].sub(value);
        uint256 gonValue = value.mul(_gonsPerFragment);
        _gonBalances[from] = _gonBalances[from].sub(gonValue);
        _gonBalances[to] = _gonBalances[to].add(gonValue);
        return true;
    }

    function approve(address spender, uint256 value) external returns (bool) {
        _allowedFragments[msg.sender][spender] = value;
        return true;
    }
}
```

Add SafeMath.sol to test/helpers
```solidity
pragma solidity 0.8.20;

/**
 * @title SafeMath
 * @dev Math operations with safety checks that revert on error
 */
library SafeMath {
    /**
     * @dev Multiplies two numbers, reverts on overflow.
     */
    function mul(uint256 a, uint256 b) internal pure returns (uint256) {
        // Gas optimization: this is cheaper than requiring 'a' not being zero, but the
        // benefit is lost if 'b' is also tested.
        // See: https://github.com/OpenZeppelin/openzeppelin-solidity/pull/522
        if (a == 0) {
            return 0;
        }

        uint256 c = a * b;
        require(c / a == b);

        return c;
    }

    /**
     * @dev Integer division of two numbers truncating the quotient, reverts on division by zero.
     */
    function div(uint256 a, uint256 b) internal pure returns (uint256) {
        require(b > 0); // Solidity only automatically asserts when dividing by 0
        uint256 c = a / b;
        // assert(a == b * c + a % b); // There is no case in which this doesn't hold

        return c;
    }

    /**
     * @dev Subtracts two numbers, reverts on overflow (i.e. if subtrahend is greater than minuend).
     */
    function sub(uint256 a, uint256 b) internal pure returns (uint256) {
        require(b <= a);
        uint256 c = a - b;

        return c;
    }

    /**
     * @dev Adds two numbers, reverts on overflow.
     */
    function add(uint256 a, uint256 b) internal pure returns (uint256) {
        uint256 c = a + b;
        require(c >= a);

        return c;
    }

    /**
     * @dev Divides two numbers and returns the remainder (unsigned integer modulo),
     * reverts when dividing by zero.
     */
    function mod(uint256 a, uint256 b) internal pure returns (uint256) {
        require(b != 0);
        return a % b;
    }
}
```

Add SafeMathInt.sol to test/helpers
```solidity
/*
MIT License

Copyright (c) 2018 requestnetwork
Copyright (c) 2018 Fragments, Inc.

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
*/

// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.20;

/**
 * @title SafeMathInt
 * @dev Math operations for int256 with overflow safety checks.
 */
library SafeMathInt {
    int256 private constant MIN_INT256 = int256(1) << 255;
    int256 private constant MAX_INT256 = ~(int256(1) << 255);

    /**
     * @dev Multiplies two int256 variables and fails on overflow.
     */
    function mul(int256 a, int256 b) internal pure returns (int256) {
        int256 c = a * b;

        // Detect overflow when multiplying MIN_INT256 with -1
        require(c != MIN_INT256 || (a & MIN_INT256) != (b & MIN_INT256));
        require((b == 0) || (c / b == a));
        return c;
    }

    /**
     * @dev Division of two int256 variables and fails on overflow.
     */
    function div(int256 a, int256 b) internal pure returns (int256) {
        // Prevent overflow when dividing MIN_INT256 by -1
        require(b != -1 || a != MIN_INT256);

        // Solidity already throws when dividing by 0.
        return a / b;
    }

    /**
     * @dev Subtracts two int256 variables and fails on overflow.
     */
    function sub(int256 a, int256 b) internal pure returns (int256) {
        int256 c = a - b;
        require((b >= 0 && c <= a) || (b < 0 && c > a));
        return c;
    }

    /**
     * @dev Adds two int256 variables and fails on overflow.
     */
    function add(int256 a, int256 b) internal pure returns (int256) {
        int256 c = a + b;
        require((b >= 0 && c >= a) || (b < 0 && c < a));
        return c;
    }

    /**
     * @dev Converts to absolute value, and fails on overflow.
     */
    function abs(int256 a) internal pure returns (int256) {
        require(a != MIN_INT256);
        return a < 0 ? -a : a;
    }

    /**
     * @dev Computes 2^exp with limited precision where -100 <= exp <= 100 * one
     * @param one 1.0 represented in the same fixed point number format as exp
     * @param exp The power to raise 2 to -100 <= exp <= 100 * one
     * @return 2^exp represented with same number of decimals after the point as one
     */
    function twoPower(int256 exp, int256 one) internal pure returns (int256) {
        bool reciprocal = false;
        if (exp < 0) {
            reciprocal = true;
            exp = abs(exp);
        }

        // Precomputed values for 2^(1/2^i) in 18 decimals fixed point numbers
        int256[5] memory ks = [
            int256(1414213562373095049),
            1189207115002721067,
            1090507732665257659,
            1044273782427413840,
            1021897148654116678
        ];
        int256 whole = div(exp, one);
        require(whole <= 100);
        int256 result = mul(int256(uint256(1) << uint256(whole)), one);
        int256 remaining = sub(exp, mul(whole, one));

        int256 current = div(one, 2);
        for (uint256 i = 0; i < 5; i++) {
            if (remaining >= current) {
                remaining = sub(remaining, current);
                result = div(mul(result, ks[i]), 10**18); // 10**18 to match hardcoded ks values
            }
            current = div(current, 2);
        }
        if (reciprocal) {
            result = div(mul(one, one), result);
        }
        return result;
    }
}
```

To run the test use forge test -vvv --mt test_rebaseToken

## Tools Used
Manual review & Foundry

## Recommended Mitigation Steps
Implement a whitelist for allowed tokens.set

## Assessed type
Other
