from flask import Flask, jsonify

app = Flask(__name__)

@app.route("/")
def home():
    return jsonify({
        "app": "Mazaza Gold Alerts",
        "status": "online",
        "market": "XAU/USD",
        "signal_engine": "not_connected"
    })

@app.route("/health")
def health():
    return jsonify({"status": "ok"})

@app.route("/signal")
def signal():
    return jsonify({
        "status": "no_signal",
        "reason": "Live market data and SMC/ICT engine are not connected"
    })

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=10000)
