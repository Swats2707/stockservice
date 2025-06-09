# stockservice
import yfinance as yf
import numpy as np
import pandas as pd
from cachetools import TTLCache

class StockService:
    def __init__(self):
        self.cache = TTLCache(maxsize=100, ttl=3600)  # 1-hour cache

    def get_indicators(self, symbol: str) -> dict:
        """Fetch US stock data with indicators"""
        if symbol in self.cache:
            return self.cache[symbol]

        try:
            ticker = yf.Ticker(symbol)
            hist = ticker.history(period="2mo", interval="1d")

            if hist.empty:
                raise ValueError("No historical data available.")

            hist['SMA_20'] = hist['Close'].rolling(20).mean()
            hist['SMA_50'] = hist['Close'].rolling(50).mean()
            hist['RSI'] = self._calculate_rsi(hist['Close'])

            latest = hist.iloc[-1]
            data = {
                "symbol": symbol,
                "name": ticker.info.get('longName', symbol),
                "price": latest['Close'],
                "change": latest['Close'] - hist.iloc[-2]['Close'],
                "sma_20": latest['SMA_20'],
                "sma_50": latest['SMA_50'],
                "rsi": latest['RSI'],
                "risk": self._calculate_risk(latest)
            }

            self.cache[symbol] = data
            return data

        except Exception as e:
            raise ValueError(f"Failed to fetch {symbol}: {str(e)}")

    def _calculate_rsi(self, prices, window=14):
        delta = prices.diff()
        gain = delta.where(delta > 0, 0)
        loss = -delta.where(delta < 0, 0)
        avg_gain = gain.rolling(window).mean()
        avg_loss = loss.rolling(window).mean()
        rs = avg_gain / avg_loss
        return 100 - (100 / (1 + rs))

    def _calculate_risk(self, latest) -> str:
        if latest['RSI'] < 30 or latest['Close'] > latest['SMA_50']:
            return "safe"
        elif latest['RSI'] > 70 or latest['Close'] < latest['SMA_50'] * 0.95:
            return "risky"
        return "medium"

# Run the service
if __name__ == "__main__":
    service = StockService()
    result = service.get_indicators("AAPL")  # US stock
    for key, value in result.items():
        print(f"{key}: {value}")
