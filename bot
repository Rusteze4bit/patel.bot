"""
Integrated Deriv Signal + Telegram Media Bot
===========================================

This single-file bot does the following:
- Connects to Deriv WebSocket and subscribes to tick streams for configured volatility indices.
- Collects ticks per market, maintains a rolling buffer (default 120 ticks)
- Runs **two analysis methods** on demand (both enabled):
  1) "Most appearing number in the last N ticks" (mode of last window)
  2) "Adaptive analysis" — dynamic window size, streaks, and under/over heuristic
- Every 30 minutes it runs analysis for all markets and, when a signal is produced,
  sends a Telegram signal message followed by an image (combined caption) and/or a video
  (video is sent after the text signal as requested).
- Ready for deployment on GitHub + Railway. Configure via environment variables.

Files you should push to your GitHub repo alongside this script:
- Procfile (content in the "Deployment" section below)
- requirements.txt (content in the "Deployment" section below)
- Optionally a .env file for local testing (example shown below).

IMPORTANT CONFIGURATION (environment variables)
- TELEGRAM_BOT_TOKEN: your Telegram bot token from @BotFather
- TELEGRAM_CHAT_ID: channel or chat id (e.g. @mychannel or -1001234567890)
- DERIV_WS_URL: optional, default uses public Deriv endpoint
- MARKETS: comma-separated symbol list for Deriv (default: R_10,R_25,R_50,R_75,R_100)
- TICKS_BUFFER: integer - how many ticks to keep per market (default: 120)
- SIGNAL_INTERVAL_MINUTES: integer - run analysis every N minutes (default: 30)
- MEDIA_IMAGE_URL: raw URL to the advertising image (optional — will be sent combined with signal)
- MEDIA_VIDEO_URL: raw URL to the advertising video (optional — will be sent *after* signal)
- LOG_LEVEL: INFO or DEBUG

Notes about markets: update MARKETS to the exact Deriv symbol names you use. Defaults are common volatility symbols.

---
Deployment (Procfile & requirements.txt)

Procfile contents:
worker: python telegram_deriv_signal_bot.py

requirements.txt contents:
requests
python-dotenv
websocket-client

Example .env (DO NOT commit secrets to public repos; use Railway environment variables instead):
TELEGRAM_BOT_TOKEN=your_token_here
TELEGRAM_CHAT_ID=@your_channel_or_chat_id
MARKETS=R_10,R_25,R_50,R_75,R_100
MEDIA_IMAGE_URL=https://raw.githubusercontent.com/youruser/yourrepo/main/ad.jpg
MEDIA_VIDEO_URL=https://raw.githubusercontent.com/youruser/yourrepo/main/ad.mp4

---
Usage notes:
- Push this file plus the Procfile and requirements.txt to GitHub. In Railway, create a new project and connect the repo; add environment variables in Railway settings.
- The bot runs continuously, collecting ticks. Every SIGNAL_INTERVAL_MINUTES it performs analysis and sends signals.

"""

# ---------------------- Imports ----------------------
import os
import time
import json
import logging
import threading
from collections import deque, Counter
from datetime import datetime, timezone
from typing import Dict, Deque, List, Any, Optional

import requests
import websocket

try:
    from dotenv import load_dotenv
    load_dotenv()
except Exception:
    pass

# ---------------------- Configuration ----------------------
TELEGRAM_BOT_TOKEN = os.getenv("TELEGRAM_BOT_TOKEN", "")
TELEGRAM_CHAT_ID = os.getenv("TELEGRAM_CHAT_ID", "")
DERIV_WS_URL = os.getenv("DERIV_WS_URL", "wss://ws.binaryws.com/websockets/v3?app_id=1089")
MARKETS = os.getenv("MARKETS", "R_10,R_25,R_50,R_75,R_100").split(",")
TICKS_BUFFER = int(os.getenv("TICKS_BUFFER", "120"))
SIGNAL_INTERVAL_MINUTES = int(os.getenv("SIGNAL_INTERVAL_MINUTES", "30"))
MEDIA_IMAGE_URL = os.getenv("MEDIA_IMAGE_URL", "")
MEDIA_VIDEO_URL = os.getenv("MEDIA_VIDEO_URL", "")
LOG_LEVEL = os.getenv("LOG_LEVEL", "INFO")

# Telegram API
BASE_TELEGRAM_URL = f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}"

# ---------------------- Logging ----------------------
logging.basicConfig(level=getattr(logging, LOG_LEVEL.upper(), logging.INFO), format="%(asctime)s - %(levelname)s - %(message)s")
logger = logging.getLogger("deriv-telegram-bot")

# ---------------------- State: tick buffers ----------------------
# For each symbol we store a deque of the last TICKS_BUFFER 'last_digit' values
TickBuffer = Dict[str, Deque[int]]
tick_buffers: TickBuffer = {symbol: deque(maxlen=TICKS_BUFFER) for symbol in MARKETS}
buffers_lock = threading.Lock()

# ---------------------- Deriv WebSocket handling ----------------------
ws: Optional[websocket.WebSocketApp] = None


def on_open(wsapp):
    logger.info("Deriv WS connected. Subscribing to ticks for: %s", ",".join(MARKETS))
    # Subscribe to ticks for each market
    for symbol in MARKETS:
        req = {"ticks": symbol}
        wsapp.send(json.dumps(req))


def on_message(wsapp, message):
    try:
        data = json.loads(message)
    except Exception as e:
        logger.exception("Failed to parse WS message: %s", e)
        return

    # Deriv returns ticks with keys like 'tick' or responses for subscriptions
    if "tick" in data:
        tick = data["tick"]
        symbol = tick.get("symbol")
        quote = tick.get("quote")
        if symbol and quote is not None:
            last_digit = int(str(quote)[-1]) if isinstance(quote, (int, float, str)) else None
            if last_digit is not None:
                with buffers_lock:
                    if symbol not in tick_buffers:
                        tick_buffers[symbol] = deque(maxlen=TICKS_BUFFER)
                    tick_buffers[symbol].append(last_digit)
                logger.debug("Tick %s -> %s", symbol, last_digit)


def on_error(wsapp, error):
    logger.exception("WebSocket error: %s", error)


def on_close(wsapp, close_status_code, close_msg):
    logger.warning("Deriv WS closed: %s %s", close_status_code, close_msg)


def start_deriv_ws():
    global ws
    ws = websocket.WebSocketApp(DERIV_WS_URL, on_open=on_open, on_message=on_message, on_error=on_error, on_close=on_close)

    def run_ws():
        while True:
            try:
                logger.info("Starting Deriv WebSocket...")
                ws.run_forever(ping_interval=30, ping_timeout=10)
            except Exception as e:
                logger.exception("WS loop exception: %s", e)
            logger.info("Reconnecting to Deriv WS in 5 seconds...")
            time.sleep(5)

    t = threading.Thread(target=run_ws, daemon=True)
    t.start()

# ---------------------- Analysis functions ----------------------


def most_appearing(buffer: Deque[int]) -> Dict[str, Any]:
    """Return the mode and its count within the buffer."""
    if not buffer:
        return {"mode": None, "count": 0, "total": 0}
    c = Counter(buffer)
    mode, count = c.most_common(1)[0]
    return {"mode": mode, "count": count, "total": len(buffer), "distribution": dict(c)}


def adaptive_analysis(buffer: Deque[int]) -> Dict[str, Any]:
    """
    Adaptive heuristic:
    - Choose dynamic window size between 20 and len(buffer) based on variability.
    - Compute streaks (longest tail of identical digits at the end).
    - Compute frequency and return candidate predictions and confidence score.
    This is intentionally heuristic — tune thresholds to your preference.
    """
    n = len(buffer)
    if n == 0:
        return {"candidate": None, "confidence": 0.0, "details": {}}

    # Basic variability measure: how many unique digits
    unique_count = len(set(buffer))
    # Lower variability -> larger window
    window = max(20, min(n, int(20 + (n - 20) * (1 - (unique_count / 10)))))
    window = min(window, n)
    window_slice = list(buffer)[-window:]

    freq = Counter(window_slice)
    candidate, cand_count = freq.most_common(1)[0]

    # streak: count how many identical digits at the end of buffer
    streak = 1
    for i in range(len(buffer) - 2, -1, -1):
        if buffer[i] == buffer[i + 1]:
            streak += 1
        else:
            break

    # Heuristic confidence: combination of relative frequency and streak length
    rel_freq = cand_count / window
    streak_bonus = min(streak / 10.0, 0.3)
    confidence = min(1.0, rel_freq * 0.7 + streak_bonus)

    # Under/Over heuristic (Under 9 / Over 2 style): show whether candidate is in lower or upper range
    under_over = "under" if candidate < 5 else "over"

    details = {
        "window": window,
        "window_sample": window_slice,
        "freq": dict(freq),
        "streak": streak,
        "rel_freq": rel_freq,
        "under_over": under_over,
    }

    return {"candidate": candidate, "confidence": confidence, "details": details}

# ---------------------- Telegram helpers ----------------------


def _telegram_request(method: str, files=None, data=None, json_payload=None):
    url = f"{BASE_TELEGRAM_URL}/{method}"
    for attempt in range(3):
        try:
            if files:
                resp = requests.post(url, files=files, data=data, timeout=30)
            elif json_payload is not None:
                resp = requests.post(url, json=json_payload, timeout=30)
            else:
                resp = requests.post(url, data=data or {}, timeout=30)
            if resp.status_code == 200:
                return resp.json()
            else:
                logger.warning("Telegram API %s returned %s: %s", method, resp.status_code, resp.text)
        except Exception as e:
            logger.exception("Telegram request error: %s", e)
        time.sleep(2)
    logger.error("Telegram request failed for method %s", method)
    return None


def send_text(chat_id: str, text: str) -> bool:
    payload = {"chat_id": chat_id, "text": text, "parse_mode": "Markdown", "disable_web_page_preview": True}
    r = _telegram_request("sendMessage", json_payload=payload)
    return bool(r and r.get("ok"))


def send_photo_by_url(chat_id: str, photo_url: str, caption: str = "") -> bool:
    payload = {"chat_id": chat_id, "photo": photo_url, "caption": caption, "parse_mode": "Markdown"}
    r = _telegram_request("sendPhoto", data=payload)
    return bool(r and r.get("ok"))


def send_video_by_url(chat_id: str, video_url: str, caption: str = "") -> bool:
    payload = {"chat_id": chat_id, "video": video_url, "caption": caption, "supports_streaming": True, "parse_mode": "Markdown"}
    r = _telegram_request("sendVideo", data=payload)
    return bool(r and r.get("ok"))

# ---------------------- Signal generation ----------------------


def analyze_market(symbol: str) -> Optional[Dict[str, Any]]:
    with buffers_lock:
        buffer = tick_buffers.get(symbol, deque())
        buf_copy = deque(buffer, maxlen=buffer.maxlen) if buffer else deque()

    if not buf_copy:
        logger.info("No data for %s, skipping analysis", symbol)
        return None

    # Run both analyses
    ma = most_appearing(buf_copy)
    ada = adaptive_analysis(buf_copy)

    # Decide whether to emit a signal. This is heuristic — adjust thresholds as you prefer.
    signals = []
    # If mode count occupies a significant portion of window -> add signal
    if ma["total"] > 20 and ma["count"] / ma["total"] >= 0.35:
        signals.append({"type": "mode", "value": ma["mode"], "confidence": ma["count"] / ma["total"]})

    # If adaptive confidence high
    if ada["confidence"] >= 0.55:
        signals.append({"type": "adaptive", "value": ada["candidate"], "confidence": ada["confidence"]})

    # If there is a streak >= 5, include
    streak_len = ada["details"].get("streak") if ada.get("details") else 0
    if streak_len and streak_len >= 5:
        signals.append({"type": "streak", "length": streak_len, "value": buf_copy[-1]})

    if not signals:
        logger.debug("No strong signals for %s", symbol)
        return None

    # Build final signal summary text
    now = datetime.now(timezone.utc).strftime('%Y-%m-%d %H:%M UTC')
    lines = [f"*Signal — {symbol}*", f"Time: {now}"]
    for s in signals:
        if s["type"] == "mode":
            lines.append(f"Mode: `{s['value']}` (freq {s['confidence']:.2f})")
        elif s["type"] == "adaptive":
            lines.append(f"Adaptive candidate: `{s['value']}` (conf {s['confidence']:.2f})")
        elif s["type"] == "streak":
            lines.append(f"Streak detected: last `{s['value']}` x{s['length']}")

    # include short stats
    lines.append(f"Window samples: {min(len(buf_copy), 60)} ticks used")

    signal_text = "
".join(lines)
    return {"symbol": symbol, "text": signal_text, "signals": signals}


def run_all_analyses_and_send():
    logger.info("Running scheduled analysis for all markets: %s", ",".join(MARKETS))
    for symbol in MARKETS:
        result = analyze_market(symbol)
        if result:
            try:
                # Send header signal text
                send_text(TELEGRAM_CHAT_ID, result["text"])                
                # If image exists, send combined with caption (image combined with signal)
                if MEDIA_IMAGE_URL:
                    caption = f"Advertisement / Sponsored:
{symbol} — check our service"
                    success_img = send_photo_by_url(TELEGRAM_CHAT_ID, MEDIA_IMAGE_URL, caption)
                    logger.info("Image sent: %s", success_img)

                # If video exists, send after the signal message
                if MEDIA_VIDEO_URL:
                    caption_vid = f"Sponsored video — {symbol}"
                    success_vid = send_video_by_url(TELEGRAM_CHAT_ID, MEDIA_VIDEO_URL, caption_vid)
                    logger.info("Video sent: %s", success_vid)

            except Exception as e:
                logger.exception("Failed to send signal for %s: %s", symbol, e)
        else:
            logger.debug("No signal for %s", symbol)

    logger.info("Analysis run completed.")


def scheduler_loop():
    interval = SIGNAL_INTERVAL_MINUTES * 60
    logger.info("Scheduler started: running analysis every %d minutes", SIGNAL_INTERVAL_MINUTES)
    while True:
        try:
            run_all_analyses_and_send()
        except Exception as e:
            logger.exception("Scheduler run error: %s", e)
        time.sleep(interval)

# ---------------------- Main ----------------------


def main():
    if not TELEGRAM_BOT_TOKEN or not TELEGRAM_CHAT_ID:
        logger.error("Missing TELEGRAM_BOT_TOKEN or TELEGRAM_CHAT_ID. Please set env vars.")
        return

    start_deriv_ws()

    # Start scheduler thread
    t = threading.Thread(target=scheduler_loop, daemon=True)
    t.start()

    # Keep main thread alive
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        logger.info("Shutting down by user interrupt")


if __name__ == "__main__":
    main()

