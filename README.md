### 1. Objective of this review

- to explain design consideration necessity for Hub liquidity module to AiB
- to introduce brief review of iris-coinswap repository

### 2. Comparisons among implementations

Difference in specs result in huge design structure for implementation. We listed several feature comparisons to explain the reason why we need significant redesign considerations.

| feature                             | Uniswap               | iris-coinswap         | Hub liqudity module |
|-------------------------------------|-----------------------|-----------------------|---------------------|
|Swap Execution Price                 |Each Different Swap    |Each Different Swap    |Universal Swap Price |
|Free Swap Fee for Self-Matched Swaps |X                      |X                      |O                    |
|Passive Swap with Waiting Period     |X                      |X                      |O                    |
|Base Token                           |No Base Token Concept  |One Base Token         |Multiple Base Token  |
|Flash Swap                           |X                      |O                      |O                    |
|Generalized Swap Price Function      |X                      |X                      |O                    |

### 3. Design considerations

**1) Universal swap price**

- our basic swap execution mechanism uses universal swap price for all swaps executed in one block.
- iris-coinswap swap model uses different swap prices for each swap transaction, so the objects and functions should be modified to deploy universal swap price execution model.

**2) Free swap fee for self-matched swaps**

- our universal swap price relates to free swap fee for self-matched swaps which are not considered in iris-coinswap fee calculation model.
- we need to redesign the structure so that we can comply with differentiation of swap fee calculation to increase user benefit.
- the new design should consider further expansion of fee differentiation model to flexibly evolve under economic market feedbacks.

**3) Passive swap with waiting period**

- Uniswap has significant disadvantage for swap users to account too much slippage cost to each swap.
- our model tackle this problem by introducing new type of swap, passive swap, which only can be executed when there exists residual active(immediate) swaps in the block.
- the model will be expanded to dynamically launch a waiting period for a pool to accept additional passive swap orders to digest significantly large one side swap orders are received for a block. this dynamic liquidity management model will give our Hub DeX better ability to absorb larger size of swap demands from the market without consuming too much slippage cost or liquidity of the pool.
- iris-coinswap do not anticipate this problem so the codebase structure should be largely redesigned.

**4) Multiple base token assumption**

- our pool model has a governance parameter called baseTokenList. For any pool, at least one of the token of the pair should be an element of baseTokenList to concentrate liquidity of entire marketplace
- iris-coinswap assumes one baseToken so it needs a reconstruction of design

**5) Flash swap**

- flash swap is very useful economic functionality for Uniswap to make arbitrage activities more open to wider community members without significant capital possession. it makes arbitrage trading more competitive caused by lots of developers building and operating arbitrage machines which can be utilized without much of capital necessity. this ultimately result in efficiency of price discovery for pools to have much less impermenant losses from price volatilities.
- our module should be designed wisely so that flash swap can be easily adopted in future to acquire the efficient price discovery characteristics. iris-coinswap design does not consider this future at all.

**6) Generalized swap price derivation function design**

- we expect that the swap price function will be innovated to see many different candidates. for example, curve.finance uses different swap price function for stablecoin-stablecoin pools, which makes them biggest liquidity pool platform for stablecoin swap use-cases.
- our module will anticipate this kind of open-future to design the swap price function in a more generalized way. iris-coinswap module hardcoded the constant production function.

**7) Lack of testing codes**

- iris-coinswap repository has no testing codes, which will cost us several weeks to have complete testing codes for checking the functionalities of each part of the codebase

### 4. C**onclusion**

**Reusing existing codes is not an efficient approach** 

- we expect that it will cost even more time by reusing currently existing liquidity modules than constructing it from scratch because existing modules needs heavy redesigning to comply with more generalized and flexible designs for necessary feature for near future.
- the result codebase by reusing existing codes will be relatively less concise and clean.

**Existing codes can be referenced for designing and implementation**

- Existing codes can be helpful for us as references for us to more quickly build Hub liquidity module.
