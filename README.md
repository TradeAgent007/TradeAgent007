# Dockerfile for Railway Deployment
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt requirements.txt
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "app.py"]

# requirements.txt
flask
alpaca-trade-api
requests

# Railway.toml
[service]
name = "alpaca-zapier-trade-bot"
start_command = "python app.py"

# app.py (Updated for Railway)
import json
import requests
from flask import Flask, request, jsonify
from alpaca.tradeapi import REST, TradeAPIError
import os

# Use environment variables for security
ALPACA_API_KEY = os.getenv('ALPACA_API_KEY')
ALPACA_API_SECRET = os.getenv('ALPACA_API_SECRET')
ALPACA_BASE_URL = 'https://paper-api.alpaca.markets'

alpaca = REST(ALPACA_API_KEY, ALPACA_API_SECRET, ALPACA_BASE_URL)
app = Flask(__name__)

@app.route('/trade', methods=['POST'])
def trade():
    data = request.json
    try:
        order = alpaca.submit_order(
            symbol=data['symbol'],
            qty=data['quantity'],
            side=data['side'],
            type='market',
            time_in_force='gtc'
        )
        response = {'status': 'success', 'order_id': order.id}
    except TradeAPIError as e:
        response = {'status': 'error', 'message': str(e)}
    
    if 'zapier_webhook' in data:
        requests.post(data['zapier_webhook'], json=response)
    return jsonify(response)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=int(os.getenv('PORT', 5000)))
