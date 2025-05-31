# CRT BOT - MTF + CRT Algı + Flask + Telegram
import requests
from flask import Flask
import threading
import time
import os

# Telegram ayarları (Koyeb Environment Variables'dan alınır)
BOT_TOKEN = os.getenv("BOT_TOKEN")
CHAT_ID = os.getenv("CHAT_ID")

# İzlenecek coinler
PAIRS = ["BTCUSDT", "ETHUSDT"]

# Zaman dilimleri
TIMEFRAMES = {
    "1h": "1h",
    "4h": "4h",
    "1d": "1d",
    "1w": "1w"
}

# Flask uygulaması
app = Flask(__name__)

@app.route('/')
def home():
    return "CRT Bias Bot is running!"

@app.route('/ping')
def ping():
    return "pong"

# Telegram mesaj gönderimi
def send_signal(pair, tf, direction, price):
    emoji = "📈" if direction == "Long" else "📉"
    tp = price * 1.04 if direction == "Long" else price * 0.96
    sl = price * 0.98 if direction == "Long" else price * 1.02
    rr = abs(price - tp) / abs(sl - price)
    text = f"""{emoji} CRT {direction} Signal | #{pair}
🕒 Zaman Dilimi: {tf.upper()}
💰 Fiyat: {price:.2f}
🎯 Take Profit: {tp:.2f}
🛑 Stop Loss: {sl:.2f}
💎 Risk-Ödül Oranı (RR): {rr:.2f}"""
    url = f"https://api.telegram.org/bot{BOT_TOKEN}/sendMessage"
    requests.post(url, data={"chat_id": CHAT_ID, "text": text})

# Binance candle verisi çek
def get_candles(pair, interval, limit=5):
    url = f"https://api.binance.com/api/v3/klines?symbol={pair}&interval={interval}&limit={limit}"
    res = requests.get(url)
    if res.status_code == 200:
        return res.json()
    return []

# CRT tespiti (3 mum analizi)
def detect_crt(candles):
    if len(candles) < 3:
        return False, None

    c1 = candles[-3]
    c2 = candles[-2]
    c3 = candles[-1]

    o1, h1, l1, c1c = float(c1[1]), float(c1[2]), float(c1[3]), float(c1[4])
    o2, h2, l2, c2c = float(c2[1]), float(c2[2]), float(c2[3]), float(c2[4])
    o3, h3, l3, c3c = float(c3[1]), float(c3[2]), float(c3[3]), float(c3[4])

    # Bullish CRT
    if l2 < l1 and c2c > o2 and c3c > o3:
        return True, "Long"

    # Bearish CRT
    if h2 > h1 and c2c < o2 and c3c < o3:
        return True, "Short"

    return False, None

# Tüm coin ve zaman dilimlerini tara
def analyze_all():
    for pair in PAIRS:
        for label, tf in TIMEFRAMES.items():
            candles = get_candles(pair, tf)
            crt_found, direction = detect_crt(candles)
            if crt_found:
                price = float(candles[-1][4])
                send_signal(pair, label, direction, price)

# Saatlik kontrol döngüsü
def run_bot():
    while True:
        analyze_all()
        time.sleep(3600)  # Her saat çalışır

# Flask ve analiz döngüsünü başlat
if __name__ == "__main__":
    threading.Thread(target=run_bot).start()
    app.run(host="0.0.0.0", port=8080)
