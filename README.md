# Football Alert Bot v2

import os
import aiohttp
import asyncio
from telegram import Update
from telegram.ext import Application, CommandHandler, ContextTypes

# Variabili d'ambiente
TELEGRAM_BOT_TOKEN = os.getenv("TELEGRAM_BOT_TOKEN")
API_FOOTBALL_KEY = os.getenv("API_FOOTBALL_KEY")
TELEGRAM_CHAT_ID = os.getenv("TELEGRAM_CHAT_ID")

POLL_SECONDS = int(os.getenv("POLL_SECONDS", 60))  # intervallo controlli
DEFAULT_THRESHOLD = float(os.getenv("DEFAULT_THRESHOLD", 0.7))  # soglia probabilità

# === Funzioni Telegram ===
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("⚽ Bot attivo! Ti avviserò se c’è un’alta probabilità di gol.")

# === Analisi partite ===
async def fetch_live_matches():
    """Scarica le partite live da API-Football"""
    url = "https://v3.football.api-sports.io/fixtures?live=all"
    headers = {"x-apisports-key": API_FOOTBALL_KEY}
    async with aiohttp.ClientSession() as session:
        async with session.get(url, headers=headers) as resp:
            if resp.status != 200:
                print("Errore API:", resp.status)
                return []
            data = await resp.json()
            return data.get("response", [])

def compute_goal_probability(stats: dict) -> float:
    """
    Calcola una probabilità semplificata di gol basata su:
    - Expected Goals (xG)
    - Tiri in porta
    - Attacchi pericolosi
    """
    xg_home = stats.get("xG_home", 0)
    xg_away = stats.get("xG_away", 0)
    shots_home = stats.get("shots_on_home", 0)
    shots_away = stats.get("shots_on_away", 0)
    attacks_home = stats.get("attacks_home", 0)
    attacks_away = stats.get("attacks_away", 0)

    score = (xg_home + xg_away) * 0.5
    score += (shots_home + shots_away) * 0.05
    score += (attacks_home + attacks_away) * 0.01

    return min(score / 2, 1.0)  # normalizzato [0-1]

async def analyze_matches(app: Application):
    """Controlla live match e manda alert su Telegram"""
    matches = await fetch_live_matches()
    for match in matches:
        teams = match["teams"]
        stats = match.get("statistics", [])

        # NB: Alcune API hanno stats separate per team → qui simuliamo estrazione
        fake_stats = {
            "xG_home": 1.2,
            "xG_away": 0.8,
            "shots_on_home": 5,
            "shots_on_away": 3,
            "attacks_home": 20,
            "attacks_away": 15
        }

        prob = compute_goal_probability(fake_stats)
        if prob >= DEFAULT_THRESHOLD:
            message = (
                f"⚡ Probabilità alta di gol!\\n"
                f"{teams['home']['name']} 🆚 {teams['away']['name']}\\n"
                f"Probabilità stimata: {prob:.2%}"
            )
            await app.bot.send_message(chat_id=TELEGRAM_CHAT_ID, text=message)

async def background_task(app: Application):
    """Loop infinito per analizzare partite"""
    while True:
        try:
            await analyze_matches(app)
        except Exception as e:
            print("Errore nel loop:", e)
        await asyncio.sleep(POLL_SECONDS)

# === Main ===
def main():
    if not TELEGRAM_BOT_TOKEN or not API_FOOTBALL_KEY or not TELEGRAM_CHAT_ID:
        raise ValueError("❌ Manca una variabile d'ambiente: TELEGRAM_BOT_TOKEN, API_FOOTBALL_KEY o TELEGRAM_CHAT_ID")

    app = Application.builder().token(TELEGRAM_BOT_TOKEN).build()
    app.add_handler(CommandHandler("start", start))

    # Avvia il task in background correttamente
    async def on_startup(app):
        asyncio.create_task(background_task(app))

    app.post_init = on_startup

    app.run_polling()

if __name__ == "__main__":
    main()
    

## Variabili d'ambiente richieste
- 8228166343:AAE8cP3nymWroOhBNP6ylHlzWEa2z1O1CH0
- 218af83359cc0bd584387e47a5792cd8
- 1228335702
