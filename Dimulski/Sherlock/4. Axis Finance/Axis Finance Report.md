# [Axis Finance](https://audits.sherlock.xyz/contests/206/report)

## Findings by Dimulski
| Severity | Title | 
|:--:|:---|
| [H-01](#H-01) | Revoking user's vesting doesn't remove all of the user's votes | 
| [H-02](#H-02) | Revoking user vesting DOSes the last withdrawers | 
| [H-03](#H-03) | Rewards in the different ZivoeRewards implementations can be diluted by a malicious user | 
| [H-04](#H-04) | If the reward token in ZivoeRewards and ZivoeRewardsVesting is a token with less than 18 decimals, rewards may get stuck in the contract | 
| [M-01](#M-01) | Rewards are calculated as distributed even if there are no stakers, locking the rewards forever | 

## <a id='H-01'></a>[H-01] Revoking user's vesting doesn't remove all of the user's votes
## Summary
The ZivoeGovernorV2.sol contract overrides the _getVotes() function to the following:

```solidity
    function _getVotes(
        address account,
        uint256 blockNumber,
        bytes memory /*params*/
    ) internal view virtual override(Governor, GovernorVotes) returns (uint256) {
        return token.getPastVotes(account, blockNumber) + 
            IZivoeRewardsVesting_ZVG(IZivoeGlobals_ZVG(GBL).vestZVE()).getPastVotes(account, blockNumber) +
            IZivoeRewards_ZVG(IZivoeGlobals_ZVG(GBL).stZVE()).getPastVotes(account, blockNumber);
    }
```

which means that when people are voting for new proposals their voting is comprised of the amount of ZVE tokens they hold, the amount they have staked in the ZivoeRewards.sol contract represented by $stZVE balance, and their vested amount in ZivoeRewardsVesting.sol represented by $vestZVE. In the ZivoeRewardsVesting.sol contract a vesting schedule can be created either by the ZivoeITO.sol contract, based on user deposits, or directly by the ZVL address, which is a multisig wallet controlled by the Zivoe Laboratory by calling the createVestingSchedule() function. The ZivoeRewardsVesting.sol contract allows the ZVL wallet to revoke the vesting schedule of users, this might happen due to different reasons. For example a user is no longer contributing to the development of the protocol, his wallets gets compromised, etc. Revoking a vesting schedule is performed by calling the revokeVestingSchedule() function. However the amount subtracted from _checkpoints[account] is incorrect:

```solidity
function revokeVestingSchedule(address account) external updateReward(account) onlyZVLOrITO nonReentrant {
        ...
        
        uint256 amount = amountWithdrawable(account);
        uint256 vestingAmount = vestingScheduleOf[account].totalVesting;

        vestingTokenAllocated -= amount;

        vestingScheduleOf[account].totalWithdrawn += amount;
        vestingScheduleOf[account].totalVesting = vestingScheduleOf[account].totalWithdrawn;
        vestingScheduleOf[account].cliff = block.timestamp - 1;
        vestingScheduleOf[account].end = block.timestamp;

        vestingTokenAllocated -= (vestingAmount - vestingScheduleOf[account].totalWithdrawn);

        _totalSupply = _totalSupply.sub(vestingAmount);
        _writeCheckpoint(_totalSupplyCheckpoints, _subtract, vestingAmount);
        _writeCheckpoint(_checkpoints[account], _subtract, amount);
        _balances[account] = 0;
        stakingToken.safeTransfer(account, amount);

        vestingScheduleOf[account].revokable = false;
        ...
    }
```

As can be seen from the above code snippet amount is subtracted from the _checkpoints[account] of the user whose vesting is revoked. However amount represent the amount of tokens that a user can withdraw from his vesting schedule based on how much time has passed since his vesting schedule was created:

```solidity
    function amountWithdrawable(address account) public view returns (uint256 amount) {
        if (block.timestamp < vestingScheduleOf[account].cliff) { return 0; }
        if (
            block.timestamp >= vestingScheduleOf[account].cliff && 
            block.timestamp < vestingScheduleOf[account].end
        ) {
            return vestingScheduleOf[account].vestingPerSecond * (
                block.timestamp - vestingScheduleOf[account].start
            ) - vestingScheduleOf[account].totalWithdrawn;
        }
        else if (block.timestamp >= vestingScheduleOf[account].end) {
            return vestingScheduleOf[account].totalVesting - vestingScheduleOf[account].totalWithdrawn;
        }
        else { return 0; }
    }
```

A user who must not have any votes from the ZivoeRewardsVesting.sol contract will have them indefinitely, there is nothing that can be done to remove his votes. He can vote on all new proposals and the total votes that can be casted for a proposal will be more than all the votes that should exist. Depending on when a user vesting schedule is revoked, the consequences for the protocol will be severe, for example if a user got a vesting schedule for a year, but after a month the protocol team decides to revoke it, the user will be left with a big percentage of his initially allocated votes.
NOTE: there is another problem with how _totalSupply and _totalSupplyCheckpoints are calculated, however this is a separate issue.

## Vulnerability Detail
[Gist](https://gist.github.com/AtanasDimulski/e2edba2c03e4dd1325b9e73c8fd58ddb)
After following the steps in the above linked [gist](https://gist.github.com/AtanasDimulski/e2edba2c03e4dd1325b9e73c8fd58ddb) add the following test to the AuditorTests.t.sol:

```solidity
    function test_RevokedVestingDoesNotRemoveAllUserVotes() public {
        unlockYDLandVZVT();
        vm.startPrank(ZVL);
        uint256 amountToVest = 10_000e18;
        zivoeToken.transfer(address(zivoeRewardsVesting), amountToVest);
        zivoeRewardsVesting.createVestingSchedule(alice, 0, 360, amountToVest, true);
        console2.log("Alice initial votes: ", zivoeRewardsVesting.getVotes(alice));
        console2.log("Alice initial withdrawable amount: ", zivoeRewardsVesting.amountWithdrawable(alice));

        /// @notice assuming 1 ethereum block = 12 sec, 86400 / 12 = 7200 
        skip(86400);
        vm.roll(7200);
        console2.log("Alice withdrawable amount after 1 day has passed: ", zivoeRewardsVesting.amountWithdrawable(alice));
        zivoeRewardsVesting.revokeVestingSchedule(alice);

        /// @notice we skip one more day to better illustrate that alice still has votes
        skip(86400);
        vm.roll(7200);
        console2.log("Alice votes after her vesting has been revoked: ", zivoeRewardsVesting.getVotes(alice));
        vm.stopPrank();
    }
```

```solidity
Logs:
  Alice initial votes:  10000000000000000000000
  Alice initial withdrawable amount:  0
  Alice withdrawable amount after 1 day has passed:  27777777777777715200
  Alice votes after her vesting has been revoked:  9972222222222222284800
```

To run the test use: forge test -vvv --mt test_RevokedVestingDoesNotRemoveAllUserVotes

## Impact
A user who must not have any votes from the ZivoeRewardsVesting.sol contract will have them indefinitely, there is nothing that can be done to remove his votes. He can vote on all new proposals and the total votes that can be casted for a proposal will be more than all the votes that should exist.

## Code Snippet
https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeRewardsVesting.sol#L429-L467

## Tool used
Manual Review & Foundry

## Recommendation
In the revokeVestingSchedule() function consider adding a new variable vestingAmountUser which is equal to totalVesting and subtract the totalWihtdrawnAmount from it, right before the current withdrawable amount is added to the totalWithdrawn and subtract vestingAmountUser from the _checkpoints[account]


## <a id='H-02'></a>[H-02] Revoking user vesting DOSes the last withdrawers
## Summary
In the ZivoeRewardsVesting.sol contract a vesting schedule can be created either by the ZivoeITO.sol contract, based on user deposits, or directly by the ZVL address, which is a multisig wallet controlled by the Zivoe Laboratory by calling the createVestingSchedule() function. The ZivoeRewardsVesting.sol contract allows the ZVL wallet to revoke the vesting schedule of users, this might happen due to different reasons. For example a user is no longer contributing to the development of the protocol, his wallets gets compromised, etc. Revoking a vesting schedule is performed by calling the revokeVestingSchedule() function. However the subtractions in the revokeVestingSchedule() function don't take into account that a user may have already withdrawn a portion of his vested tokens.

```solidity
        uint256 amount = amountWithdrawable(account);
        uint256 vestingAmount = vestingScheduleOf[account].totalVesting;
       
        ...

        _totalSupply = _totalSupply.sub(vestingAmount);
        _writeCheckpoint(_totalSupplyCheckpoints, _subtract, vestingAmount);
```

As can be seen in the above code snippet the totalVesting of an account is subtracted from the _totalSupply and the _totalSupplyCheckpoints. A user may have called the withdraw() function before his vesting schedule has been revoked or he cam simply frontrun the transaction.

```solidity
    function withdraw() public nonReentrant updateReward(_msgSender()) {
        uint256 amount = amountWithdrawable(_msgSender());
        require(amount > 0, "ZivoeRewardsVesting::withdraw() amountWithdrawable(_msgSender()) == 0");
        
        vestingScheduleOf[_msgSender()].totalWithdrawn += amount;
        vestingTokenAllocated -= amount;

        _totalSupply = _totalSupply.sub(amount);
        _writeCheckpoint(_totalSupplyCheckpoints, _subtract, amount);
        _writeCheckpoint(_checkpoints[_msgSender()], _subtract, amount);
        _balances[_msgSender()] = _balances[_msgSender()].sub(amount);
        stakingToken.safeTransfer(_msgSender(), amount);

        emit Withdrawn(_msgSender(), amount);
    }
```

As can be seen from the above code snippet the withdrawable amount is subtracted from the _totalSupply and the _totalSupplyCheckpoints but it is not subtracted from the totalVesting of the user who is withdrawing. When the revokeVestingSchedule() function is called to revoke the vesting schedule of a user who has already withdrawn a bigger amount will be subtracted from the _totalSupply and the _totalSupplyCheckpoints , which will lead to the last users not being able to withdraw their vested toke

## Vulnerability Detail
[Gist](https://gist.github.com/AtanasDimulski/e2edba2c03e4dd1325b9e73c8fd58ddb)
After following the steps in the above mentioned [gist](https://gist.github.com/AtanasDimulski/e2edba2c03e4dd1325b9e73c8fd58ddb) add the following test to the AuditorTests.t.sol file:

```solidity
function test_RevokingVestingScheduleDOSLastWithdrawers() public {
        unlockYDLandVZVT();
        vm.startPrank(ZVL);
        uint256 amountToVest = 10_000e18;
        zivoeToken.transfer(address(zivoeRewardsVesting), amountToVest * 2);
        zivoeRewardsVesting.createVestingSchedule(alice, 0, 360, amountToVest, true);
        zivoeRewardsVesting.createVestingSchedule(bob, 0, 360, amountToVest, false);
        console2.log("Total Supply: ", zivoeRewardsVesting.totalSupply());
        vm.stopPrank();

        vm.startPrank(alice);
        /// @notice assuming 1 ethereum block = 12 sec, 86400 / 12 = 7200 
        skip(86400);
        vm.roll(7200);
        console2.log("Alice withdrawable amount after 1 day has passed: ", zivoeRewardsVesting.amountWithdrawable(alice));
        zivoeRewardsVesting.withdraw();
        console2.log("Alice balance: ", zivoeRewardsVesting.balanceOf(alice));
        uint256 totalVestingAlice;
        (,,,totalVestingAlice,,,) = zivoeRewardsVesting.viewSchedule(alice);
        console2.log("Total vesting of alice: ", totalVestingAlice); 
        console2.log("Total Supply: ", zivoeRewardsVesting.totalSupply());
        vm.stopPrank();

        vm.startPrank(ZVL);
        zivoeRewardsVesting.revokeVestingSchedule(alice);
        skip(12);
        vm.roll(7201);
        console2.log("Total supply after alice vesting shcedule is revoked: ", zivoeRewardsVesting.totalSupply());
        console2.log("Total voting stored in checkpoints after alice vesting is revoked: ", zivoeRewardsVesting.getPastTotalSupply(block.number -1));
        vm.stopPrank();

        vm.startPrank(bob);
        /// @notice we skip the entire vesting period
        skip(359 * 86400);
        vm.roll(359 * 7200 + 7201);
        console2.log("Bob withdrawable amount after the whole vesting period has passed: ", zivoeRewardsVesting.amountWithdrawable(bob));
        vm.expectRevert(stdError.arithmeticError);
        zivoeRewardsVesting.withdraw();
        vm.stopPrank();
    }
```

```solidity
Logs:
  Total Supply:  20000000000000000000000
  Alice withdrawable amount after 1 day has passed:  27777777777777715200
  Alice balance:  9972222222222222284800
  Total vesting of alice:  10000000000000000000000
  Total Supply:  19972222222222222284800
  Total supply after alice vesting shcedule is revoked:  9972222222222222284800
  Total voting stored in checkpoints after alice vesting is revoked:  9972222222222222284800
  Bob withdrawable amount after the whole vesting period has passed:  10000000000000000000000
```

To run the test use: forge test -vvv --mt test_RevokingVestingScheduleDOSLastWithdrawers

## Code Snippet
https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeRewardsVesting.sol#L429-L467

## Tool used
Manual Review & Foundry

## Recommendation
Consider subtracting totalWithdrawn from vestingAmount right before vestingAmount is subtracted from _totalSupply and _totalSupplyCheckpoints in the revokeVestingSchedule() function


## <a id='H-03'></a>[H-03] Rewards in the different ZivoeRewards implementations can be diluted by a malicious user
## Summary
The ZivoeRewards.sol and the ZivoeRewardsVesting.sol are forked versions of the Synthetix rewards distribution contract, with slight modifications supporting different assets simultaneously as rewards, allowing everybody to call the depositRewards() function with no requirement on the minimum amount deposited, and requiring the reward for a new period to be deposited every time the depositRewards() function is called. This allows a malicious actor to dilute the rewards users are supposed to receive. Given the fact that the Zivoe protocol is a lending protocol. One of its core functions is to properly incentivize users to lend their assets to the Zivoe protocol so it can later utilize them, this happens by distributing rewards to users (usually advertised as APY). Usually the protocol will distribute rewards to the different ZivoeRewards.sol implementations (stJTT, stSTT, stZVE) and the ZivoeRewardsVesting.sol on a 30 day basis. If there are no previous reward deposits the rewardRate will be calculated in the depositRewards() function in the following way:

```solidity
        rewardData[_rewardsToken].rewardRate = reward.div(rewardData[_rewardsToken].rewardsDuration);
```

Let's take a reward deposit of 30_000 USDC tokens.
reward = 30_000e6
rewardsDuration = 30 days = 2_592_000 seconds
rewardRate = 30_000e6 / 2_592_000 ≈ 11_574

10 days pass and a malicious actor calls the depositRewards() function with the reward parameter = 0. The function will calculate the rewardRate in the following way:

```solidity
        uint256 remaining = rewardData[_rewardsToken].periodFinish.sub(block.timestamp);
        uint256 leftover = remaining.mul(rewardData[_rewardsToken].rewardRate);
        rewardData[_rewardsToken].rewardRate = reward.add(leftover).div(rewardData[_rewardsToken].rewardsDuration);
```

remaining = 1_728_000
leftover = 11_574 * 1_728_000 = 19_999_872_000
rewardRate = 19_999_872_000 / 2_592_000 ≈ 7_716

For the following 20 days users that have staked will accrue a lot less rewards, than what they should, this is effectively stealing rewards for them, because if they want to have the chance to receive the full amount of rewards they were supposed to receive, they will have to continue staking for the next periods. However if there are new stakers in the new period the rewards will be further diluted as the new stakers should receive rewards proportional to their stake percentage as well.

Lets say 20 days have passed, and the protocol makes a new deposit of 30_000e6 tokens
remaining = 864_000
leftover = 7_716 * 864_000 = 6_666_624_000 ≈ 6_666e6
rewardRate = (30_000e6 + 6_666e6) / 2_592_000 ≈ 14_146

As can be seen from the above calculations, the rewardRate will be higher, and stakers that are staking during this period will receive more rewards than they should, effectively stealing from the previous stakers.

Lets say 10 more days pass and a malicious actor calls the depositRewards() function with the reward parameter = 0.

remaining = 1_728_000
leftover = 14_146 * 1_728_000 = 24_444_288_000
rewardRate = 24_444_288_000 / 2_592_000 ≈ 9_430

A malicious actor can dilute the rewards again, as we can see the rewardRate is less than what it is supposed to be for a reward deposit of 30_000e6 (should be 11_574 as per the first calculations). A malicious actor can dilute rewards until there is a period where a lot more rewards will be distributed, and make a big deposit for that period, effectively stealing rewards. Once the rewardRate gets normalized (for example he increases the rewardRate up until a certain period, then stakes in that reward period, and receives much bigger rewards, than he is supposed to) he can start over, and keep diluting rewards forever.

## Vulnerability Detail
[Gist](https://gist.github.com/AtanasDimulski/e2edba2c03e4dd1325b9e73c8fd58ddb)
After following the steps in the above linked [gist](https://gist.github.com/AtanasDimulski/e2edba2c03e4dd1325b9e73c8fd58ddb) add the following test to the AuditorTests.t.sol file:

```solidity
    function test_DiluteRewards() public {
        vm.startPrank(ZVL);
        uint256 duration = 30 days;
        stZVE.addReward(address(mockUSDC), duration);
        vm.stopPrank();

        vm.startPrank(simulateYLD);
        mockUSDC.mint(simulateYLD, 30_000e6); // this represent 30_000 USDC tokens to be distributed in the next 30 days
        mockUSDC.approve(address(stZVE), type(uint256).max);
        stZVE.depositReward(address(mockUSDC), 30_000e6);
        vm.stopPrank();

        vm.startPrank(attacker);
        console2.log("Initial rewards per second per token: ", getRewardRate());
        /// @notice 10 days has passed
        skip(864_000);
        console2.log("Rewards per token per second after 10 days have passed: ", getRewardRate());
        stZVE.depositReward(address(mockUSDC), 0);
        console2.log("Rewards per token per second after 10 days have passed, and rewards have been diluted: ", getRewardRate());
        vm.stopPrank();

        vm.startPrank(simulateYLD);
        skip(1_728_000);
        mockUSDC.mint(simulateYLD, 30_000e6); // this represent 30_000 USDC tokens to be distributed in the next 30 days
        stZVE.depositReward(address(mockUSDC), 30_000e6);
        vm.stopPrank();

        vm.startPrank(attacker);
        /// @notice above we skip 20 days, this represents the new reward period for the system (a whole reward period being 30 days)
        console2.log("Rewards per second per token after 20 more days have passed and a new deposit have been made: ", getRewardRate());
        /// @notice 10 days has passed
        skip(864_000);
        console2.log("Rewards per token per second after 10 more days have passed: ", getRewardRate());
        stZVE.depositReward(address(mockUSDC), 0);
        console2.log("Rewards per token per second after more 10 days have passed, and rewards have been diluted: ", getRewardRate());
        vm.stopPrank();
    }
```

```solidity
Logs:
  Initial rewards per second per token:  11574
  Rewards per token per second after 10 days have passed:  11574
  Rewards per token per second after 10 days have passed, and rewards have been diluted:  7716
  Rewards per second per token after 20 more days have passed and a new deposit have been made:  14146
  Rewards per token per second after 10 more days have passed:  14146
  Rewards per token per second after more 10 days have passed, and rewards have been diluted:  9430
```

To run the test use: ``forge test -vvv --mt test_DiluteRewards``

## Impact
A malicious actor can dilute user rewards. Users won't receive the rewards they should on time, and if they want to they will have to continue staking, as rewards keep getting pushed into the next cycle. New user may stake tokens in the new period effectively stealing rewards from the previous users (we are not talking about dust amounts here). A malicious actor can dilute the rewards for several rounds, until a lot of rewards are to be distributed in a singe period, then make a big stake effectively stealing rewards. Most of the stakers would be decentivized to keep their stake until this point, as they have never received the full rewards for a period up until this point.

## Code Snippet
https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeRewards.sol#L228-L243

## Tool used
Manual Review & Foundry

## Recommendation
Implement an access control allowing only the ZivoeYDL.sol, the ZVL multisig or the ZivoeDAO to call the depositReward() function.
## <a id='H-04'></a>[H-04] If the reward token in ZivoeRewards and ZivoeRewardsVesting is a token with less than 18 decimals, rewards may get stuck in the contract
## Summary
The ZivoeRewards.sol and ZivoeRewardsVesting.sol contracts allow for multiple tokens to be used as rewards simultaneously. The ZivoeRewards.sol contract is a fork of the Synthetix rewards distribution contract, with slight modifications allowing for multiple tokens with different decimals to be used as rewards, requiring the new reward to be deposited when calling the depositReward() function , and allowing everybody to call the depositReward() function with no minimum amount required. However tokens with less than 18 decimals can result in rewards being locked in the contract, and stakers not receiving their rewards. Rewards will be distributed for a period of 30 days in most cases as the main source of rewards will come from the ZivoeYDL.sol contract. The amounts in the following example are chosen to better illustrate the severity of the attack, they are not a specific edge case. Now lets take the USDC token as the rewards token. We have a total stake of 5_000_000e18 tokens, no matter in which ZivoeRewards.sol instance (it can be the junior, senior or the zve staking rewards instances). We have a reward of 50_000e6 tokens that have to be distributed towards the stakers. If there are no previous rewards for the specific token deposited the rewardRate will be calculated by the depositReward() function in the following way:

```solidity
  rewardData[_rewardsToken].rewardRate = reward.div(rewardData[_rewardsToken].rewardsDuration);
```

*30 days = 2592000 seconds*
*50_000e6 / 2592000 = 19290*
The rewardPerToken() function updates the rewardPerTokenStored which is a global variable, as well as the lastUpdateTime meaning that if one user stakes or withdraws this values will be updated, and later used in the reward calculations for other users.

```solidity
lastTimeRewardApplicable(_rewardsToken).sub(rewardData[_rewardsToken].lastUpdateTime)
```

is the duration between someone staking, withdrawing his stake or withdrawing his rewards.
The rewardPerToken() function calculates rewards in the following way:

```solidity
   function rewardPerToken(address _rewardsToken) public view returns (uint256 amount) {
        if (_totalSupply == 0) { return rewardData[_rewardsToken].rewardPerTokenStored; }
        return rewardData[_rewardsToken].rewardPerTokenStored.add(
            lastTimeRewardApplicable(_rewardsToken).sub(
                rewardData[_rewardsToken].lastUpdateTime
            ).mul(rewardData[_rewardsToken].rewardRate).mul(1e18).div(_totalSupply)
        );
    }
```
Let X be the time interval between user actions and Y be the rewardPerTokenStored, we get the following equation:

Y + (X * 19_290 * 1e18 / 5_000_000e18)
If Y = 0 and X <= 250 the result will always be 0. 250 seconds is 4 minutes and 10 seconds. So if an attacker calls the stake() function every 4 minutes and stakes 1 WEI, and starts doing this once the rewards are deposited, no rewards will accrue. Resulting in stakers not receiving any rewards, and tokens being stuck in the contract, as there is no mechanism to withdraw them, and once some time has passed the rewards for that time can't be added to a next reward period. Lets say 10 days which is 864_000 seconds has passed 16_666_560_000 ≈ 16_666e6 tokens will be locked forever due to the way rewards are calculated if the current period hasn't finished:

```solidity
     uint256 remaining = rewardData[_rewardsToken].periodFinish.sub(block.timestamp);
     uint256 leftover = remaining.mul(rewardData[_rewardsToken].rewardRate);
     rewardData[_rewardsToken].rewardRate = reward.add(leftover).div(rewardData[_rewardsToken].rewardsDuration);
```

A malicious user can further dilute the rewardRate as the depositReward() function can be called by anyone with a reward parameter = 0, but this is an issue of it own. Lets say 10 days have passed,

remaining = 1_728_000 seconds
leftover = 1_728_000 * 19_290 = 33_333_120_000
rewardRate = 33_333_120_000 / 2_592_000 = 12_860

As we can see the rewardRate becomes a lot less, if rewards are diluted, this way the malicious user can call the stake() function less often and still achieve the same result. Or if the reward deposited is too big, and a malicious actor can't make the amount added to the rewardPerTokenStored to be equal to 0, no matter the time interval, lowering the rewardRate will help him achieve that. Due to the following line in the depositReward() function:

```solidity
      IERC20(_rewardsToken).safeTransferFrom(_msgSender(), address(this), reward);
```
rewards, that were not properly distributed can't be used in new reward periods.

## Vulnerability Detail
[Gist](https://gist.github.com/AtanasDimulski/e2edba2c03e4dd1325b9e73c8fd58ddb)
After following the steps in the above linked [gist](https://gist.github.com/AtanasDimulski/e2edba2c03e4dd1325b9e73c8fd58ddb) add the following test to AuditorTests.t.sol:

```solidity
    function test_RewardsWithLessDecimalsWillBeLost() public {
        vm.startPrank(ZVL);
        zivoeToken.transfer(alice, 5_000_000e18); // this will represent the total stake in the stZVE contract
        zivoeToken.transfer(attacker, 1e18);
        uint256 duration = 30 days;
        stZVE.addReward(address(mockUSDC), duration);
        vm.stopPrank();

        vm.startPrank(alice);
        zivoeToken.approve(address(stZVE), type(uint256).max);
        stZVE.stake(5_000_000e18);
        vm.stopPrank();

        vm.startPrank(simulateYLD);
        mockUSDC.mint(simulateYLD, 50_000e6); // this represent 50_000 USDC tokens to be distributed in 30 days
        mockUSDC.approve(address(stZVE), type(uint256).max);
        stZVE.depositReward(address(mockUSDC), 50_000e6);
        vm.stopPrank();

        vm.startPrank(attacker);
        zivoeToken.approve(address(stZVE), type(uint256).max);
        console2.log("Initial block timestamp: ", block.timestamp);
        skip(249);
        stZVE.stake(1);
        console2.log("The amount that alice have earned after ", block.timestamp, stZVE.earned(alice, address(mockUSDC)));
        skip(249);
        stZVE.stake(1);
        console2.log("The amount that alice have earned after: ", block.timestamp, stZVE.earned(alice, address(mockUSDC))); // after 498 seconds has passed reawrds should have been 5000000
        skip(249);
        stZVE.stake(1);
        console2.log("The amount that alice have earned after: ", block.timestamp, stZVE.earned(alice, address(mockUSDC))); // after 498 seconds has passed reawrds should have been 10000000
        vm.stopPrank();
    }
```

```solidity
Logs:
  Initial block timestamp:  1
  The amount that alice have earned after  250 0
  The amount that alice have earned after:  499 0
  The amount that alice have earned after:  748 0
```

To run the test use: ``forge test -vvv --mt test_RewardsWithLessDecimalsWillBeLost``

## Impact
Users that have staked their tokens won't receive rewards, and the reward tokens will be stuck in the contract, as there is no mechanism to withdraw them, or include them in a new reward distribution period.

## Code Snippet
https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeRewards.sol#L196-L203

## Tool used
Manual Review & Foundry

## Recommendation
Consider a way to convert tokens with decimals less than 18 to an amount with 18 decimals
## <a id='M-01'></a>[M-01] Rewards are calculated as distributed even if there are no stakers, locking the rewards forever
## Summary
The ZivoeRewards.sol and the ZivoeRewardsVesting.sol contracts are a fork of the Synthetix rewards distribution contract, with slight modifications. The contract logic keeps track of the total time passed, as well as the cumulative rate of rewards that have been generated for each token used for rewards, so that for every user, it just has to keep track of the last timestamp and last rate at the time of the last reward withdrawal, stake, or unstake in order to calculate the rewards owed since the user began staking. The code special-cases the scenario where there are no users, by not updating the cumulative rate when the _totalSupply is zero, but it does not include such a condition for the tracking of the timestamp. Because of this, even when there are no users staking, the accounting logic still thinks funds were being dispersed during that timeframe (because the starting timestamp is updated), which means the funds effectively are distributed to nobody, rather than being saved for when there is someone to receive them. And will be locked in the contract forever. One of the modifications is this line in the depositReward() function.

```solidity
      IERC20(_rewardsToken).safeTransferFrom(_msgSender(), address(this), reward);
```
Contrary to the Synthetix implementation, Zivo requires each time reward is deposited to the contact, the reward amount to be transferred in the same transaction. However if a reward is deposited but there are no stakers, the reward that should have been distributed for the period until the first user stakes, will be locked in the contract forever, and won't be able to be added to the rewards for the next reward period because of the above code snippet.

## Vulnerability Detail
[Gist](https://gist.github.com/AtanasDimulski/e2edba2c03e4dd1325b9e73c8fd58ddb)
After following the steps in the above linked [gist](https://gist.github.com/AtanasDimulski/e2edba2c03e4dd1325b9e73c8fd58ddb) add the following test to the AuditorTests.t.sol contract:

```solidity
    function test_LockedRewards() public {
        vm.startPrank(ZVL);
        zivoeToken.transfer(alice, 5_000_000e18);
        uint256 duration = 30 days;
        stZVE.addReward(address(mockUSDC), duration);
        vm.stopPrank();

        vm.startPrank(simulateYLD);
        mockUSDC.mint(simulateYLD, 50_000e6); // this represent 50_000 USDC tokens to be distributed in the next 30 days
        mockUSDC.approve(address(stZVE), type(uint256).max);
        stZVE.depositReward(address(mockUSDC), 50_000e6);
        skip(172_800); /// @notice two days pass, before anybody stakes in the contract
        vm.stopPrank();

        vm.startPrank(alice);
        console2.log("USDC balance of alice before staking: ", mockUSDC.balanceOf(alice));
        console2.log("USDC balance of stZVE contract: ", mockUSDC.balanceOf(address(stZVE)));
        zivoeToken.approve(address(stZVE), type(uint256).max);
        stZVE.stake(5_000_000e18);
        skip(duration);
        stZVE.fullWithdraw();
        console2.log("USDC balance of alice after staking for 30 days: ", mockUSDC.balanceOf(alice));
        console2.log("USDC balance of stZVE contract, when users have withdrawn everything for the first period: ", mockUSDC.balanceOf(address(stZVE)));
        console2.log("USDC balance of stZVE contract, when users have withdrawn everything for the first period normalized: ", mockUSDC.balanceOf(address(stZVE)) / 1e6);
        console2.log("zivoToken balance of alice after unstaking: ", zivoeToken.balanceOf(alice));
        vm.stopPrank();
    }
```

```solidity
Logs:
  USDC balance of alice before staking:  0
  USDC balance of stZVE contract:  50000000000
  USDC balance of alice after staking for 30 days:  46665000000
  USDC balance of stZVE contract, when users have withdrawn everything for the first period:  3335000000
  USDC balance of stZVE contract, when users have withdrawn everything for the first period normalized:  3335
  zivoToken balance of alice after unstaking:  5000000000000000000000000
```

As can be seen from the above logs 3335 USDC tokens will be locked forever.

To run the test use: ``forge test -vvv --mt test_LockedRewards``
## Impact
If the depositReward() function is called prior to there being any users staking, the funds that should have gone to the first stakers will instead accrue to nobody, and be locked in the contract forever.

## Code Snippet
https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeRewards.sol#L228-L243
## Tool used
Manual Review & Foundry

## Recommendation
Remove the safeTransfer() from the depositReward() function, and instead check if there are enough reward tokens already in the contract. Something similar to the Synthetix implementation.
