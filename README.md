# share_ordering_demo

This repository holds tutorials that demonstrate ways to buy and sell stocks in cloudquant backtesting, and different ways to determine the volumes of shares that are purchased.

There is no single "right way" to do any of these. You will have to think carefully about your algorithm, how it determines when to buy and sell, how large a trade you want to implement, and how quickly you need your orders filled.



Let's start with the aspect of trade volume. One common way to determine volume is simply to purchase a fixed number of shares:
num_shares=100
This way is great for its simplicity, but it's really not recommended because high-priced stocks will have a much more dramatic impact on your algorithms gains and losses than lower priced stocks.


Another way is to base the volume purchased on some fraction of the volume of ask price in the previous minute.
For example:
num_shares=int(bar_askvol[0]*.1)
This is useful if you want to order a large number of shares quickly without ordering so many that you actually change the price of the shares. 
Be careful, though. The dollar value of the trade can be wildly different with this strategy, anywhere from less than $1000 to over $500000. Again, your algorithm will be dominated by high-price and high-volume shares.


Finally, the commonly recommended way, is with a dollar amount:
num_shares=np.round(purchase_amount/md[self.symbol].L1.last)
where "purchase_amount" is the dollar value of a stock you want to purchase. Usually between $10000 and $25000 is a good amount, and you won't disrupt anything on the Russell 2000 with those numbers.



OK, now, let's focus on different ways to initiate a trade.

To give an example, let's imagine a hypothetical stock XYZ, at time t0 with bid prices at 29.95 and ask prices at 30.05. 

One option is a simple market order:
order_id = order.algo_buy(self.symbol, algorithm="market", intent="init", order_quantity=num_shares)
This means that your order will fill at the lowest price someone is actively willing to sell it at. 
In real trading, you would never buy significant shares of stocks like this, because people will raise their ask price when they realize someone is buying on market.
In backtesting, however, it is essentially assuming you are buying at the ask price for that time, which is reasonable.
Your order will always fill immediately, and the only risk is that if the stock price shoots up, you will be paying whatever price the stock goes up to.
In our example of stock XYZ, you are buying at 30.05 at t0, though if your order is placed at t0, you may be purchasing at the ask price at time t1, which could be different from the ask at t0. Market orders will simply purchase at whatever that ask price is.

Another way is to initiate a limit order, based on the ask price:
order_id = order.algo_buy(self.symbol, algorithm="limit", price=md[self.symbol].L1.ask-.01, intent="init", order_quantity=num_shares)
or
order_id = order.algo_buy(self.symbol, algorithm="limit", price=md[self.symbol].L1.ask*.99, intent="init", order_quantity=num_shares)
These two algorithms place an order for a stock with a limit on the price. In the first case, we set a limit one cent below the ask, and in the second our limit is 99% of the ask price. These are likely to get filled, but not guaranteed, though they will get a better deal than a simple market order. The farther below ask you go, the better a deal you might get, but the higher a chance that you won't get filled. The lowest you can go is probably 95% ask or 5 cents less than the ask. 
If it's important that you get your order filled immediately, you will want to place a more aggressive limit, such as the ask price PLUS 5 cents or 105% the ask price. This is similar to a market order, but will keep a lid on how much you actually will pay for the shares.
This is a critical distinction in live trading, but less important in back-testing.
In our example of stock XYZ, if we, at t0, place a limit order one cent below the ask, we are essentially offering at t1 a price of 30.04. Cloudquants backtesting environment does its best to approximate whether someone would have been likely to meet our price limit or not. If the price of stock XYZ moved up at t1, there is a very high chance we would not have been filled. If, however, we set a limit of ask + 5c, we would have placed a limit at $30.10, and we would have likely been filled unless the stock shot up more than that.

Finally, we have a slightly more complex way of computing our trade price, using a "midpoint peg." 
order_id = order.algo_buy(self.symbol, algorithm=lime_midpoint_limit_buy, price=md[self.symbol].L1.ask*1.05, intent="init", order_quantity=num_shares)
order_id = order.algo_buy(self.symbol, algorithm=lime_midpoint_limit_buy, price=md[self.symbol].L1.ask+.05, intent="init", order_quantity=num_shares)

You will also, earlier in the code, need the lines:
lime_midpoint_limit_buy = "4e69745f-5410-446c-9f46-95ec77050aa5"
lime_midpoint_limit_sell = "23d56e4a-ca4e-47d0-bf60-7d07da2038b7"

This algorithm is only available in the elite cloudquant version, but you could approximate it in lite by using the mean of the ask and bid prices. This is similar to what the "lime" midpoint peg does, but the real version should include elements such as the volume of the shares to accurately estimate where the price would have been. 
If your trade doesn't need to urgently fill, the lime midpoint peg is a good way to go, however if your trade requires an immediate fill, this may give you unrealistic purchase prices, and make your algorithm seem better than it really is.
In our XYZ example, this essentially assumes we would always be purchasing shares of XYZ for $30.00 at t0, and then the average between bid and ask at t1, and so on.

Check the demo scripts for examples of implementation.
