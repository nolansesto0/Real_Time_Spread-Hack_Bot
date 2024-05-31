# Real_Time_Spread-Hack_Bot
Real time Bot that I use to make sure I am in line with my strategies outlooks and models.
-------------------------------
import requests
import pandas as pd
import time
import logging

# Setup logging
logging.basicConfig(filename='trading_strategy.log', level=logging.INFO, format='%(asctime)s %(message)s')

# Polygon API key
API_KEY = 'EcqFgP0PhDpUCcxY6EuDncsAC4_OvXeA'


# Function to print trade entry alert
def print_trade_entry_alert():
    alert_message = "***TRADE ENTRY ALERT*** Spread Hack: Type: Vertical Put, Ticker: QQQ, Short Leg Delta: <-0.09"
    print(alert_message)
    logging.info(alert_message)


# Function to fetch live data
def fetch_live_data(ticker="QQQ"):
    end_time = int(time.time()) * 1000
    start_time = end_time - (5 * 60 * 1000)  # Last 5 minutes
    url = f'https://api.polygon.io/v2/aggs/ticker/{ticker}/range/1/minute/{start_time}/{end_time}?apiKey={API_KEY}'
    response = requests.get(url)
    data = response.json()
    return data.get('results', [])


# Function to calculate MACD
def calculate_macd(df):
    df['EMA_12'] = df['close'].ewm(span=12, adjust=False).mean()
    df['EMA_26'] = df['close'].ewm(span=26, adjust=False).mean()
    df['MACD'] = df['EMA_12'] - df['EMA_26']
    df['Signal_Line'] = df['MACD'].ewm(span=9, adjust=False).mean()
    df['MACD_Histogram'] = df['MACD'] - df['Signal_Line']
    return df


# Function to calculate RSI
def calculate_rsi(df, period=14):
    delta = df['close'].diff(1)
    gain = (delta.where(delta > 0, 0)).rolling(window=period).mean()
    loss = (-delta.where(delta < 0, 0)).rolling(window=period).mean()
    rs = gain / loss
    rsi = 100 - (100 / (1 + rs))
    return rsi


# Main trading function
def run_strategy():
    df = pd.DataFrame()

    while True:
        try:
            # Fetch live data and append to DataFrame
            live_data = fetch_live_data()
            if not live_data:
                logging.info("No new data fetched.")
                continue

            live_df = pd.DataFrame(live_data)

            # Print the structure of the fetched data for verification
            print("Fetched data:")
            print(live_df.head())
            logging.info(f"Fetched data structure: {live_df.columns}")

            # Ensure columns are named correctly
            live_df.rename(columns={
                'c': 'close',
                'o': 'open',
                'h': 'high',
                'l': 'low',
                'v': 'volume',
                't': 'timestamp'
            }, inplace=True)

            live_df['timestamp'] = pd.to_datetime(live_df['timestamp'], unit='ms')
            live_df.set_index('timestamp', inplace=True)

            if df.empty:
                df = live_df
            else:
                df = pd.concat([df, live_df])
                df = df[~df.index.duplicated(keep='last')]

            # Ensure 'close' column exists
            if 'close' not in df.columns:
                raise KeyError("The 'close' column is not found in the data")

            # Apply the MACD calculation
            df = calculate_macd(df)

            # Apply the RSI calculation
            df['RSI'] = calculate_rsi(df)

            # Check for trade entry condition
            entry_range = 0.05  # Define the range within which MACD and Signal Line are considered close
            for i in range(1, len(df)):
                current_row = df.iloc[i]
                previous_rsi = df['RSI'].iloc[i - 1] if i > 0 else 50  # Get the previous RSI value

                if current_row['RSI'] < 50 and previous_rsi < 50:
                    if (current_row['MACD'] < 0 and current_row['Signal_Line'] < 0 and
                            abs(current_row['MACD'] - current_row['Signal_Line']) <= entry_range):
                        print_trade_entry_alert()
                        logging.info(f"Trade entry signal at {current_row.name}.")
                        break

            # Save the data to CSV periodically
            df.to_csv('qqq_data.csv', mode='a', header=not pd.io.common.file_exists('qqq_data.csv'))
            logging.info("Data saved to qqq_data.csv")

            # Sleep for a minute before fetching new data
            time.sleep(60)

        except Exception as e:
            logging.error(f"Error in running strategy: {e}")
            print(f"Error: {e}")


if __name__ == "__main__":
    run_strategy()
