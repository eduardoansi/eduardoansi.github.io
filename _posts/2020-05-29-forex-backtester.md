---
layout: post
title:  "Creating a basic backtesting algorithm for trading"
date:   2020-05-29 15:00:00
categories: [python,pandas,scipy,forex]
comments: true
excerpt: "Building a backtesting system for financial markets can be a challenge, but the whole process for sure is very rewarding. I created a simple algorithm with an optimization function to test the performance of a given set of trades."
---
DISCLAIMER: Foreign Exchange trading is a high-risk activity. This post or any content on this blog should not be taken as financial or trading advice of any kind.
{: .notice}

When I started my trading career I looked for a good backtesting software that could give me the flexibility of trying not only different entry/exit methods, but also to test various money management techniques.

While there are several options out there, free and paid, they all have their restrictions, from being expensive to not having enough historical data to work with. I then decided to create an algorithm with Python to do the work, and it gave me some valuable experience, and I will show here some basic ideas to do so.

Before having any experience with Python I discovered Pandas. For my first algorithm I created a nice code using mostly what I could find in Pandas documentation. The result is that I used a lot of *for* loops, iterating with iterrows, one of the slowest methods possible. I gave up when a test with a preliminar dataset was taking too much time to run, considering that I was still going to try different variations of parameters on much bigger datasets. Then I decided to learn Python properly.

### What a backtesting algorithm does

Normal financial data is organized by date/time, price (usually be split into open, high, low and close) and volume. Then, the trader will create the entry conditions, a precise point in time and price where the strategy will buy or sell an asset.

When this entry condition is triggered, the algorithm now has to check how that specific trade will perform, given a set of parameters, like maximum loss, maximum profit, trailing stop loss, etc. In a simple example, it would look like this, with a buy entry when price reaches 107.00:

```
Date	  Price	     Action
27-april  105.00	
28-april  104.00	
29-april  104.50	
30-april  106.00	
01-may    105.50	
04-may    106.50	
05-may    107.00     Buy entry
06-may    107.50	
```

The challenge now is to analyze the trade within its parameters, the stop loss, take profit, its own entry point and any other indicators the system requires.

With a spreadsheet mindset, whenever you have an entry you could replicate the entry price to the next row, and so on until the trade is over. But if you have a maximum loss of 2 units (price reaching 105.00), you have to also check for each row if that loss was triggered, and when it happens, close the trade with the loss.

```
Date	Price	Action
05-may	107.00	Buy entry
06-may	107.50	
07-may	107.00	
08-may	106.50	
11-may	106.00	
12-may	105.50	
13-may	105.00	Exit loss
14-may	104.50	
```

The algorithm then could simply identify the entry points and iterate over the dataset, starting from that point in time, checking if difference between the current price and the entry price does not exceed the maximum profit or maximum loss; when it does, the trade is closed and it can finally be closed and registered in the results.

The code would look more or less like this, for a single trade:

```py
for i in big_time_range:
    entry_price = price_in_05_may
    difference = actual_price - entry_price
    if difference <= -2.00: # -2.00 being the max loss
        result = "loss"
        print(result)
        break
    elif difference >= 5.00: # +5.00 being the max profit
        result = "win"
        print(result)
        break
```

This will work fine, but it will be heavy on the processing side. This loop will run thousands of times (depending on the number of trades) for each asset and for an indefinite number of rows on each trade. Not to mention that probably all the other indicators and trading rules are also being ran at the same time for each row, creating loops inside loops. On top of all that, it doesn't make sense to create such a sophisticated algorithm for backtesting if we are not going to optimize some of the parameters. If you try the function minimize from SciPy with a brute-force method, it will take forever. This is one of the disadvantages of Python, its running time, but some tricks can make it faster.

We can use some vectorized operations to do simple and more complex calculations, while the usage of methods like *loc*, *iloc*, *shift* and *rolling* also save a lot of processing time.

With vectorized operations you can trim down the code and keep it more readable as well.

### Experimenting

Here I will show my commented code on a simple test I performed to showcase these ideas, considering that the entry signals were already given:

```py
# Imports
import pandas as pd
import numpy as np

# Read pair data (using a 1-minute timeframe dataset)
gbpusd = pd.read_csv('GBPUSD1.csv')
​
# Clean pair data
gbpusd.columns = ['Date','Time','Open','High','Low','Close','Volume']
gbpusd = gbpusd.drop(['Open','Close','Volume'],axis=1)
gbpusd['Datetime'] = gbpusd['Date'] + ' ' + gbpusd['Time']
gbpusd = gbpusd.drop(['Date','Time'], axis=1)
​
# Calculate ATR (ATR is an indicator of volatility per period)
# using the .rolling method
gbpusd['Atr255'] = (gbpusd['High'] - gbpusd['Low']).rolling(window=255).mean()
gbpusd['Atr17'] = (gbpusd['High'] - gbpusd['Low']).rolling(window=17).mean()
```

Now we have the price data in a proper format with the ATR calculated over it.

Next we get the list of trades taken.​

```py
# Read trades
gutrades = pd.read_csv('Object List---GBPUSD,M1.csv')
​
# Clean trades
# The trade list was given manually, so this algo doesn't try to get the entry rules.
# The method of getting this set of trades was manually placing them on Meta Trader
gutrades = gutrades[gutrades['Type'] == 'Arrow']
gutrades = gutrades[['Description','Time1','Price1','Arrow']]
gutrades = gutrades.drop([0,1,2,3], axis=0)
gutrades.columns= ['Description','Datetime','Price','Direction']
gutrades['Direction'].replace(241,'Long',inplace=True)
gutrades['Direction'].replace(242,'Short',inplace=True)
​
# Now I have a list of entries with date, time, price and direction.
​
# Create empty dataframe and minutes array for trade analysis.
# I had a time limit for each trade of 300 minutes for this particular strategy.
# This array was meant to serve as a range for each calculation.
gucharts = pd.DataFrame()
minutesarray = list(range(1,301,1))
​
# Create chart details for each trade taken
for i in range(gutrades.shape[0]):

    # Get index or identifier of trade
    tradeinfo = gbpusd[gbpusd['Datetime'] == 
                       gutrades.iloc[i]['Datetime']].index.values[0]
​
    # Append chart info to provisory dataframe
    thistrade = gbpusd.iloc[tradeinfo:(tradeinfo+300)][['High','Low','Datetime']]
    thistrade['Description'] = gutrades.iloc[i]['Description']
    thistrade['Price'] = gutrades.iloc[i]['Price']
    thistrade['Direction'] = gutrades.iloc[i]['Direction']
    thistrade['Trade code'] = 'GBPUSD ' + gutrades.iloc[i]['Datetime']
    thistrade['Atr255'] = gbpusd.iloc[tradeinfo]['Atr255']
    thistrade['Atr17'] = gbpusd.iloc[tradeinfo]['Atr17']
    thistrade['Minutes'] = minutesarray
​
    # Check if trade was buy (long) or sell (short) and calculate point (pips) differential
    if gutrades.iloc[i]['Direction'] == 'Long':
        thistrade['Pips high'] = thistrade['High'] - thistrade['Price']
        thistrade['Pips low'] = thistrade['Low'] - thistrade['Price']
        thistrade['Pips high Atr'] = thistrade['Pips high'] / thistrade['Atr255']
        thistrade['Pips low Atr'] = thistrade['Pips low'] / thistrade['Atr255']
    else:
        thistrade['Pips high'] = (thistrade['Price'] - thistrade['Low']) * (-1)
        thistrade['Pips low'] = (thistrade['Price'] - thistrade['High']) * (-1)
        thistrade['Pips high Atr'] = thistrade['Pips high'] / thistrade['Atr255']
        thistrade['Pips low Atr'] = thistrade['Pips low'] / thistrade['Atr255']
​
    # Append trade info to analysis dataframe
    gucharts = gucharts.append(thistrade)
​
# Delete unnecessary tables and variables to clear memory
del thistrade
del tradeinfo
del i
del gbpusd
del gutrades
gucharts = gucharts.drop(['High','Low'], axis=1)

# Get list of unique trades
uniquetrades = gucharts['Trade code'].unique()
```
​Now we have the trade list in a database with the ATR information (because our stop losses and take profits are measured in terms of ATR). We also have a list of the trades taken with a code that can be helpful later for some kind of visualization, if needed.
​
Following, we define a function to analyze the performance of all the trades separated and combined.
​
```py
# Define trading function
def tradingpips (parameters):
​
    # Create results dataframe
    results = pd.DataFrame()
​
    # Assign parameters of function to variables
    stoploss = parameters[0]
    takeprofit = parameters[1]
    belevel = parameters[2]
​
    # Analyze trade
    for i in range(uniquetrades.shape[0]):
​
        # Create dataframe to receive actual trade info
        actualtrade = gucharts[gucharts['Trade code'] == uniquetrades[i]].copy()
        actualtrade.loc[:,'SL'] = stoploss 
        actualtrade.loc[:,'TP'] = takeprofit
​
        # Move stop loss to break even
        firstbe = actualtrade[actualtrade['Pips high Atr'] >= belevel].head(1)['Minutes'].copy()
        actualtrade.loc[(actualtrade['Minutes'] >= firstbe.sum()) & (firstbe.sum() > 0),'SL'] = 0
​
        # Calculate take profit and stop loss being hit
        actualtrade.loc[:,'Hit TP'] = actualtrade.loc[actualtrade['Pips high Atr'] >= actualtrade['TP'],'TP']
        actualtrade.loc[:,'Hit SL'] = actualtrade.loc[actualtrade['Pips low Atr'] <= actualtrade['SL'],'SL']
​
        # # Determine the first one to be hit and delete rest of trade info
        actualtrade.loc[:,'Result'] = actualtrade[['Hit TP','Hit SL']].min(axis=1, skipna=True)
        getclosedminute = actualtrade.loc[actualtrade['Result'].notnull(),['Minutes']].head(1).copy()
        actualtrade.loc[:,:] = (actualtrade.loc[actualtrade['Minutes'] <= getclosedminute.iloc[0,0].sum()]).dropna(axis=0, how='all')
        actualtrade = actualtrade.dropna(axis=0, how='all')
​
        # Append results to dataframe
        results = results.append(actualtrade)
​
    # Calculate sum of pips
    return (results['Result'].sum()) * (-1)
```
This function receives three values as input and outputs one value. The inputs are stop loss, take profit and move to breakeven, all of them in terms of ATR's. Stop loss and take profit are more straight forward; the move to breakeven parameter is the level of profit which I would move my stop loss to break even.

We also get the output as a negative value for reasons that we will see soon.

It can be curious to notice that, even though this is a time-series, I don't really use the Datetime column for calculating the duration of the trade. There are a few reasons for that:

1. I am 100% sure my data is saved minute by minute - there are no gaps on the periods I am studying;
2. Therefore, instead of calculating 30 minutes with Datetime, I can just get 30 rows and it will be the same;
3. It was unnecessary to work with date and time.

Now we can run the algorithm with a simple line and see the results. It is important to keep in mind that I am using my profit and loss levels as a function of the ATR. So, if ATR is 5 pips, and there is a 10 pip movement, the algorithm will read that movement as 2 ATR's.

```py
print(tradingpips([-1,2,2])) # The parameters are a list: [SL, TP, MovetoBE]
```

```
-4.0
```
This means that with a stop loss of -1 ATR, take profit of 2 ATR's and moving the stop loss to entry level at +2 ATR's, we get a final profit equivalent of +4.0 ATR's (remember the signal is inverted).

### A word about optimization

Optimizing parameters within the context of a Forex strategy, or any strategy related to financial markets has to be done very cautiously. First of all, the trader can think that an optimized function for trades taken between 2010 and 2015 will give the best parameters for the subsequent months, which can be very much not the case. The strategy should be drawn with a good theory behind and incorporate market behavior to determine its entries, exits and risk management techniques.

Also, it will fall short from the goal of machine learning good practices, which is to train and test on different parts of the data.

That said, we will still optimize this function with SciPy to show the capabilities of this particular algorithm, using the *minimize* function and the method of Sequential Least Squares Programming.

As a reminder, we had the output of our trading function in a negative value so that we can minimize it (there is not a maximize function).

```py
from scipy.optimize import minimize

# For this method we have to set the bounds of testing...
slrange = (-6,-1) # from -6 ATR's to -1 ATR
tprange = (1,6)   # from +1 ATR to +6 ATR's
mvrange = (1,6)   # from +1 ATR to +6 ATR's

# ... as well as define initial values for the optimization
# This will help the function to start from a better and more reasonable point
x0=[-4,5,5]

# and then we can run the function and print the result
res = minimize(fun=tradingpips, x0=x0, method='SLSQP', bounds=[slrange,tprange,mvrange])
print(res)
```

```
fun: -23.611327498037312
     jac: array([-1., -5.,  0.])
 message: 'Optimization terminated successfully.'
    nfev: 229
     nit: 27
    njev: 25
  status: 0
 success: True
       x: array([-3.40022842,  5.40231118,  5.        ])
```

And there we have it. If we use -3.4 for stop loss, 5.4 for take profit and move stop loss to entry when we have a profit of 5.0, we will have our best result for this sample. The outcome is a profit of +23.6 ATR's (remember the negative value).

Keep in mind that this sample is very, very small, quite insignificant. Also we set our bounds and we had an initial guess to help the process of optimization. For this particular sample, it is quite likely that the third parameter was completely useless, but maybe it would not be the case for a proper dataset.

### Conclusion

So, while the results are not to be considered, the algorithm seems to do its job. It has some redundant variables sometimes, but for me it made things clearer while coding and then when reading it again. There is plenty room for improvements, both in structure and efficiency, but it shows that taking advantage of the vectorized aspect of NumPy and Pandas can help when performing tasks that have some kind of similarities with this one. Trading costs should also be taken into consideration.

Ther optimization function can help one to have an idea if the best result of a given strategy is compatible with the theory behind it, and maybe even give an indication of where to put more work into, if it should be discarded completely (imagine that no set of parameters whatsoever yields a profitable result) or if the results are more or less the expected ones. Most of all, it seems that this practice should be a confirmation or a way to eliminate some trading ideas, which is very helpful in terms of not wasting so much time.