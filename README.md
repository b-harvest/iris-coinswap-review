### 1. Objective of this review

- To explain design consideration necessity for Hub liquidity module to AiB
- To introduce brief review of IRIS-coinswap repository

<br/><br/>

### 2. Comparisons among implementations

Difference in specs result in huge design structure for implementation. We listed several feature comparisons to explain the reason why we need significant redesign considerations.

| feature                             | Uniswap               | IRIS-coinswap         | Hub liqudity module |
|-------------------------------------|-----------------------|-----------------------|---------------------|
|Swap Execution Price                 |Each Different Swap    |Each Different Swap    |Universal Swap Price |
|Free Swap Fee for Self-Matched Swaps |X                      |X                      |O                    |
|Passive Swap with Waiting Period     |X                      |X                      |O                    |
|Base Token                           |No Base Token Concept  |One Base Token         |Multiple Base Token  |
|Flash Swap                           |O                      |X                      |O                    |
|Generalized Swap Price Function      |X                      |X                      |O                    |

<br/><br/>

### 3. Design considerations

<br/>

**1) Universal swap price**

- Our basic swap execution mechanism uses universal swap price for all swaps executed in one block.
- IRIS-coinswap swap model uses different swap prices for each swap transaction, so the objects and functions should be modified to deploy universal swap price execution model.
- Code comments
    - To calculate universal swap price, we need to accumulate all liquidity messages in the current queue, and then calculate the universal swap at [endBlocker](https://docs.cosmos.network/master/building-modules/beginblock-endblock.html#beginblock) from accumulated list of messages in current queue.
    - However, in IRIS-coinswap module, in [Handler](https://github.com/irismod/coinswap/blob/acd65c0955884b15265972812dcffc70d0a7b0d7/handler.go#L10-L26) where messages are diverging into [handleMsgSwapOrder](https://github.com/irismod/coinswap/blob/acd65c0955884b15265972812dcffc70d0a7b0d7/handler.go#L35), [handleMsgAddLiquidity](https://github.com/irismod/coinswap/blob/acd65c0955884b15265972812dcffc70d0a7b0d7/handler.go#L60-L62) and [handleMsgRemoveLiquidity](https://github.com/irismod/coinswap/blob/acd65c0955884b15265972812dcffc70d0a7b0d7/handler.go#L84-L86). They are separately handled so that the strucutre does not allow universal swap price calculation in one location.
    - To calculate universal swap price, each handler function should only validate and store messages in a queue, and [EndBlock](https://github.com/irismod/coinswap/blob/805fda07404c37225e5ca6465b141955996fcda7/module.go#L152-L154) needs to calculate universal swap price from the queue.

<br/>

**2) Free swap fee for self-matched swaps**

- Our universal swap price relates to free swap fee for self-matched swaps which are not considered in IRIS-coinswap fee calculation model.
- We need to redesign the structure so that we can comply with differentiation of swap fee calculation to increase user benefit.
- The new design should consider further expansion of fee differentiation model to flexibly evolve under economic market feedbacks.
- Code comments
    - To handle self-batched swaps, as in 1), we need to accumulate messages in a queue and execute self-matched swap algorithm in [EndBlock](https://github.com/irismod/coinswap/blob/805fda07404c37225e5ca6465b141955996fcda7/module.go#L152-L154).
    - We also need to adjust swap fee rate from self-matched swap ratio, so the self-matched swap ratio should be calculated in [EndBlock](https://github.com/irismod/coinswap/blob/805fda07404c37225e5ca6465b141955996fcda7/module.go#L152-L154) too.

<br/>

**3) Passive swap with waiting period**

- Uniswap has significant disadvantage for swap users to account too much slippage cost to each swap.
- Our model tackle this problem by introducing new type of swap, passive swap, which only can be executed when there exists residual active(immediate) swaps in the block.
- The model will be expanded to dynamically launch a waiting period for a pool to accept additional passive swap orders to digest significantly large one side swap orders are received for a block. This dynamic liquidity management model will give our Hub DeX better ability to absorb larger size of swap demands from the market without consuming too much slippage cost or liquidity of the pool.
- IRIS-coinswap do not anticipate this problem so the codebase structure should be largely redesigned.
- Code comments
    - Passive swap with waiting period needs queue structure for each liquidity module message to handle swap execution with accumulated message list in the queue.

<br/>

**4) Multiple base token assumption**

- Our pool model has a governance parameter called baseTokenList. For any pool, at least one of the token of the pair should be an element of baseTokenList to concentrate liquidity of entire marketplace
- IRIS-coinswap assumes one baseToken so it needs a reconstruction of design
- Code comments
    - [standardDenom](https://github.com/irismod/coinswap/blob/805fda07404c37225e5ca6465b141955996fcda7/internal/types/params.go#L12-L37) is defined as string, not list of strings.
    - Many codebases are assuming unique standardDenom, which results in necessity of large code reconstruction to allow multiple standardDenom
        - In [swap message diverging code](https://github.com/irismod/coinswap/blob/a04b9fdbc5dfa268076891225fadbe086ea81650/internal/keeper/keeper.go#L50-L66), it brings input/output coin denoms and compare to standardDenom to consider whether the swap is DoubleSwap or not.
        - PoolToken is defined with uniDenom which is the standardDenom. We need to allow it to have other denoms too, hence needs reconstruction of codebase.
    - In our implementation
        - We will have multiple standardDenom such as Atom, USDT or USDX so that the market can create liquidity pools in more free way.
        - We will not support DoubleSwap because it can be easily executed by multiple swap messages wrapped from frontend.

<br/>

**5) Flash swap**

- Flash swap is very useful economic functionality for Uniswap to make arbitrage activities more open to wider community members without significant capital possession. It makes arbitrage trading more competitive caused by lots of developers building and operating arbitrage machines which can be utilized without much of capital necessity. This ultimately result in efficiency of price discovery for pools to have much less impermenant losses from price volatilities.
- Our module should be designed wisely so that flash swap can be easily adopted in future to acquire the efficient price discovery characteristics. IRIS-coinswap design does not consider this future at all.
- Code comments
    - The transaction with flash swap messages should be atomically managed. It means that the messages should be success or fail in all-or-nothing way.
    - It needs deeper design consideration because the state machine need to deliverately gather information about failure of each message, which results in forced failure of rest of the messages in the transaction.

<br/>

**6) Generalized swap price derivation function design**

- We expect that the swap price function will be innovated to see many different candidates. for example, curve.finance uses different swap price function for stablecoin-stablecoin pools, which makes them biggest liquidity pool platform for stablecoin swap use-cases.
- Our module will anticipate this kind of open-future to design the swap price function in a more generalized way. IRIS-coinswap module hardcoded the constant production function.
- Code comments
    - IRIS-coinswap does not use independent generalized swap price derivation function
        - [https://github.com/irismod/coinswap/blob/387c7e3327d3e1191554120823725ccdfae15384/internal/keeper/swap.go#L30-L58](https://github.com/irismod/coinswap/blob/387c7e3327d3e1191554120823725ccdfae15384/internal/keeper/swap.go#L30-L58)
        - [https://github.com/irismod/coinswap/blob/387c7e3327d3e1191554120823725ccdfae15384/internal/keeper/swap.go#L127-L152](https://github.com/irismod/coinswap/blob/387c7e3327d3e1191554120823725ccdfae15384/internal/keeper/swap.go#L127-L152)
    - GetInputPrice, GetOutputPrice
        - [https://github.com/irismod/coinswap/blob/387c7e3327d3e1191554120823725ccdfae15384/internal/keeper/swap.go#L216-L234](https://github.com/irismod/coinswap/blob/387c7e3327d3e1191554120823725ccdfae15384/internal/keeper/swap.go#L216-L234)
    - Our solution needs to allow governance decided swap price derivation function to be used in liquidity pool. Also we need to consider different price functions for different categories of liquidity pool, so it needs further design consideration.

<br/>

**7) Lack of testing codes**

- IRIS-coinswap repository has no testing codes, which will cost us several weeks to have complete testing codes for checking the functionalities of each part of the codebase

```go
coinswap
├── README.md
├── alias.go
├── client
│   └── rest
│       ├── query.go
│       ├── rest.go
│       └── tx.go
├── genesis.go
├── go.mod
├── go.sum
├── handler.go
├── internal
│   ├── keeper
│   │   ├── keeper.go
│   │   ├── querier.go
│   │   └── swap.go
│   └── types
│       ├── codec.go
│       ├── errors.go
│       ├── events.go
│       ├── expected_keepers.go
│       ├── genesis.go
│       ├── keys.go
│       ├── msgs.go
│       ├── params.go
│       ├── querier.go
│       └── utils.go
├── module.go
└── simulation
    ├── decoder.go
    ├── genesis.go
    └── params.go

6 directories, 26 files
```
<br/><br/>

### 4. **Conclusion**

<br/>

**Reusing existing codes is not an efficient approach** 

- we expect that it will cost even more time by reusing currently existing liquidity modules than constructing it from scratch because existing modules needs heavy redesigning to comply with more generalized and flexible designs for necessary feature for near future.
- the result codebase by reusing existing codes will be relatively less concise and clean.

<br/>

**Existing codes can be referenced for designing and implementation**

- Existing codes can be helpful for us as references for us to more quickly build Hub liquidity module.
