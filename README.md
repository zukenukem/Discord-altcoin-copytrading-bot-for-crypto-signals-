This is the code I have so far that Deepseek cooked up.
I have zero coding knowledge so any help is much appreciated!

import discord
import requests
import time
import hmac
import hashlib
import logging
from decouple import config
from discord import Intents

# Configure logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

# Load environment variables
DISCORD_TOKEN = config('DISCORD_TOKEN')
MEXC_API_KEY = config('MEXC_API_KEY')
MEXC_API_SECRET = config('MEXC_API_SECRET')
DRY_RUN = config('DRY_RUN', default=True, cast=bool)
MAX_LEVERAGE = config('MAX_LEVERAGE', default=5, cast=int)
RISK_PERCENTAGE = config('RISK_PERCENTAGE', default=1.0, cast=float)

# MEXC API endpoints
MEXC_REST_URL = "https://api.mexc.com/api/v3"
MEXC_WS_URL = "wss://wbs.mexc.com/ws"

# Initialize Discord client
intents = Intents.default()
intents.message_content = True
discord_client = discord.Client(intents=intents)

# MEXC API authentication
def generate_mexc_signature(secret, params):
    query_string = '&'.join([f"{k}={v}" for k, v in params.items()])
    return hmac.new(secret.encode(), query_string.encode(), hashlib.sha256).hexdigest()

# Fetch MEXC account balance
def get_mexc_balance():
    try:
        timestamp = int(time.time() * 1000)
        params = {'timestamp': timestamp}
        params['signature'] = generate_mexc_signature(MEXC_API_SECRET, params)
        headers = {'X-MEXC-APIKEY': MEXC_API_KEY}
        response = requests.get(f"{MEXC_REST_URL}/account", headers=headers, params=params)
        response.raise_for_status()
        return float(response.json()['balances']['USDT']['free'])
    except Exception as e:
        logging.error(f"Failed to fetch balance: {e}")
        return 0.0

# Execute trade on MEXC
def execute_mexc_trade(symbol, side, quantity, leverage):
    try:
        if DRY_RUN:
            logging.info(f"Dry Run: Would execute {side} {quantity} {symbol} at {leverage}x leverage")
            return

        # Set leverage (MEXC futures example)
        timestamp = int(time.time() * 1000)
        leverage_params = {'symbol': symbol, 'leverage': leverage, 'timestamp': timestamp}
        leverage_params['signature'] = generate_mexc_signature(MEXC_API_SECRET, leverage_params)
        headers = {'X-MEXC-APIKEY': MEXC_API_KEY}
        leverage_response = requests.post(
            f"{MEXC_REST_URL}/leverage", headers=headers, params=leverage_params
        )
        leverage_response.raise_for_status()

        # Place order
        order_params = {
            'symbol': symbol,
            'side': 'BUY' if side == 'long' else 'SELL',
            'type': 'MARKET',
            'quantity': quantity,
            'timestamp': timestamp
        }
        order_params['signature'] = generate_mexc_signature(MEXC_API_SECRET, order_params)
        order_response = requests.post(
            f"{MEXC_REST_URL}/order", headers=headers, params=order_params
        )
        order_response.raise_for_status()
        logging.info(f"Order executed: {order_response.json()}")

    except Exception as e:
        logging.error(f"Trade failed: {e}")

# Discord message handler
@discord_client.event
async def on_message(message):
    if message.author == discord_client.user:
        return

    if message.channel.name == 'trading-signals':
        content = message.content.lower()
        if 'long' in content or 'short' in content:
            try:
                # Parse message (e.g., "BTCUSDT long 10x 1.0")
                parts = content.split()
                symbol = parts[0].upper()
                side = 'long' if 'long' in content else 'short'
                leverage = min(int(parts[2].replace('x', '')), MAX_LEVERAGE)
                risk_amount = float(parts[3])

                # Fetch balance and calculate quantity
                balance = get_mexc_balance()
                if balance <= 0:
                    logging.error("Insufficient balance")
                    return
                price = float(requests.get(f"{MEXC_REST_URL}/ticker/price?symbol={symbol}").json()['price'])
                quantity = min(risk_amount, (balance * RISK_PERCENTAGE / 100) / price)

                # Execute trade
                execute_mexc_trade(symbol, side, quantity, leverage)

            except Exception as e:
                logging.error(f"Signal parsing failed: {e}")

# Run bot
if __name__ == '__main__':
    discord_client.run(DISCORD_TOKEN)
