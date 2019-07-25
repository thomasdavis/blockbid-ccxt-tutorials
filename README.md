# Blockbid CCXT Tutorials

[blockbid.io](https://platform.blockbid.io/trade/btctusd) - Australian crypto exchange, no KYC required

We thought it would be useful to show our users how to trade programatically at Blockbid. This repo will eventually contain a series of tutorials ranging from super beginner to advanced. Please use issues to suggest ideas about what you would like to see.

## Requirements

- Basic understanding of Javascript
- Basic understanding of trading
- NodeJS

## Getting Started

After you have cloned the repo.

1.  Generate an API key on your [settings page](https://platform.blockbid.io/settings/preferences)
2.  Edit `stragies/safeMargins.js` to add your new API key
3.  Execute the commands below

```
npm i
npm run start
```

## Safe High Margin Limit Order Tutorial

This tutorial will show you how to periodically set orders with high margins based off the average global market price.

It is actually quite easy to write trading bots, and this one in particular is really only around 30-40 lines of code.

It expects you to have two currencies e.g. BTC TUSD

It will then place one sell order and one buy order every minute with fluctuating prices on the BTCTUSD market.

We call it safe because the script is set to place orders 20% above and below market price, ensuring no one will likely match them, but if they do you should be happy.

The code is fully documentated and you should be able to read it even if you have no programming experience.

We love all feedback!

Successful output should look like:

```
Starting safe margin limit order strategy
Fetching current orders on BTC/TUSD
Cancelling current orders on BTC/TUSD
Finished cancelling 2 open orders
Fetching the global average price for BTC/TUSD
The global average price for BTC/TUSD is 10055.30417295
By our calculations we want to
Buy 0.0003019033084706648 BTC when the price is 8044.24333836
Sell 0.0011350974232612501 TUSD when the price is 12066.36500754
Successfully placed orders
```

Here is the contents of the script

```js
const ccxt = require("ccxt");
const axios = require("axios"); // Used for API requests

// WARNING: You will be placing orders on a real market place
const TRADE = false;
// WARNING: Change the above to true, if you want to actually execute orders

const API_KEY = "";
const SECRET = "";

// CCXT is a framework that was built to make trading across hundreds of exchanges
// Here we initialize CCXT to trade with Blockbid
let blockbidClient = new ccxt.blockbid({
  apiKey: API_KEY,
  secret: SECRET
});

// Here we define the configuration for a simple safe margin limit order strategy

const marketQuote = "BTC";
const marketBase = "TUSD";
const market = `${marketQuote}/${marketBase}`; // e.g BTC/TUSD"

const volumePercentage = 0.1; // What percentage of our available funds do we want to trade
const priceVariation = 0.2; // What price increase/decrease do we want to margin our orders with
const shiftSeconds = 60; // How often should we shift the price of our orders

// This is our main function, once executed, our orders will be placed
const strategy = async () => {
  console.log("\n\n Starting safe margin limit order strategy");

  // First let's cancel any open orders open on this market
  // To do that, we need to fetch a list of our open orders
  console.log("Fetching current orders on", market);
  const openOrders = await blockbidClient.fetchOpenOrders(market);
  // Now that we have the open orders, let's loop over them and cancel them
  console.log("Cancelling current orders on", market);
  openOrders.forEach(async order => {
    await blockbidClient.cancelOrder(order.id);
  });
  console.log("Finished cancelling", openOrders.length, "open orders");

  // The purpose of this script is to set safe/high-margin orders
  // We use cryptonator to get a current rough global price for the market
  console.log("Fetching the global average price for", market);
  const globalPriceUrl = `https://api.cryptonator.com/api/ticker/${marketQuote}-${marketBase}`;
  const globalPriceResponse = await axios(globalPriceUrl); // Axios makes network requests
  const globalPrice = globalPriceResponse.data.ticker.price * 1;
  console.log("The global average price for", market, "is", globalPrice);

  // Now that we have an average price, we want to calculate what our safe margins would be
  // e.g. If the average global price for BTCTUSD is $10000
  //      And our desired price variation is 20%
  //      We would want to sell at $10000 + ($10000 * 0.2) which would be ~$12000

  const sellPrice = globalPrice + globalPrice * priceVariation;
  const buyPrice = globalPrice - globalPrice * priceVariation;

  // Now we calculate what amount of volume we want to trade based off what we have

  // First, let's fetch our current balances on the exchange
  const balances = await blockbidClient.fetchBalance();

  // Now let's find the balance of our accounts for the market we wish to trade on
  const quoteBalance = balances.free[marketQuote]; // e.g. 0.01 BTC
  const baseBalance = balances.free[marketBase]; // e.g. 30 TUSD

  // To calculate how much we want to sell on this market we just
  // simply use our account balance of the quote currency and multiply
  // it by the percentage configured at the top of the file
  const sellVolume = quoteBalance * volumePercentage;

  // To buy on this market is slightly different. We can only buy as much as we
  // can afford by the balance of our base pair.
  // If the price of BTCTUSD is $10000 and we only have $1000 TUSD
  // then we can only afford to buy $1000 / $10000 = 0.1 BTC
  const buyVolume = (baseBalance * volumePercentage) / globalPrice;

  console.log("By our calculations we want to");
  console.log("Buy", buyVolume, marketQuote, "when the price is", buyPrice);
  console.log("Sell", sellVolume, marketBase, "when the price is", sellPrice);

  if (TRADE) {
    // Now we simply execute our limit orders using CCXT
    await blockbidClient.createLimitSellOrder(market, sellVolume, sellPrice);
    await blockbidClient.createLimitBuyOrder(market, buyVolume, buyPrice);
    console.log("Successfully placed orders");
  } else {
    console.log("TRADE EXECUTION IS DISABLED, SEE TOP OF FILE WARNING");
  }
};

const safeMargins = () => {
  if (API_KEY.length === 0) {
    console.log("You need to set your API key at the top of the file");
    return;
  }
  strategy(); // Call the strategy once
  // Now set an interval which calls the function every X seconds
  setInterval(strategy, shiftSeconds * 1000);
};
module.exports = safeMargins;
```
