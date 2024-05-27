# [Axis Finance](https://audits.sherlock.xyz/contests/206/report)

## Findings by Dimulski
| Severity | Title | 
|:--:|:---|
| [H-01](#H-01) | Overflow in curate() function, results in permanently stuck funds | 
| [H-02](#H-02) | Incorrect setting of lotId, makes the whole protocol obsolete | 
| [M-01](#M-01) | Unsold tokens from a FPAM auction, will be stuck in the protocol, after the auction concludes | 

## <a id='H-01'></a>[H-01] Overflow in curate() function, results in permanently stuck funds
## Summary
The Axis-Finance protocol has a curate() function that can be used to set a certain fee to a curator set by the seller for a certain auction. Typically, a curator is providing some service to an auction seller to help the sale succeed. This could be doing diligence on the project and vouching for them, or something simpler, such as listing the auction on a popular interface. A lot of memecoins have a big supply in the trillions, for example SHIBA INU has a total supply of nearly 1000 trillion tokens and each token has 18 decimals. With a lot of new memecoins emerging every day due to the favorable bullish conditions and having supply in the trillions, it is safe to assume that such protocols will interact with the Axis-Finance protocol. Creating auctions for big amounts, and promising big fees to some celebrities or influencers to promote their project. The funding parameter in the Routing struct is of type uint96

```solidity
    struct Routing {
        ...
        uint96 funding; 
        ...
    }
```

The max amount of tokens with 18 decimals a uint96 variable can hold is around 80 billion. The problem arises in the curate() function, If the auction is prefunded, which all batch auctions are( a normal FPAM auction can also be prefunded), and the amount of prefunded tokens is big enough, close to 80 billion tokens with 18 decimals, and the curator fee is for example 7.5%, when the curatorFeePayout is added to the current funding, the funding will overflow.

```solidity
unchecked {
   routing.funding += curatorFeePayout;
}
```

## Vulnerability Detail
[Gist](https://gist.github.com/AtanasDimulski/a47112fc7ae473fd69b42ba997819726)
After following the steps in the above mentioned [gist](https://gist.github.com/AtanasDimulski/a47112fc7ae473fd69b42ba997819726), add the following test to the AuditorTests.t.sol

```solidity
function test_CuratorFeeOverflow() public {
        vm.startPrank(alice);
        Veecode veecode = fixedPriceAuctionModule.VEECODE();
        Keycode keycode = keycodeFromVeecode(veecode);
        bytes memory _derivativeParams = "";
        uint96 lotCapacity = 75_000_000_000e18; // this is 75 billion tokens
        mockBaseToken.mint(alice, 100_000_000_000e18);
        mockBaseToken.approve(address(auctionHouse), type(uint256).max);

        FixedPriceAuctionModule.FixedPriceParams  memory myStruct = FixedPriceAuctionModule.FixedPriceParams({
            price: uint96(1e18),
            maxPayoutPercent: uint24(1e5)
        });

        Auctioneer.RoutingParams memory routingA = Auctioneer.RoutingParams({
            auctionType: keycode,
            baseToken: mockBaseToken,
            quoteToken: mockQuoteToken,
            curator: curator,
            callbacks: ICallback(address(0)),
            callbackData: abi.encode(""),
            derivativeType: toKeycode(""),
            derivativeParams: _derivativeParams,
            wrapDerivative: false,
            prefunded: true
        });

        Auction.AuctionParams memory paramsA = Auction.AuctionParams({
            start: 0,
            duration: 1 days,
            capacityInQuote: false,
            capacity: lotCapacity,
            implParams: abi.encode(myStruct)
        });

        string memory infoHashA;
        auctionHouse.auction(routingA, paramsA, infoHashA);       
        vm.stopPrank();

        vm.startPrank(owner);
        FeeManager.FeeType type_ = FeeManager.FeeType.MaxCurator;
        uint48 fee = 7_500; // 7.5% max curator fee
        auctionHouse.setFee(keycode, type_, fee);
        vm.stopPrank();

        vm.startPrank(curator);
        uint96 fundingBeforeCuratorFee;
        uint96 fundingAfterCuratorFee;
        (,fundingBeforeCuratorFee,,,,,,,) = auctionHouse.lotRouting(0);
        console2.log("Here is the funding normalized before curator fee is set: ", fundingBeforeCuratorFee/1e18);
        auctionHouse.setCuratorFee(keycode, fee);
        bytes memory callbackData_ = "";
        auctionHouse.curate(0, callbackData_);
        (,fundingAfterCuratorFee,,,,,,,) = auctionHouse.lotRouting(0);
        console2.log("Here is the funding normalized after curator fee is set: ", fundingAfterCuratorFee/1e18);
        console2.log("Balance of base token of the auction house: ", mockBaseToken.balanceOf(address(auctionHouse))/1e18);
        vm.stopPrank();
    }
```

```solidity
Logs:
  Here is the funding normalized before curator fee is set:  75000000000
  Here is the funding normalized after curator fee is set:  1396837485
  Balance of base token of the auction house:  80625000000
```

To run the test use: forge test -vvv --mt test_CuratorFeeOverflow

## Impact
If there is an overflow occurs in the curate() function, a big portion of the tokens will be stuck in the Axis-Finance protocol forever, as there is no way for them to be withdrawn, either by an admin function, or by canceling the auction (if an auction has started, only FPAM auctions can be canceled), as the amount returned is calculated in the following way
```solidity
        if (routing.funding > 0) {
            uint96 funding = routing.funding;

            // Set to 0 before transfer to avoid re-entrancy
            routing.funding = 0;

            // Transfer the base tokens to the appropriate contract
            Transfer.transfer(
                routing.baseToken,
                _getAddressGivenCallbackBaseTokenFlag(routing.callbacks, routing.seller),
                funding,
                false
            );
            ...
        }
```

## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L665-L667

## Tool used
Manual review & Foundry

## Recommendation
Either remove the unchecked block

```solidity
unchecked {
   routing.funding += curatorFeePayout;
}
```
so that when overflow occurs, the transaction will revert, or better yet also change the funding variable type from uint96 to uint256 this way sellers can create big enough auctions, and provide sufficient curator fee in order to bootstrap their protocol successfully.
## <a id='H-02'></a>[H-02] Unsold tokens from a FPAM auction, will be stuck in the protocol, after the auction concludes
## Summary
The whole purpose of the Axis-Finance protocol is to allow users to create different auctions in order to bootstrap their tokens, or perform a partial sale. Auctions can be created via the AuctionHouse.sol contract by calling the auction() function. Each auction has an unique lotId that is used to store the parameters for the auction such as baseToken, quoteToken, price, etc. In the auction() function the lotId for each auction is set in the beginning of the function in the following way, and will always be 0.

```solidity
Routing storage routing = lotRouting[lotId];
```

The problem is that lotId is set before it is updated on line 194.

```solidity
lotId = lotCounter++;
```

## Impact
The whole protocol, can support only one auction, if another auction is created it will overwrite the initial one, making the protocol obsolete. If the first auction is prefunded, and afterwards another auction is created all tokens from the first auction will be stuck in the Axis-Finance protocol. Creating auctions is permisionless, and doesn't require any funds.

## Code Snippet
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/bases/Auctioneer.sol#L174

## Tool used
Manual Review

## Recommendation
Set the Routing struct

```solidity
Routing storage routing = lotRouting[lotId];
```

after lotId is updated

```solidity
lotId = lotCounter++;
```

on line 194 in the Auctioneer.sol contract


## <a id='M-01'></a>[M-01] Unsold tokens from a FPAM auction, will be stuck in the protocol, after the auction concludes
## Summary
The Axis-Finance protocol allows sellers to create two types of auctions: FPAM & EMPAM. An FPAM auction allows sellers to set a price, and a maxPayout, as well as create a prefunded auction. The seller of a FPAM auction can cancel it while it is still active by calling the cancel function which in turn calls the cancelAuction() function. If the auction is prefunded, and canceled while still active, all remaining funds will be transferred back to the seller. The problem arises if an FPAM prefunded auction is created, not all of the prefunded supply is bought by users, and the auction concludes. There is no way for the baseTokens still in the contract, to be withdrawn from the protocol, and they will be forever stuck in the Axis-Finance protocol. As can be seen from the below code snippet cancelAuction() function checks if an auction is concluded, and if it is the function reverts.

```solidity
    function _revertIfLotConcluded(uint96 lotId_) internal view virtual {
        // Beyond the conclusion time
        if (lotData[lotId_].conclusion < uint48(block.timestamp)) {
            revert Auction_MarketNotActive(lotId_);
        }

        // Capacity is sold-out, or cancelled
        if (lotData[lotId_].capacity == 0) revert Auction_MarketNotActive(lotId_);
    }
```

## Vulnerability Detail
[Gist](https://gist.github.com/AtanasDimulski/a47112fc7ae473fd69b42ba997819726)
After following the steps in the above mentioned [gist](https://gist.github.com/AtanasDimulski/a47112fc7ae473fd69b42ba997819726) add the following test to the AuditorTests.t.sol:

```solidity
    function test_FundedPriceAuctionStuckFunds() public {
        vm.startPrank(alice);
        Veecode veecode = fixedPriceAuctionModule.VEECODE();
        Keycode keycode = keycodeFromVeecode(veecode);
        bytes memory _derivativeParams = "";
        uint96 lotCapacity = 75_000_000_000e18; // this is 75 billion tokens
        mockBaseToken.mint(alice, lotCapacity);
        mockBaseToken.approve(address(auctionHouse), type(uint256).max);

        FixedPriceAuctionModule.FixedPriceParams  memory myStruct = FixedPriceAuctionModule.FixedPriceParams({
            price: uint96(1e18), 
            maxPayoutPercent: uint24(1e5)
        });

        Auctioneer.RoutingParams memory routingA = Auctioneer.RoutingParams({
            auctionType: keycode,
            baseToken: mockBaseToken,
            quoteToken: mockQuoteToken,
            curator: curator,
            callbacks: ICallback(address(0)),
            callbackData: abi.encode(""),
            derivativeType: toKeycode(""),
            derivativeParams: _derivativeParams,
            wrapDerivative: false,
            prefunded: true
        });

        Auction.AuctionParams memory paramsA = Auction.AuctionParams({
            start: 0,
            duration: 1 days,
            capacityInQuote: false,
            capacity: lotCapacity,
            implParams: abi.encode(myStruct)
        });

        string memory infoHashA;
        auctionHouse.auction(routingA, paramsA, infoHashA);       
        vm.stopPrank();

        vm.startPrank(bob);
        uint96 fundingBeforePurchase;
        uint96 fundingAfterPurchase;
        (,fundingBeforePurchase,,,,,,,) = auctionHouse.lotRouting(0);
        console2.log("Here is the funding normalized before purchase: ", fundingBeforePurchase/1e18);
        mockQuoteToken.mint(bob, 10_000_000_000e18);
        mockQuoteToken.approve(address(auctionHouse), type(uint256).max);
        Router.PurchaseParams memory purchaseParams = Router.PurchaseParams({
            recipient: bob,
            referrer: address(0),
            lotId: 0,
            amount: 10_000_000_000e18,
            minAmountOut: 10_000_000_000e18,
            auctionData: abi.encode(0),
            permit2Data: ""
        });
        bytes memory callbackData = "";
        auctionHouse.purchase(purchaseParams, callbackData);
        (,fundingAfterPurchase,,,,,,,) = auctionHouse.lotRouting(0);
        console2.log("Here is the funding normalized after purchase: ", fundingAfterPurchase/1e18);
        console2.log("Balance of seler of quote tokens: ", mockQuoteToken.balanceOf(alice)/1e18);
        console2.log("Balance of bob in base token: ", mockBaseToken.balanceOf(bob)/1e18);
        console2.log("Balance of auction house in base token: ", mockBaseToken.balanceOf(address(auctionHouse)) /1e18);
        skip(86401);
        vm.stopPrank();

        vm.startPrank(alice);
        vm.expectRevert(
            abi.encodeWithSelector(Auction.Auction_MarketNotActive.selector, 0)
        );
        auctionHouse.cancel(uint96(0), callbackData);
        vm.stopPrank();
    }
```

```solidity
Logs:
  Here is the funding normalized before purchase:  75000000000
  Here is the funding normalized after purchase:  65000000000
  Balance of seler of quote tokens:  10000000000
  Balance of bob in base token:  10000000000
  Balance of auction house in base token:  65000000000800
```

To run the test use: forge test -vvv --mt test_FundedPriceAuctionStuckFunds

## Impact
If a prefunded FPAM auction concludes and there are still tokens, not bought from the users, they will be stuck in the Axis-Finance protocol.

## Tool used
Manual Review & Foundry

## Recommendation
Implement a function, that allows sellers to withdraw the amount left for a prefunded FPAM auction they have created, once the auction has concluded.

