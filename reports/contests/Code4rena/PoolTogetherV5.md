# What is Pool Together?

PoolTogether is a no-loss prize savings protocol that enables you to win by saving.

[Link to the contest](https://code4rena.com/audits/2023-08-pooltogether-v5-part-deux#top)

## Issues found by me

| Severity | Title                                                                                                       | Link                                                         |
| :------- | :---------------------------------------------------------------------------------------------------------- | :----------------------------------------------------------- |
| [Analysis](#A)     | Grade B analysis | [Link](https://github.com/code-423n4/2023-08-pooltogether-findings/blob/main/data/cholakov-Analysis.md)  |


## <a id='A'></a>Grade B Analysis  

Author handle: cholakov

## Project overview

PoolTogether is a no-loss prize savings protocol that enables you to win by saving. You deposit USDC and have a chance to win prizes every day. You can withdraw your deposit at any time, even if you don't win. PoolTogether is provably fair, globally accessible, fully non-custodial, open-source and secure, and decentralized.

The codebase is separated into 4 repositories:
- [pt-v5-cgda-liquidator](#cgda-liquidator)
- [pt-v5-draw-auction](#draw-auction)
- [pt-v5-vault-boost](#vault-boost)
- [remote-owner ](#remote-owner)


## What's New in V5?

V5 of PoolTogether brings massive improvements. The protocol is now autonomous and permissionless.

1. **Autonomous:** There is no central entity controlling the protocol. This means that the protocol is self-governing and can't be shut down or tampered with by any one person or group.

2. **Permissionless:** Anyone can add new assets and yield sources to the protocol. This means that the protocol is open to innovation and can be easily adapted to new market conditions.

## Protocol Design

- Fundamentally, PoolTogether's value proposition is high-variance yield.** This means that users have the opportunity to earn a significantly higher yield than they would if they were simply earning interest on their deposits. However, there is also a higher risk involved, as users may not win any yield at all.

- Instead of earning their own yield, users have a chance to win everyone's yield.** This is because the yield that is generated from all of the deposits in the pool is randomly distributed to the users. This means that there is no guarantee that any individual user will earn any yield, but there is also the potential to win a significant amount.

- PoolTogether V5 combines yield from any number of assets in a autonomous, permissionless way.** This means that the protocol is not controlled by any central authority, and anyone can add new assets to the pool. This makes the protocol more flexible and adaptable to new market conditions.

![Example Image](https://drive.google.com/uc?id=1hoB0B6Ntw9jIlHSjPcDpcS4kvB6jRr_U)

## CGDA Liquidator

PoolTogether V5 uses the CGDA liquidator to sell yield for POOL tokens and contribute those tokens to the prize pool. 

## What is CGDA (Continuous Gradual Dutch Auction)?

CGDAs are a type of auction mechanism used for selling tokens or assets. In CGDAs, the time intervals between each auction approach zero, meaning that sales are split into an infinite sequence of tiny auctions. Each of these auctions sells an almost negligible amount of the token.

The uniqueness of CGDAs lies in their ability to efficiently determine the purchase price for any quantity of tokens. This is because the computation of prices in CGDAs is gas-efficient, even though there are an infinite number of auctions. This is crucial in blockchain environments where gas costs are a consideration.

CGDAs provide an alternative to discrete auction mechanisms, where tokens are sold in fixed-sized lots or batches. By taking the auction process to the limit and continuously dividing sales, CGDAs give participants the flexibility to purchase tokens in the quantity they desire. This is beneficial for both buyers and sellers. Buyers can purchase the exact amount of tokens they need, while sellers can ensure an optimal selling strategy that aligns with their needs.


#### The CGDA liquidator is a set of three contracts:

## LiquidationPair: 

The `LiquidationPair.sol` contract represents a token pair involved in a periodic CGDA mechanism. The contract combines two primary components: a pricing mechanism for the auction and an external liquidation source responsible for executing token swaps. Below is an analysis of the key aspects of the contract:

1. **Auction Mechanism**:

The contract utilizes the Continuous GDA auction mechanism, where the auction price continuously decays over time. This allows for more flexible and dynamic pricing, enabling participants to buy or sell tokens at varying rates based on the auction duration and other parameters.

2. **Auction Parameters**:

The contract utilizes several auction parameters to configure the auction behavior:

```
  constructor(
    ILiquidationSource _source,
    address _tokenIn,
    address _tokenOut,
    uint32 _periodLength,
    uint32 _periodOffset,
    uint32 _targetFirstSaleTime,
    SD59x18 _decayConstant,
    uint112 _initialAmountIn,
    uint112 _initialAmountOut,
    uint256 _minimumAuctionAmount
  ) {
    source = _source;
    tokenIn = _tokenIn;
    tokenOut = _tokenOut;
    decayConstant = _decayConstant;
    periodLength = _periodLength;
    periodOffset = _periodOffset;
    targetFirstSaleTime = _targetFirstSaleTime;
    /// more code
```

3. **Token Swap and Liquidation**:

The contract relies on an external ILiquidationSource contract to execute token swaps during the auction. The liquidation source handles the actual exchange of tokens at the auction price determined by the "LiquidationPair" contract.

The `swapExactAmountOut` and `computeExactAmountIn` functions handle token swaps. Users can provide either the amount of tokens they want to receive (`_amountOut`) or the amount of tokens they want to spend (`_amountInMax`). The contract calculates the exact amount of tokens required for the swap based on the current auction price and emission rate.

[swapExactAmountOut](https://github.com/GenerationSoftware/pt-v5-cgda-liquidator/blob/7f95bcacd4a566c2becb98d55c1886cadbaa8897/src/LiquidationPair.sol#L211C2-L226C4)
```
 function swapExactAmountOut(
    address _account,
    uint256 _amountOut,
    uint256 _amountInMax
  ) external returns (uint256) {
    _checkUpdateAuction();
    uint swapAmountIn = _computeExactAmountIn(_amountOut);
    if (swapAmountIn > _amountInMax) {
      revert SwapExceedsMax(_amountInMax, swapAmountIn);
    }
    _amountInForPeriod += uint96(swapAmountIn);
    _amountOutForPeriod += uint96(_amountOut);
    _lastAuctionTime += uint48(uint256(convert(convert(int256(_amountOut)).div(_emissionRate))));
    source.liquidate(_account, tokenIn, swapAmountIn, tokenOut, _amountOut);
    return swapAmountIn;
  }
```

[_computeExactAmountIn](https://github.com/GenerationSoftware/pt-v5-cgda-liquidator/blob/7f95bcacd4a566c2becb98d55c1886cadbaa8897/src/LiquidationPair.sol#L297C3-L319C4)
```
function _computeExactAmountIn(uint256 _amountOut) internal returns (uint256) {
    if (_amountOut == 0) {
      return 0;
    }
    uint256 maxOut = _maxAmountOut();
    if (_amountOut > maxOut) {
      revert SwapExceedsAvailable(_amountOut, maxOut);
    }
    SD59x18 elapsed = _getElapsedTime();
    uint purchasePrice = uint256(convert(ContinuousGDA.purchasePrice(
        convert(int(_amountOut)),
        _emissionRate,
        _initialPrice,
        decayConstant,
        elapsed
      ).ceil()));

    if (purchasePrice == 0) {
      revert PurchasePriceIsZero(_amountOut);
    }

    return purchasePrice;
  }
```

4. **Gas Efficiency Consideration**:

The contract implements a `minimumAuctionAmount `parameter, which requires a minimum number of tokens before triggering an auction. This ensures that gas costs are not disproportionately high compared to the value of the auctioned tokens. If the minimum is not met, the auction won't proceed, saving gas costs.

The contract aims to be gas-efficient by carefully calculating emission rates and token amounts to avoid unnecessary computations.

[minimumAuctionAmount](https://github.com/GenerationSoftware/pt-v5-cgda-liquidator/blob/7f95bcacd4a566c2becb98d55c1886cadbaa8897/src/LiquidationPair.sol#L54C2-L54C49)
```
uint256 public immutable minimumAuctionAmount;
```

5. **Error Handling**:

The contract includes various error functions to handle specific exceptional cases, such as zero amounts for tokens, incorrect timing settings, insufficient available tokens for swaps, and excessively large decay constants.

[Custom Errors](https://github.com/GenerationSoftware/pt-v5-cgda-liquidator/blob/7f95bcacd4a566c2becb98d55c1886cadbaa8897/src/LiquidationPair.sol#L10C1-L16C46)
```
error AmountInZero();
error AmountOutZero();
error TargetFirstSaleTimeLtPeriodLength(uint passedTargetSaleTime, uint periodLength);
error SwapExceedsAvailable(uint256 amountOut, uint256 available);
error SwapExceedsMax(uint256 amountInMax, uint256 amountIn);
error DecayConstantTooLarge(SD59x18 maxDecayConstant, SD59x18 decayConstant);
error PurchasePriceIsZero(uint256 amountOut);
```

6. **Contract Updation**:

The contract continuously checks for updates to the auction period and emission rate to ensure the pricing remains up to date.

[_checkUpdateAuction](https://github.com/GenerationSoftware/pt-v5-cgda-liquidator/blob/7f95bcacd4a566c2becb98d55c1886cadbaa8897/src/LiquidationPair.sol#L322C2-L327C4)
```
 function _checkUpdateAuction() internal {
    uint256 currentPeriod = _computePeriod();
    if (currentPeriod != _period) {
      _updateAuction(currentPeriod);
    }
  }
```

7. **Conclusion:**

The `LiquidationPair.sol` contract facilitates a dynamic and flexible token auction mechanism that continuously adapts pricing over time. The use of CDA allows for more efficient and fair token swaps, benefiting both sellers and buyers. However, users must carefully consider the auction parameters to ensure an optimal trading experience and gas efficiency. Additionally, the integration with an external liquidity source adds flexibility and ensures proper token swaps at the determined auction prices.


## LiquidationPairFactory:

The primary functionality of the `LiquidationPairFactory.sol` contract is to create new instances of `LiquidationPair.sol` contracts. This is achieved through the createPair function, which takes various parameters required to configure the auction mechanism for the new pair. The function deploys a new "LiquidationPair" contract and registers it within the factory. 

[createPair](https://github.com/GenerationSoftware/pt-v5-cgda-liquidator/blob/7f95bcacd4a566c2becb98d55c1886cadbaa8897/src/LiquidationPairFactory.sol#L65C3-L109C1)
```
 function createPair(
    ILiquidationSource _source,
    address _tokenIn,
    address _tokenOut,
    uint32 _periodLength,
    uint32 _periodOffset,
    uint32 _targetFirstSaleTime,
    SD59x18 _decayConstant,
    uint112 _initialAmountIn,
    uint112 _initialAmountOut,
    uint256 _minimumAuctionAmount
  ) external returns (LiquidationPair) {
  /// more code...
```

Once the new pair is successfully created and registered, the contract emits a `PairCreated` event. This event provides important details about the newly created pair, including the pair address, the liquidation source used, the input and output tokens, the auction parameters, and the initial amounts.

[PairCreated Event](https://github.com/GenerationSoftware/pt-v5-cgda-liquidator/blob/7f95bcacd4a566c2becb98d55c1886cadbaa8897/src/LiquidationPairFactory.sol#L93C5-L105C7)
```
emit PairCreated(
      _liquidationPair,
      _source,
      _tokenIn,
      _tokenOut,
      _periodLength,
      _periodOffset,
      _targetFirstSaleTime,
      _decayConstant,
      _initialAmountIn,
      _initialAmountOut,
      _minimumAuctionAmount
    );
```

## LiquidationRouter:
 
 The `LiquidationRouter.sol` contract is a user-facing swapping interface for Liquidation Pairs. It allows users to swap tokens using the CGDA mechanism provided by the `LiquidationPair.sol` contracts.

 The primary functionality of the contract is to facilitate token swaps using the CGDA mechanism. The `swapExactAmountOut` function allows users to swap a given amount of output tokens for at most a specified number of input tokens. The function interacts with the corresponding `LiquidationPair.sol` contract to execute the swap.

[swapExactAmountOut](https://github.com/GenerationSoftware/pt-v5-cgda-liquidator/blob/7f95bcacd4a566c2becb98d55c1886cadbaa8897/src/LiquidationRouter.sol#L63C3-L80C4)
 ```
 function swapExactAmountOut(
    LiquidationPair _liquidationPair,
    address _receiver,
    uint256 _amountOut,
    uint256 _amountInMax
  ) external onlyTrustedLiquidationPair(_liquidationPair) returns (uint256) {
    IERC20(_liquidationPair.tokenIn()).safeTransferFrom(
      msg.sender,
      _liquidationPair.target(),
      _liquidationPair.computeExactAmountIn(_amountOut)
    );

    uint256 amountIn = _liquidationPair.swapExactAmountOut(_receiver, _amountOut, _amountInMax);

    emit SwappedExactAmountOut(_liquidationPair, _receiver, _amountOut, _amountInMax, amountIn);

    return amountIn;
  }
 ```

## Draw Auction:

The Draw Auction is a suite of contracts that auctions off the transactions required to push a new random number to the Prize Pool. This is necessary because PoolTogether V5 Prize Pools have periodic Draws, where a random number is used to distribute the next batch of prizes to the users.

## How Draw Auction works?

1. The first auction is for the initial RNG request. This auction is held on L1, and the winner is awarded a reward that is a fraction of the available Prize Pool reserve.

2. The winner of the first auction then requests a random number from an RNG service on Ethereum.
Once the random number is received, the winner of the first auction relays it to the RngRelayAuction contract.

3. The RngRelayAuction then holds an auction for the relaying of the random number to L2. The winner of this auction is awarded a reward that is a fraction of the available Prize Pool reserve.

4. Once the random number is relayed to L2, the Prize Pool closes the draw and distributes prizes to the users.

#### The Draw Auction is a set of four contracts:

## RngAuction:

The `RngAuction.sol` contract facilitates the initiation of random number generation (RNG) requests using the RNG service set for the contract. It incentivizes RNG requests to be started in sync with prize pool draw periods across all chains. Below are the key aspects of the contract:

1.**RNG Service and Auction Duration:**

The contract depends on an external RNG service contract (RNGInterface) for generating random numbers. The contract also includes an auction duration, representing the time duration for which the auction will be open.

2.**Auction Target Time:**

The contract sets a target time for completing the auction. The auction should be completed within the specified target time. If the auction is not completed within this time, certain functions may revert.

3.**Sequence period and offset:**

The contract introduces a sequence period representing the duration of the sequence that the auction should align with. Additionally, there is a sequence offset representing the offset of the sequence in seconds. This ensures that the auction aligns with the correct sequence.

4.**StartRngRequest function:**

The main function of the contract is the startRngRequest function. This function starts an RNG request, ends the current auction, and stores the reward fraction to be allocated to the recipient. The auction is considered complete when the RNG has been requested for the current sequence.

5.**Auction results:**

After the auction is completed, the contract emits an event with the relevant information, including the recipient of the auction rewards, the sequence ID for the auction, the RNG service used, the elapsed time for the auction, and the reward fraction received by the recipient.

6.**Auction status and reward calculation:**

The contract provides several functions to check the status of the auction, such as whether it can start the next sequence, whether the auction is open, the auction elapsed time, and the current fractional reward based on the elapsed time. The contract uses `RewardLib`` to compute the reward fraction for the auction based on the elapsed time, auction duration, auction target time fraction, and the last sold fraction.


## RngAuctionRelayerDirect:

The `RngAuctionRelayerDirect.sol` contract is an implementation of a relay mechanism that allows anyone to trigger the relay of Random Number Generator (RNG) results to an "IRngAuctionRelayListener." The relay mechanism facilitates the transfer of RNG results from the "RngAuction" contract to a designated relay listener contract.

The main functionality of the contract is the `relay` function. This function allows anyone to trigger the relay of RNG results to a specific `IRngAuctionRelayListener` contract. Before calling the relay listener, the function encodes the required data and calls the listener contract using the low-level "call" function.

[relay](https://github.com/GenerationSoftware/pt-v5-draw-auction/blob/f1c6d14a1772d6609de1870f8713fb79977d51c1/src/RngAuctionRelayerDirect.sol#L34C5-L45C6)
```
function relay(
        IRngAuctionRelayListener _rngAuctionRelayListener,
        address _relayRewardRecipient
    ) external returns (bytes memory) {
        bytes memory data = encodeCalldata(_relayRewardRecipient);
        (bool success, bytes memory returnData) = address(_rngAuctionRelayListener).call(data);
        if (!success) {
            revert DirectRelayFailed(returnData);
        }
        emit DirectRelaySuccess(_relayRewardRecipient, returnData);
        return returnData;
    }
```

## RngAuctionRelayerRemoteOwner:

The `RngAuctionRelayerRemoteOwner.sol` contract is an implementation of a relay mechanism that allows anyone to relay Random Number Generator results to an `IRngAuctionRelayListener` contract on another chain. The contract uses a Remote Owner, which enables a contract on one chain to operate an address on another chain through cross-chain communication.

The main functionality of the contract is the `relay` function, which allows anyone to trigger the relay of RNG results to an `IRngAuctionRelayListener` contract on the destination chain. The relay is achieved using the ERC-5164 Dispatcher.

[relay](https://github.com/GenerationSoftware/pt-v5-draw-auction/blob/f1c6d14a1772d6609de1870f8713fb79977d51c1/src/RngAuctionRelayerRemoteOwner.sol#L57C4-L69C6)
```
 function relay(
        IRngAuctionRelayListener _remoteRngAuctionRelayListener,
        address rewardRecipient
    ) external returns (bytes32) {
        bytes memory listenerCalldata = encodeCalldata(rewardRecipient);
        bytes32 messageId = messageDispatcher.dispatchMessage(
            toChainId,
            address(account),
            RemoteOwnerCallEncoder.encodeCalldata(address(_remoteRngAuctionRelayListener), 0, listenerCalldata)
        );
        emit RelayedToDispatcher(rewardRecipient, messageId);
        return messageId;
    }
```

## RngRelayAuction:

The `RngRelayAuction.sol` contract is a sophisticated piece of code that orchestrates the auction process for the Random Number Generator (RNG) relay and then closes the Prize Pool using the RNG results. The contract uses a variety of libraries and interfaces to manage the auction process and reward distribution, ensuring that it is fair and transparent.

The contract uses fixed-point arithmetic for accurate reward calculations and provides functions to interact with the auction and compute rewards efficiently. This ensures that participants are rewarded fairly based on their contribution and the elapsed auction time.

Overall, the `RngRelayAuction.sol` contract is a valuable tool for managing the auction process for the RNG relay and closing the Prize Pool using the RNG results.

## Vault Boost

The Vault Boost allows anyone to boost the winning chances of all users of a vault. The Vault Booster can liquidate any tokens and contribute them to the prize pool on behalf of the target vault.

#### The Vault Boost is a set of two contracts:

## VaultBooster

The `VaultBooster.sol` contract is a sophisticated piece of code that allows users to liquidate arbitrary tokens for a vault and improve the vault's chance of winning. The contract contributes to a Prize Pool and boosts the chances of the associated vault by providing liquidation pairs and boost configurations for specific tokens.

The boost configurations are set by the owner of the contract and can be adjusted over time. The contract contributes to the Prize Pool and rewards the liquidation pairs for their contributions. This potentially improves the vault's performance in the Prize Pool competition.

## VaultBoosterFactory

The `VaultBoosterFactory.sol` contract is a factory contract responsible for creating instances of the `VaultBooster` contract. It allows users to deploy new `VaultBooster` contracts, each tailored to a specific Prize Pool and vault.

## Remote Owner

The `RemoteOwner.sol` contract is a sophisticated piece of code that allows an account on one chain to have a "remote" account on another chain using an ERC-5164 compatible bridge. The contract uses ERC-5164 so that the bridge layer is swappable.

This makes it possible for a governance system on one chain to extend itself to another chain. For example, a governance system on Ethereum could deploy a `RemoteOwner` contract on Optimism and set the owner to be the governance address and the `fromChainId` to be 1. The governance system on Ethereum could then send execution messages through a ERC-5164 bridge layer to the `RemoteOwner`. The `RemoteOwner` contract would execute the transactions, effectively acting as governance on Optimism.




### Time spent:
10 hours

