# Telegram Signal Bot (Deriv + Media Ads)

This bot connects to Deriv's Synthetic Indices, runs both:
- **Most Appearing Number (60 ticks)**
- **Adaptive Analysis (Under 9 / Over 2)**

Then it sends a signal to Telegram with:
- A captioned image (ad)
- Followed by a video ad (optional)

## 🚀 How to Deploy on Railway

1. Fork or upload this repo to GitHub.
2. Go to [Railway.app](https://railway.app) → New Project → Deploy from GitHub.
3. Add environment variables:
   - `TELEGRAM_BOT_TOKEN`
   - `TELEGRAM_CHAT_ID`
   - `DERIV_APP_ID`
   - (Optional) `SIGNAL_INTERVAL_MINUTES`
4. Railway will detect Python automatically.
5. Done! Logs show your bot’s activity.

## 🧠 Notes
- Images are sent *with* the signal (caption).
- Videos are sent *after* the signal.
- Uses both adaptive and tick-frequency logic.
- Fully compatible with GitHub & Railway hosting.

