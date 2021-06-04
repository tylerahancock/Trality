'''
v0.2 Not sell or buy twice the same level over/under the price
'''
import math

def initialize(state):
    state.number_offset_trades = 0

# 50 BUSD every time
buy_value = 50
# number of lines above and under the price
grid_size = 6
# separation between lines
grid_interval_percentage = 2

base_coin = "BTC"
quoted_coin = "USD"
symbol = base_coin +'-'+ quoted_coin


@schedule(interval="1h", symbol=symbol, window_size=200)
def handler(state, data):
    if data is None:
        return

    # get price, levels and orders
    current_price = data.close_last
    buy_levels, sell_levels = get_price_levels( current_price )
    open_buy_orders, open_sell_orders = get_open_orders()

    # calculate levels where we need to buy
    levels_to_buy, closest_missing_buy_levels = get_pending_levels( buy_levels, open_buy_orders )

    # if we have some missing levels below the current price is because they have been bought
    # try to sell them again immediately if current price is over their expected sell price
    value_to_sell_immediately = get_value_to_sell( current_price, closest_missing_buy_levels )
    if value_to_sell_immediately > buy_value :
        print("Selling {} {} immediately at {}".format(value_to_sell_immediately, quoted_coin, current_price))
        order_market_value( symbol=data.symbol, value=-value_to_sell_immediately )
    
    # place any missing buy order
    for level in levels_to_buy :
        try_place_buy_level_order( level )

    # calculate levels where we need to sell
    levels_to_sell, closest_missing_sell_levels = get_pending_levels( sell_levels, open_sell_orders )

    # if we have some missing levels above the current price is because they have been sold
    # try to buy them again immediately if current price is under their expected buy price
    value_to_buy_immediately = get_value_to_buy( current_price, closest_missing_sell_levels )
    if value_to_buy_immediately > 0 :
        print("Buying {} {} immediately at {}".format(value_to_buy_immediately, quoted_coin, current_price))
        order_market_value( symbol=data.symbol, value=value_to_buy_immediately )

    # place any missing sell order
    for level in levels_to_sell :
        try_place_sell_level_order( level )

    # close any order that is out of the grid range to freed their balance
    buy_orders, sell_orders = get_open_orders()
    close_far_orders( buy_orders, sell_orders )

    # that's all, print some logs to check what's happening
    buy_orders, sell_orders = get_open_orders()
    print_orders(buy_orders, sell_orders, current_price);


# get open orders that match grid levels
# any other order not related to the bot should be ignored
# !! still buggy and messing with other orders
def get_open_orders() :
    orders = query_open_orders()
    buy_orders = {}
    sell_orders = {}

    for order in orders :
        if order.limit_price is not None :
            level = get_level_id(order.limit_price)
            if order.side == OrderSide.Buy :
                if order.symbol == symbol :
                    buy_orders[ level ] = order.id
            elif order.symbol == symbol :
                sell_orders[ level ] = order.id

    return buy_orders, sell_orders

# get levels where we need to place new orders
def get_pending_levels( levels, open_orders ):
    orders_to_place = []
    closest_missing_levels = []
    first_orders = True
    for level in levels : 
        if get_level_id(level[0]) not in open_orders :
            orders_to_place.append( level )
            # closest missing levels to the price tell us
            # the orders that have been fulfilled in the last candle
            if first_orders :
                closest_missing_levels.append( level )
        else :
            first_orders = False
    
    # if the 2 lists are the same, we don't really miss any close level
    if len(orders_to_place) == len( closest_missing_levels ) :
        closest_missing_levels = []

    return orders_to_place, closest_missing_levels

def get_value_to_sell( price, buy_levels ) :
    value = 0
    for level in buy_levels :
        if level[1] < price :
            value += buy_value * price / level[0]

    if value == 0 :
        return 0

    available = float(query_balance_free(base_coin))
    if available is None :
        return 0
    
    available_value = available * price * .995
    if available_value < buy_value :
        return 0

    # substract a bit of value to skip fee errors
    value = value * .995
    if available < value :
        return available
    return value

def get_value_to_buy( price, sell_levels ) :
    value = 0
    for level in sell_levels :
        if level[1] > price :
            value += buy_value
    
    if value == 0 :
        return 0
    
    available = float(query_balance_free(quoted_coin)) * .995

    if available < buy_value :
        return 0

    # substract a bit of value to skip fee errors
    value = value * .995
    if available < value :
        return available
    return value

# this is buggy as query_balance_free might return
# more free balance than it should because it's not counting
# orders we are being placed during the current handle run.
# fortunatelly, if there is not enough balance, order placing fails gently
# that it's the same than not placing the order
def try_place_buy_level_order( level ) :
    available = float(query_balance_free(quoted_coin))

    # we can only buy if we have enough money
    if available > buy_value :
        order_limit_value(symbol=symbol, value=buy_value, limit_price=level[0])


def try_place_sell_level_order( level ) :
    available = float(query_balance_free(base_coin))

    available_value = available * level[0]
    to_sell = buy_value * level[0] / level[1]
    # we can only sell if we have enough balance
    if available_value > to_sell :
        order_limit_value(symbol=symbol, value=-to_sell, limit_price=level[0])

# close orders that are out of the grid
def close_far_orders( buy_orders, sell_orders ) :
    if len(buy_orders) > grid_size :
        buy_far = get_far_orders( buy_orders )
        cancel_orders( buy_far )
    if len(sell_orders) > grid_size :
        sell_far = get_far_orders( sell_orders, False )
        cancel_orders( sell_far )

def get_far_orders( orders, smallest=True ) :
    keys = list(orders.keys())
    keys.sort(reverse=smallest)
    far_keys = keys[grid_size:]
    far_ids = []
    for key in far_keys :
        far_ids.append(orders[key])
    
    print("Cleaning {} orders".format(len(far_ids)))
    return far_ids

def cancel_orders( order_ids ) :
    for id in order_ids :
        cancel_order(id)

def print_orders( buy_orders, sell_orders, current_price ) :
    buy = list(buy_orders.keys())
    sell = list(sell_orders.keys())

    buy.sort()
    sell.sort()

    print("levels: {} - {} - {}".format( buy, get_level_id(current_price), sell ));

def trim_levels_by_placed_orders( buy_levels, buy_orders, sell_levels, sell_orders ) :
    trimmed_buys = buy_levels.copy()
    trimmed_sells = sell_levels.copy()

    # Don't trim anything if we have no orders
    if len(buy_orders) == 0 or len(sell_orders) == 0 :
        return trimmed_buys, trimmed_sells
    
    first_sell = get_level_id(sell_levels[0][0]) in sell_orders
    first_buy = get_level_id(buy_levels[0][0]) in buy_orders

    if not first_buy and first_sell :
        # we have just bought the first level in the last candle
        # but we couldn't sell it yet, don't buy it again
        print("Trimming first buy level")
        trimmed_buys.pop(0)

    if not first_sell and first_buy :
        # we have just sold the first level in the last candle
        # but we didn't buy it again yet, don't sell it again
        print("Trimming first sell level")
        trimmed_sells.pop(0)
    
    return trimmed_buys, trimmed_sells


# every level is 1.0098887 times bigger than the previous one
# so we can select increments close to 1,2,3...n percentage by picking 1 out of n levels
# base_levels[0] is always used to be sure that same levels are used for different 10^n order
# that would make some `grid_interval_percentage`s not to be respected in the limits of the list
base_levels = [1000000,1009889,1019875,1029960,1040145,1050431,1060818,1071309,1081902,1092601,1103405,1114317,1125336,1136464,1147702,1159051,1170513,1182088,1193777,1205582,1217504,1229543,1241702,1253981,1266381,1278904,1291550,1304322,1317220,1330246,1343400,1356685,1370100,1383649,1397331,1411149,1425104,1439196,1453428,1467800,1482315,1496973,1511776,1526726,1541823,1557070,1572467,1588017,1603720,1619579,1635594,1651768,1668102,1684598,1701256,1718079,1735069,1752226,1769554,1787052,1804724,1822570,1840593,1858794,1877175,1895738,1914484,1933416,1952535,1971843,1991342,2011034,2030920,2051004,2071285,2091768,2112453,2133342,2154438,2175743,2197258,2218986,2240929,2263089,2285468,2308068,2330892,2353941,2377219,2400726,2424466,2448441,2472653,2497104,2521797,2546735,2571919,2597352,2623036,2648974,2675169,2701623,2728339,2755319,2782565,2810081,2837869,2865932,2894272,2922893,2951796,2980986,3010464,3040234,3070297,3100659,3131320,3162285,3193556,3225136,3257028,3289236,3321762,3354610,3387783,3421284,3455116,3489282,3523787,3558633,3593823,3629361,3665251,3701495,3738098,3775063,3812394,3850093,3888166,3926615,3965444,4004657,4044258,4084250,4124638,4165425,4206616,4248214,4290223,4332648,4375492,4418760,4462456,4506584,4551148,4596153,4641603,4687502,4733856,4780667,4827942,4875684,4923898,4972589,5021762,5071420,5121570,5172216,5223362,5275014,5327178,5379856,5433056,5486782,5541039,5595833,5651168,5707051,5763486,5820480,5878037,5936163,5994864,6054145,6114013,6174472,6235530,6297191,6359462,6422349,6485858,6549995,6614765,6680177,6746235,6812947,6880318,6948355,7017065,7086455,7156531,7227300,7298768,7370944,7443833,7517442,7591780,7666853,7742668,7819233,7896555,7974642,8053501,8133139,8213566,8294787,8376812,8459648,8543302,8627785,8713102,8799263,8886277,8974150,9062893,9152513,9243020,9334421,9426727,9519945,9614084,9709155,9805166,9902127]

# generates `grid_size` levels above and below the given price
def get_price_levels( price ):
    # make the price fit in the base_levels array
    levelized_price = price
    factor = 1
    while levelized_price < base_levels[0] :
        levelized_price = levelized_price * 10
        factor = factor * 10

    # find its position in the base_levels
    price_index = find_price_level_index( levelized_price )

    buy_index = price_index
    buy_indices = []
    while len(buy_indices) < grid_size :
        # only use levels that are separated by `grid_interval_percentage`
        if buy_index % grid_interval_percentage == 0 :
            buy_indices.append( buy_index )
        buy_index = buy_index - 1

    sell_index = price_index + 1
    sell_indices = []
    while len(sell_indices) < grid_size :
        # only use levels that are separated by `grid_interval_percentage`
        if sell_index % grid_interval_percentage == 0 :
            sell_indices.append( sell_index )
        sell_index = sell_index + 1

    buy_levels = get_levels_by_indices( factor, buy_indices, True )
    sell_levels = get_levels_by_indices( factor, sell_indices, False )

    return buy_levels, sell_levels
        
    
def find_price_level_index( price ) :
    start = 0
    end = len( base_levels ) - 1

    if base_levels[end] <= price :
        return end

    while not (base_levels[start] <= price and base_levels[start+1]> price) :
        half = round( (end + start) / 2 )
        if base_levels[half] <= price :
            start = half
        else :
            end = half

    return start   

# levels are just lists of 2 prices
# in buy levels, the first price is where to buy and the second where we want to sell
# in sell levels, the first price is where to sell and the second where we bought
def get_levels_by_indices( factor, indices, ascendingPair ):
    levels = []
    for index in indices :
        secondary_index = index - grid_interval_percentage
        if ascendingPair :
            secondary_index = index + grid_interval_percentage

        levels.append([
            get_level_by_index( factor, index ),
            get_level_by_index( factor, secondary_index )
        ])
    return levels

# with the index of the level and the factor we can know
# what's the real price we want for the level
def get_level_by_index( initial_factor, index ) :
    factor = initial_factor
    parsed_index = index
    levels_len = len( base_levels )

    # negative index, we need to look at the end of the base_levels
    # and the factor is too small
    if index < 0 :
        factor = factor * 10
        parsed_index = index + levels_len
        if (parsed_index % grid_interval_percentage) != 0 :
            parsed_index = parsed_index - (parsed_index % grid_interval_percentage)
    
    # index overflows, we need to look at the start of the base_levels
    # and the factor is too big
    if index > levels_len - 1 :
        factor = factor / 10
        parsed_index = index - levels_len
        if (parsed_index % grid_interval_percentage) != 0 :
            parsed_index = parsed_index - (parsed_index % grid_interval_percentage)
    
    return base_levels[ parsed_index ] / factor

# The ids will only have 4 meaningful numbers, all the rest are trimmed/padded with 0s
# being too precise would fail to recognize levels in order prices
# 123456 -> 123400
# 12.3456 -> 12.34
# 0.00123456 -> 0.001234
def get_level_id( price ) :
    parts = str( price ).split('.')
    id = str(price)  
    if len(parts[0]) > 3 :
        id = parts[0][0:4] + pad( len(parts[0]) - 4 )
    elif len(parts) > 1 :
        if int(parts[0]) > 0 :
            id = parts[0]
            dec = parts[1][0:4-len(id)]
            id = id + "." + dec
        else :
            dec = ""
            i = 0
            while parts[1][i] == "0" :
                dec = dec + "0"
                i = i + 1
            
            id = "0." + dec + parts[1][i:i+4]

    return float(id)

def pad( length ):
    zeros = ""
    while length > 0 :
        zeros = zeros + "0"
        length = length - 1
    return zeros
