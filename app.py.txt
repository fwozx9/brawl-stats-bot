import os
import asyncio
import threading
import requests
from flask import Flask
from aiogram import Bot, Dispatcher
from aiogram.filters import CommandStart, Command
from aiogram.types import Message

TOKEN = os.environ.get("TELEGRAM_TOKEN")
if not TOKEN:
    raise RuntimeError("TELEGRAM_TOKEN не задан!")

BRAWL_API_KEY = os.environ.get("BRAWL_API_KEY")
if not BRAWL_API_KEY:
    raise RuntimeError("BRAWL_API_KEY не задан!")

BASE_URL = "https://api.brawlstars.com/v1"

bot = Bot(token=TOKEN)
dp = Dispatcher()

def clean_tag(tag):
    return tag.replace('#', '').upper().strip()

def get_player(tag):
    url = f"{BASE_URL}/players/%23{clean_tag(tag)}"
    headers = {"Authorization": f"Bearer {BRAWL_API_KEY}"}
    try:
        resp = requests.get(url, headers=headers, timeout=30)
        return resp.json() if resp.status_code == 200 else None
    except:
        return None

@dp.message(CommandStart())
async def start(message: Message):
    await message.answer(
        "🎮 **Brawl Stars Stats Bot**\n\n"
        "Отправь тег игрока:\n"
        "`/stats #2PPU89J2`"
    )

@dp.message(Command("stats"))
async def stats_command(message: Message):
    args = message.text.split()
    if len(args) < 2:
        await message.answer("❌ Укажи тег: `/stats #2PPU89J2`")
        return
    
    tag = args[1]
    await message.answer(f"🔍 Загружаю {tag}...")
    
    player = get_player(tag)
    
    if not player or 'error' in player:
        await message.answer("❌ Игрок не найден")
        return
    
    name = player.get('name', 'Неизвестно')
    trophies = player.get('trophies', 0)
    highest = player.get('highestTrophies', 0)
    level = player.get('expLevel', 0)
    wins = player.get('wins', 0)
    club = player.get('club', {}).get('name', 'Нет клуба')
    
    response = (
        f"👤 **{name}**\n"
        f"🏆 Трофеи: {trophies} (макс: {highest})\n"
        f"⭐ Уровень: {level}\n"
        f"👑 Победы: {wins}\n"
        f"🏰 Клуб: {club}"
    )
    
    await message.answer(response)

async def run_bot():
    await dp.start_polling(bot)

def run_http():
    app = Flask(__name__)
    
    @app.route("/")
    def root():
        return "Brawl Stars Bot is running!", 200
    
    @app.route("/health")
    def health():
        return "OK", 200
    
    port = int(os.environ.get("PORT", "5000"))
    app.run(host="0.0.0.0", port=port)

if __name__ == "__main__":
    threading.Thread(target=run_http, daemon=True).start()
    asyncio.run(run_bot())