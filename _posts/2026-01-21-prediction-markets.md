---
layout: post
title: "A Relative Value Trade in Cash-or-Nothing Options on Prediction Markets"
math: true
---
## A quick primer on prediction markets

Prediction markets, such as Polymarket and Kalshi, have received [a lot](https://www.bloomberg.com/opinion/newsletters/2025-11-19/fine-trade-labubu-futures) of attention recently. One interesting product that is traded on Polymarket is a binary option on Bitcoin, in which you can bet if it will go up or down in some time period:

![15 minute option orderbook](/assets/images/15min-orderbook.png)

Above is the shortest and most popular 15-minute option. There is a bug in the volume counter shown at the top of the orderbook – volume always goes back to $1.0k when the page is refreshed, and then ticks up. The bitcoin market easily generates over $100k of volume in a 15-minute market. In case you don't know how these markets work, you can buy a share of "Yes", in this case for 92 cents, which will resolve for $1 if Bitcoin finishes above \$96420.35, and $0 otherwise. Your counterparty has bought a "No" share for 8 cents.

These markets are fairly competitive, with the spread almost always at one cent. This isn't very competitive as a percentage, but it's the minimum tick size for prices between 1 and 99 cents. Market Makers have made outsized profits on these markets, with [Account88888](https://polymarket.com/@Account88888)[^account] previously known as "JaneStreetIndia" making $645k USD in just over a month trading only 15-minute markets, and would stand to make more with the introduction of maker rebates, although they seem to have stopped trading.

Unfortunately, neither Polymarket nor Kalshi lets Australians trade. The next best option is the much less popular Limitless. For context, Polymarket's notional volume for the week beginning the 5th of January was $1.6bn, about 178 times greater than Limitless' $9mn.

## More on Limitless

The shortest binary options traded on Limitless are hourly markets. While they have the same expiries as Polymarket, they source data differently. Polymarket gets its data from Binance, whereas Limitless gets its data from Pyth, which aggregates prices from a range of sources. The difference between Binance and Pyth prices tends to be constant, and is often mean reverting, and so can mostly, but not always be ignored. The mean reversion comes from the fact that Pyth is just a bit slower at reporting prices – if Binance ticks up, it will take a few hundred milliseconds for Pyth to receive this data, and then factor it in.

The main difference is the mechanism in which these products are traded. All markets on Polymarket (that is, BTC, ETH, SOL, XRP) are traded through a central limit order book (CLOB).

Only Bitcoin and Solana are offered through a CLOB on Limitless. There seems to be only one market maker[^onemm]. Their strategy is fairly rudimentary – they set their midpoint to be that of Polymarket's, and quote 3 cents wide on each side, for a spread of 6 cents. Interestingly, a spread of 6 cents is the maximum allowed to qualify for rewards from the exchange. Previously, they always quoted 100 contracts of volume on each side. This isn't the case anymore – perhaps they wish to obfuscate their strategy, or someone is queueing up behind the original MM to farm rewards, knowing their quotes are very unlikely to get filled.

While this seems an attractive market to make, the infrastructure offered by Limitless is quite poor. It takes seconds to confirm if one of our orders has been executed, and it is not possible to send a maker only order. If we happen to be a taker, then we risk being slugged with a fee up to 3%. While the fee curve can be approximated, the exact equation isn't available in documentation, nor upon request, but perhaps can be derived from on-chain data.

## AMMs

Anyway, the mechanism on which Ethereum and XRP are traded is more interesting. Limitless uses a constant product automated market maker (AMM). This means that token reserves follow the equation x * y = k, where x and y are the reserves for yes and no contracts respectively, and k is some constant. Also, since the prices of yes and no contracts must add up to 1, we have $$p_x + p_y = 1$$.

On Limitless, k = 2500. At the start, there are 50 yes and 50 no shares. Some liquidity provider has put up $50, and in return, will receive a cut of trading fees on this market. Currently, fees are 0.1%, as most of the cost incurred will be in slippage. Suppose we wish to buy $5 of yes tokens, ignoring fees. The AMM will take our $5, and mint 5 yes and 5 no tokens. We only want exposure to the yes tokens, so we'll swap the remaining no tokens for more yes tokens.

As x*y = 2500, and we know that y is 50 + 5 = 55, x must be about 45.45. So, we receive 50 – 45.45 = 4.55 further yes tokens, bringing our share up to 9.55 tokens, having paid an average of 52.3 cents per token.

Even trading a paltry $5 has resulted in 4.6% slippage. The disadvantage of trading this is obvious – we will need a considerable amount of edge to perform a trade of reasonable size. However, any time someone trades with reasonable size, we should get an opportunity to knock the price back to its fair value.

An interesting decision made by Limitless is that some accounts have a minimum trade size on these markets. If you use a custodial account[^custodial], in which Limitless manages your wallet (and pays gas fees), they impose a $3 minimum buy.

Although we trade by buying yes and no tokens (and selling them if we have all our capital tied up), I'll quote everything hereafter in terms of yes quotes for simplicity. So, instead of 'shorting yes', we are really buying no. 

## The Setup

Here's an example of a few interesting trades:

![Example of a good trade graph](/assets/images/good-trade-graph.png)

There's a bit going on here. First, the blue line. This is the midpoint on the corresponding Polymarket market. The yellow line is my attempt at normalizing the Polymarket midpoint[^fairprice]. We back out the implied volatility under Black-Scholes from Polymarket, and plug it to Limitless. This isn't optimal for a couple of reasons:

1. BS isn't great at pricing these markets.
2. Volatility isn't constant across strikes, and
3. As mentioned above, the distance between Pyth and Binance often reverts – you can see this when the yellow line went up to 93 cents, and then crashed back down.

But, it's better than nothing. Also, it's just easier than constantly looking at another graph of the underlying's prices and trying to eyeball the magnitude of a move. The orange line is the current price of a yes token on Limitless, and the bold green and red lines are the average price we would get if we were to buy or sell $1 worth of yes[^slippage]. (If we did not have yes to sell, then we can buy no tokens).

The faint green and red lines are the average price if we were to buy or sell $3. You might notice that the marginal slippage is much higher from selling than buying. The opposite would be true if we were trading at, say, 16 cents. Also, dumb-money on Limitless is surprisingly risk-averse – overwhelmingly buying the most likely outcome, in this case yes.

## Fade the short?

This is a little annoying for us – as we will likely be net short when Yes is by far the most likely outcome. Of course, we will receive a rich payout if it resolves No, but we may go for many hours making losses. There are some more considerations to be made:

1. If there is a while to expiry, the expected displacement will be reasonably high, we will be able to do some further trades, and so the distribution of our payoff will not be: "lose most of the time, win really big a little of the time", which may be more convenient. We should be happy to go short.
2. If we are close to expiry, and the outcome is highly probable, we are almost certain that a large volume of buy orders will come in. Perhaps we may wish to get in front of them. We should consider fading the short opportunity.

I actually think it's worth shorting, even with a low TTE. It is quite inexpensive to get out of a short position (that is, there is not a lot of slippage to go long) when the price is high. If the time to expiry was short here (which, it wasn't), I think the trader represented by the blue down arrow has made the best trade. The trader represented by the pink arrow can be reliably expected (as in, this particular trader is running their strategy in these markets 24/7) to go long or short 10 lots to push price back to its fair value, when it is approximately 4 cents over or undervalued. Interestingly, it seems to transact the same amount of shares, regardless of the price, resulting in quite a high amount of slippage when price is high. They are also not very fast, and you can quite reliably predict when they are going to trade. So, the trader in blue can trade short say, 4 lots, wait for pink to sell 10 lots, and then buy to cover for what is an almost riskless profit. Pink has made a good trade too - they have sold for an average price above that of the fair value, but they must endure some variance as a cost of being slow, and indiscriminate with their sizing.



Here is the tape for the last few minutes of an Ethereum market:

![Market Tape](/assets/images/tape.png)

Here, delta is a wallet's overall exposure to Yes. Wallets with a colour, in this case only 9EC7, are those that have traded multiple times in this market. There are a couple of interesting things here:

1. 7 orders coming in at 7:57:51 of similar size. These wallets have all done a bit over $200 in volume total, which is notably the minimum amount to be eligible for an airdrop.
2. There exist people buying a contract, which has a maximum value of $1, at a price of 100.2 cents.
3. The people who have bought the contract after expiry, at what is seemingly an unfavourable price each have over $100,000 in volume traded!

I'm not sure if this is somehow a profitable trade (I doubt it?), or these traders are farming volume, with the goal of receiving an [airdrop](https://x.com/trylimitless/status/2010773556192239652). One can generate a lot of volume with just $100. Trading in the few seconds after this market is resolved means that it is impossible to lose, and so we only pay fees. Assuming that we pay 10 bps of fees, and a 20bps 'premium' (that is, we pay 100.2 cents for a contract worth 100, as well as 0.1 cents in fees), we can generate:

$$
100(0.997) + 100(0.997)^2 + \dots = \frac{100(0.997)}{1 - 0.997} \approx \$33,233
$$

worth of volume. Of course, we only have a finite amount of time to trade, and might not be able to commit our desired amount of capital to each market. This isn't a particularly exciting trade, in my opinion. Unfortunately even selling $1 of this contract would result in an average execution price of 99.5 cents. Hence, arbitragers are very hesitant to get short at around 95 cents, or long at 5 cents, especially with a low time to expiry, as there is little chance of ending up flat at resolution.

## Back to the trade

Here, the time to expiry was actually quite high, about 30 minutes, and so we should be pretty happy to go short. I mentioned above that with a short TTE, the trader represented by the blue down arrow made the best trade, and I still agree with this with a longer TTE, although they perhaps could have traded with more size.

These types of opportunities tend to present themselves quite a lot. This is a two minute window showing edge in a $1 trade:

![Edge Graph](/assets/images/edge.png)

Currently, I automatically enter positions when there's more than 3 cents of edge, and these positions are automatically hedged whenever there is more than 1 cent of edge. This approach has been pretty successful so far as shown in the below equity curve:

![Equity Curve](/assets/images/equity_curve.png)

This only tracks my USDC at the end of every market, and so doesn't capture all of the strategy's variance, not that there is much.

---

[^account]: I would really, really love to know who is behind this account!
    
[^slippage]: You might notice that the orange line lags the green and red lines. We query the slippage from the blockchain, whereas we get the actual price provided by Limitless. They are both somewhat delayed – you can see the up and down arrows representing a trade with long (buy yes, sell no) or short (sell yes, buy no) intentions. These are only provided a few seconds after the fact, unfortunately.
    
[^fairprice]: It's worth noting that I don't actually use a model of my own to work out a fair price. I have tried modelling this - I think Polymarket is a little bit more accurate than I am, especially at tail probabilities. Also, the price on Limitless does seem to mean revert to Polymarket's, so the ROI of building one's own model is very low. (Perhaps the converse is true, as well.)
    
[^onemm]: I am fairly confident there is only one, as sometimes they cancel their quotes, and take about a second to post new ones (there is no amend order), while the spread blows up to about 20 cents.
    
[^custodial]: More specifically, you have signed in with Google, Twitter, or Discord, instead of a Crypto wallet. You then have to use Limitless to deposit and withdraw, rather than approving Limitless transactions with your wallet. I suspect that people who entrust Limitless with their capital are not particularly price sensitive.
    