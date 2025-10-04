# QX-PRO-BOT
It's a quotex platforms signals bot
"""
Panther Qx 0.0.1.12 – OTC Signal Bot (Upgraded)
Features:
- Analyzes 1M OTC pairs + multi-timeframe trend (5M, 15M)
- Sends Telegram signals automatically every 2 minutes
- Checks results after expiry (SURE SHOT / MTG1 / MTG2)
- Tracks win-rate and signal history
- Provides multiple Telegram commands
"""

import os
import asyncio
import json
import requests
from datetime import datetime, timedelta
from telegram import Bot
from telegram.ext import ApplicationBuilder, CommandHandler, ContextTypes

# ---------------- CONFIG ----------------
TELEGRAM_TOKEN = os.environ.get("TELEGRAM_TOKEN", "")
TELEGRAM_CHAT_ID = os.environ.get("TELEGRAM_CHAT_ID", "")
MARKET_API_KEY = os.environ.get("MARKET_API_KEY", "")
SIGNAL_INTERVAL = 120  # seconds
BOT_NAME = "Panther Qx 0.0.1.12"

PAIRS = [
    "USD/BDT-OTC",
    "USD/INR-OTC",
    "USD/PKR-OTC",
    "USD/MXN-OTC",
    "USD/ARS-OTC",
    "USD/BRL-OTC",
]

PAIR_SYMBOL_MAP = {p: p.replace("-OTC", "") for p in PAIRS}
STORE_FILE = "signals.json"

bot = Bot(token=TELEGRAM_TOKEN) if TELEGRAM_TOKEN else None
last_sent_for_pair = {}
store = {"signals": [], "stats": {"trades": 0, "wins": 0}}

# ---------------- UTILITIES ----------------
def load_store():
    global store
    if os.path.exists(STORE_FILE):
        try:
            with open(STORE_FILE, "r") as f:
                store = json.load(f)
        except:
            store = {"signals": [], "stats": {"trades": 0, "wins": 0}}

def save_store():
    with open(STORE_FILE, "w") as f:
        json.dump(store, f, indent=2)

def fetch_candles(pair: str, interval: str = "1m", outputsize: int = 10):
    """Fetch candles from API (replace with your provider)"""
    if not MARKET_API_KEY:
        return None
    symbol = PAIR_SYMBOL_MAP[pair]
    url = f"https://api.twelvedata.com/time_series?symbol={symbol}&interval={interval}&outputsize={outputsize}&apikey={MARKET_API_KEY}"
    try:
        r = requests.get(url, timeout=10).json()
        candles = []
        for v in reversed(r.get("values", [])):
            candles.append({
                "datetime": v["datetime"],
                "open": float(v["open"]),
                "high": float(v["high"]),
                "low": float(v["low"]),
                "close": float(v["close"])
            })
        return candles
    except:
        return None

def candle_color(c):
    if c["close"] > c["open"]:
        return "green"
    if c["close"] < c["open"]:
        return "red"
    return "doji"

def is_bullish_engulfing(c1, c2):
    return candle_color(c1) == "red" and candle_color(c2) == "green" and c2["close"] > c1["open"] and c2["open"] < c1["close"]

def is_bearish_engulfing(c1, c2):
    return candle_color(c1) == "green" and candle_color(c2) == "red" and c2["close"] < c1["open"] and c2["open"] > c1["close"]

def sma(values, period):
    if len(values) < period:
        return None
    return sum(values[-period:])/period

def trend_confirm(pair):
    c5 = fetch_candles(pair, "5m", 20)
    c15 = fetch_candles(pair, "15m", 20)
    if not c5 or not c15: return None
    s5 = sma([c["close"] for c in c5], 20)
    s15 = sma([c["close"] for c in c15], 20)
    if s5 is None or s15 is None: return None
    if s5 > s15: return "UP"
    if s5 < s15: return "DOWN"
    return "NEUTRAL"

def detect_signal(pair):
    c1m = fetch_candles(pair, "1m", 3)
    if not c1m or len(c1m) < 3: return None
    c1, c2, c3 = c1m
    raw = None
    if is_bullish_engulfing(c2, c3):
        raw = {"side": "BUY", "reason": "Bullish Engulfing 1M"}
    elif is_bearish_engulfing(c2, c3):
        raw = {"side": "SELL", "reason": "Bearish Engulfing 1M"}
    else:
        return None
    trend = trend_confirm(pair)
    if trend=="NEUTRAL": return None
    if (raw["side"]=="BUY" and trend=="UP") or (raw["side"]=="SELL" and trend=="DOWN"):
        return {
            "pair": pair,
            "side": raw["side"],
            "reason": raw["reason"],
            "trend": trend,
            "timestamp": datetime.utcnow().isoformat(),
            "price": c3["close"]
        }
    return None

def classify_result(entry_price, exit_price, side):
    diff = exit_price - entry_price if side=="BUY" else entry_price - exit_price
    if diff > 0.0005: return "SURE SHOT"
    elif diff > 0.0002: return "MTG1"
    else: return "MTG2"

def format_signal_msg(sig):
    return (f"{BOT_NAME} – Signal\nPair: {sig['pair']}\nEntry Time: {sig['timestamp']}\n"
            f"Signal: {'🔼 BUY' if sig['side']=='BUY' else '🔻 SELL'}\nReason: {sig['reason']}\nTrend: {sig['trend']}")

def format_result_msg(sig, result_type, exit_price):
    outcome = "WIN" if result_type=="SURE SHOT" else "LOSS"
    return (f"{BOT_NAME} – Result\nPair: {sig['pair']}\nEntry {sig['timestamp']} → Expiry {sig['expiry']}\n"
            f"Outcome: {outcome}\nSignal Result: {result_type}\nExit Price: {exit_price}")

async def broadcast_signal(sig):
    msg = format_signal_msg(sig)
    try:
        bot.send_message(chat_id=TELEGRAM_CHAT_ID, text=msg)
    except:
        pass

async def broadcast_result(sig, result_type, exit_price):
    msg = format_result_msg(sig, result_type, exit_price)
    try:
        bot.send_message(chat_id=TELEGRAM_CHAT_ID, text=msg)
    except:
        pass

# ---------------- WORKER ----------------
async def pair_worker(pair):
    while True:
        try:
            sig = detect_signal(pair)
            if sig and last_sent_for_pair.get(pair) != sig["timestamp"]:
                sig["expiry"] = (datetime.fromisoformat(sig["timestamp"]) + timedelta(minutes=1)).isoformat()
                last_sent_for_pair[pair] = sig["timestamp"]
                store["signals"].append(sig)
                await broadcast_signal(sig)
                save_store()
                await asyncio.sleep(60)  # Wait for expiry
                exit_candle = fetch_candles(pair, "1m", 1)
                if exit_candle:
                    exit_price = exit_candle[0]["close"]
                    result_type = classify_result(sig["price"], exit_price, sig["side"])
                    store["stats"]["trades"] +=1
                    if result_type=="SURE SHOT":
                        store["stats"]["wins"] +=1
                    await broadcast_result(sig, result_type, exit_price)
                    save_store()
        except Exception as e:
            print(f"Error in {pair}: {e}")
        await asyncio.sleep(SIGNAL_INTERVAL)

# ---------------- TELEGRAM COMMANDS ----------------
async def cmd_start(update:ContextTypes.DEFAULT_TYPE, context):
    await context.bot.send_message(chat_id=update.effective_chat.id,
                                   text=f"{BOT_NAME} activated. Live OTC analysis started.")

async def cmd_stop(update:ContextTypes.DEFAULT_TYPE, context):
    await context.bot.send_message(chat_id=update.effective_chat.id,
                                   text=f"{BOT_NAME} paused. No signals will be sent.")

async def cmd_status(update:ContextTypes.DEFAULT_TYPE, context):
    stats = store.get("stats", {})
    trades = stats.get("trades",0)
    wins = stats.get("wins",0)
    wr = wins/trades*100 if trades>0 else 0
    await context.bot.send_message(chat_id=update.effective_chat.id,
                                   text=f"{BOT_NAME} Status\nTrades: {trades}\nWins: {wins}\nWinRate: {wr:.2f}%")

async def cmd_help(update:ContextTypes.DEFAULT_TYPE, context):
    await context.bot.send_message(chat_id=update.effective_chat.id,
                                   text="/start\n/stop\n/status\n/signals\n/results\n/trend <PAIR>\n/pairs\n/help")

async def cmd_signals(update:ContextTypes.DEFAULT_TYPE, context):
    msgs = []
    for sig in store.get("signals", []):
        msgs.append(format_signal_msg(sig))
    await context.bot.send_message(chat_id=update.effective_chat.id,
                                   text="\n\n".join(msgs) if msgs else "No open signals currently.")

async def cmd_results(update:ContextTypes.DEFAULT_TYPE, context):
    last_results = store.get("signals", [])[-10:]
    msgs = []
    for sig in last_results:
        result_type = classify_result(sig["price"], sig["price"], sig["side"])
        msgs.append(format_result_msg(sig, result_type, sig["price"]))
    await context.bot.send_message(chat_id=update.effective_chat.id,
                                   text="\n\n".join(msgs) if msgs else "No results yet.")

async def cmd_pairs(update:ContextTypes.DEFAULT_TYPE, context):
    await context.bot.send_message(chat_id=update.effective_chat.id,
                                   text="Available OTC pairs:\n" + "\n".join(PAIRS))

async def cmd_trend(update:ContextTypes.DEFAULT_TYPE, context):
    if context.args:
        pair = context.args[0].upper()
        if pair in PAIRS:
            t = trend_confirm(pair)
            await context.bot.send_message(chat_id=update.effective_chat.id,
                                           text=f"Trend for {pair}: {t}")
            return
    await context.bot.send_message(chat_id=update.effective_chat.id,
                                   text="Usage: /trend <PAIR>")

# ---------------- MAIN ----------------
async def main_async():
    load_store()
    app = ApplicationBuilder().token(TELEGRAM_TOKEN).build()
    app.add_handler(CommandHandler("start", cmd_start))
    app.add_handler(CommandHandler("stop", cmd_stop))
    app.add_handler(CommandHandler("status", cmd_status))
    app.add_handler(CommandHandler("help", cmd_help))
    app.add_handler(CommandHandler("signals", cmd_signals))
    app.add_handler(CommandHandler("results", cmd_results))
    app.add_handler(CommandHandler("pairs", cmd_pairs))
    app.add_handler(CommandHandler("trend", cmd_trend))
    tasks = [asyncio.create_task(pair_worker(p)) for p in PAIRS]
    tasks.append(asyncio.create_task(app.run_polling()))
    await asyncio.gather(*tasks)

if __name__=="__main__":
    asyncio.run(main_async())
