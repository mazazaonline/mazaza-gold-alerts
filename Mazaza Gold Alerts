
import os
import requests
from flask import Flask, jsonify
from flask_cors import CORS

app = Flask(__name__)
CORS(app)

API_KEY = os.environ.get("TWELVE_DATA_API_KEY", "").strip()
API_URL = "https://api.twelvedata.com/time_series"
SYMBOL = "XAU/USD"


def get_candles(interval, outputsize=100):
    if not API_KEY:
        raise RuntimeError("Missing TWELVE_DATA_API_KEY in Render Environment")
    response = requests.get(API_URL, params={
        "symbol": SYMBOL,
        "interval": interval,
        "outputsize": outputsize,
        "timezone": "UTC",
        "apikey": API_KEY
    }, timeout=20)
    response.raise_for_status()
    payload = response.json()
    if payload.get("status") != "ok" or not payload.get("values"):
        raise RuntimeError(payload.get("message", "No market candles returned"))
    return [{
        "datetime": c["datetime"],
        "open": float(c["open"]),
        "high": float(c["high"]),
        "low": float(c["low"]),
        "close": float(c["close"])
    } for c in reversed(payload["values"])]


def analyze(candles, daily):
    if len(candles) < 20 or len(daily) < 2:
        raise RuntimeError("Insufficient candle history")

    previous_day = daily[-2]
    pdh, pdl = previous_day["high"], previous_day["low"]
    last = candles[-1]
    recent = candles[-16:-1]
    swing_low = min(c["low"] for c in recent)
    swing_high = max(c["high"] for c in recent)
    prior5 = candles[-6:-1]

    # Simplified three-candle FVG approximation
    bullish_fvg = last["low"] > candles[-3]["high"]
    bearish_fvg = last["high"] < candles[-3]["low"]

    # Simplified liquidity-sweep approximations
    bull_sweep = last["low"] < min(swing_low, pdl) and last["close"] > min(swing_low, pdl)
    bear_sweep = last["high"] > max(swing_high, pdh) and last["close"] < max(swing_high, pdh)
    bull_break = last["close"] > max(c["high"] for c in prior5)
    bear_break = last["close"] < min(c["low"] for c in prior5)

    result = {
        "status": "no_signal",
        "symbol": SYMBOL,
        "timeframe": "5min",
        "last_candle_utc": last["datetime"],
        "price": last["close"],
        "pdh": pdh,
        "pdl": pdl,
        "strategy": "Experimental simplified SMC/ICT rules",
        "note": "Not a validated Judas Swing strategy. Check data freshness and broker price differences."
    }

    side = None
    reasons = []
    if bull_sweep and bullish_fvg and bull_break:
        side = "BUY"
        reasons = ["Bullish sweep proxy", "Bullish FVG proxy", "Bullish structure break proxy"]
    elif bear_sweep and bearish_fvg and bear_break:
        side = "SELL"
        reasons = ["Bearish sweep proxy", "Bearish FVG proxy", "Bearish structure break proxy"]

    if side:
        entry = last["close"]
        stop = last["low"] if side == "BUY" else last["high"]
        risk = entry - stop if side == "BUY" else stop - entry
        if risk > 0:
            result.update({
                "status": "candidate_signal",
                "side": side,
                "entry": round(entry, 3),
                "stop_loss": round(stop, 3),
                "tp1": round(entry + risk if side == "BUY" else entry - risk, 3),
                "tp2": round(entry + 2*risk if side == "BUY" else entry - 2*risk, 3),
                "tp3": round(entry + 3*risk if side == "BUY" else entry - 3*risk, 3),
                "risk_reward": "1:1, 1:2, 1:3",
                "reasons": reasons,
                "warning": "For paper testing only; not financial advice or a guaranteed trade."
            })
    return result


@app.route("/")
def home():
    return jsonify({"app": "Mazaza Gold Alerts", "status": "online", "market": SYMBOL, "signal_engine": "prototype"})


@app.route("/health")
def health():
    return jsonify({"status": "ok", "api_key_configured": bool(API_KEY)})


@app.route("/candles")
def candles_endpoint():
    try:
        data = get_candles("5min", 100)
        return jsonify({"status": "ok", "symbol": SYMBOL, "interval": "5min", "count": len(data), "values": data})
    except Exception as exc:
        app.logger.exception("Candle request failed")
        return jsonify({"status": "error", "reason": str(exc)}), 502


@app.route("/signal")
def signal():
    try:
        candles = get_candles("5min", 100)
        daily = get_candles("1day", 10)
        return jsonify(analyze(candles, daily))
    except Exception as exc:
        app.logger.exception("Signal calculation failed")
        return jsonify({"status": "error", "reason": str(exc)}), 502


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", "10000")))
