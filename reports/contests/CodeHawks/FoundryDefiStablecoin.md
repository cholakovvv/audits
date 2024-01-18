# What is Foundry DeFi Stablecoin?

This project is meant to be a stablecoin where users can deposit WETH and WBTC in exchange for a token that will be pegged to the USD. The system is meant to be such that someone could fork this codebase, swap out WETH & WBTC for any basket of assets they like, and the code would work the same.

[Link to the contest](https://www.codehawks.com/contests/cljx3b9390009liqwuedkn0m0)

# Issues found by me

| Severity | Title                                                                                                       | Link                                                         |
| :------- | :---------------------------------------------------------------------------------------------------------- | :----------------------------------------------------------- |
| [M-01](#M-01)     | OracleLib will return the wrong price for asset if underlying aggregator hits minAnswer | [Link](https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/issues/314)  |
| [M-02](#M-02)     | Chainlink’s latestRoundData might return stale or incorrect results | [Link](https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/issues/312)  |

## <a id='M-01'></a>M-01. OracleLib will return the wrong price for asset if underlying aggregator hits minAnswer            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/main/src/libraries/OracleLib.sol#L21C3-L33C6

## Summary

OracleLib will return the wrong price for asset if underlying aggregator hits minAnswer

## Vulnerability Details

Chainlink aggregators have a built in circuit breaker if the price of an asset goes outside of a predetermined price band. The result is that if an asset experiences a huge drop in value (i.e. LUNA crash) the price of the oracle will continue to return the minPrice instead of the actual price of the asset. This would allow user to continue borrowing with the asset but at the wrong price. This is exactly what happened to Venus on BSC when LUNA imploded: https://rekt.news/venus-blizz-rekt/.

```solidity
 function staleCheckLatestRoundData(AggregatorV3Interface priceFeed)
        public
        view
        returns (uint80, int256, uint256, uint256, uint80)
    {
        (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound) =
            priceFeed.latestRoundData();

        uint256 secondsSince = block.timestamp - updatedAt;
        if (secondsSince > TIMEOUT) revert OracleLib__StalePrice();

        return (roundId, answer, startedAt, updatedAt, answeredInRound);
    }
```

`latestRoundData` pulls the associated aggregator and requests round data from it. ChainlinkAggregators have minPrice and maxPrice circuit breakers built into them. This means that if the price of the asset drops below the minPrice, the protocol will continue to value the token at minPrice instead of it's actual value. 

## Impact

This will allow users to take out huge amounts of bad debt and bankrupt the protocol.

## Tools Used

Manual review

## Recommendations

ChainlinkOracle should check the returned answer against the minPrice/maxPrice and revert if the answer is outside of bounds:

```solidity
(uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound) =
            priceFeed.latestRoundData();

        uint256 secondsSince = block.timestamp - updatedAt;
        if (secondsSince > TIMEOUT) revert OracleLib__StalePrice();

        if (answer >= maxPrice or answer <= minPrice) revert(); <-- add this
```

## <a id='M-02'></a>M-02. Chainlink’s latestRoundData might return stale or incorrect results            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/main/src/libraries/OracleLib.sol#L21C3-L33C6

## Summary

Chainlink’s latestRoundData might return stale or incorrect results

## Vulnerability Details

On OracleLib.sol is used `latestRoundData`, but there is no check if the return value indicates stale data.

```solidity
function staleCheckLatestRoundData(AggregatorV3Interface priceFeed)
        public
        view
        returns (uint80, int256, uint256, uint256, uint80)
    {
        (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound) =
            priceFeed.latestRoundData();

        uint256 secondsSince = block.timestamp - updatedAt;
        if (secondsSince > TIMEOUT) revert OracleLib__StalePrice();

        return (roundId, answer, startedAt, updatedAt, answeredInRound);
    }
```

## Impact

This could lead to stale prices according to the Chainlink documentation:

https://docs.chain.link/data-feeds/historical-data

https://docs.chain.link/docs/faq/#how-can-i-check-if-the-answer-to-a-round-is-being-carried-over-from-a-previous-round

## Tools Used

Manual review

## Recommendations

Consider adding the missing checks for stale data.

For example:

```solidity
 (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound) =
            priceFeed.latestRoundData();

uint256 secondsSince = block.timestamp - updatedAt;
        if (secondsSince > TIMEOUT) revert OracleLib__StalePrice();

require(answer > 0, "Chainlink price <= 0"); <-- add this
require(answeredInRound >= roundID, "Stale price"); <-- add this
require(timestamp != 0, "Round not complete"); <-- add this
```