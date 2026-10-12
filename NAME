import os
import asyncio
import logging
import sqlite3
from aiohttp import web
from aiogram import Bot, Dispatcher, F, types
from aiogram.filters import Command
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.types import InlineKeyboardButton, InlineKeyboardMarkup
from aiogram.webhook.aiohttp_server import SimpleRequestHandler, setup_application
from apscheduler.schedulers.asyncio import AsyncIOScheduler

# --- ATROF-MUHIT O'ZGARUVCHILARI ---
BOT_TOKEN = os.getenv("BOT_TOKEN")
ADMIN_ID = int(os.getenv("ADMIN_ID", "0"))
WEBHOOK_HOST = os.getenv("WEBHOOK_HOST")  # Masalan: https://house-duty-bot.onrender.com
WEBHOOK_PATH = f"/webhook/{BOT_TOKEN}"
WEBHOOK_URL = f"{WEBHOOK_HOST}{WEBHOOK_PATH}"
PORT = int(os.getenv("PORT", 8080))

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()
scheduler = AsyncIOScheduler(timezone="Asia/Tashkent")

# --- DATABASE SETUP ---
conn = sqlite3.connect("house_duty.db", check_same_thread=False)
cursor = conn.cursor()

cursor.execute('''CREATE TABLE IF NOT EXISTS users 
                  (user_id INTEGER PRIMARY KEY, name TEXT, duty_order INTEGER)''')
cursor.execute('''CREATE TABLE IF NOT EXISTS tasks 
                  (id INTEGER PRIMARY KEY AUTOINCREMENT, task_name TEXT)''')
cursor.execute('''CREATE TABLE IF NOT EXISTS shopping 
                  (id INTEGER PRIMARY KEY AUTOINCREMENT, item TEXT)''')
cursor.execute('''CREATE TABLE IF NOT EXISTS status 
                  (key TEXT PRIMARY KEY, value INTEGER)''')

# Odatiy vazifalarni kiritish
cursor.execute("SELECT COUNT(*) FROM tasks")
if cursor.fetchone()[0] == 0:
    default_tasks = [
        ("🗑 Musorlarni chiqarib tashlash",),
        ("🍽 Qozon-tovoqlarni yuvib olish",),
        ("🛁 Vannani qillardan tozalash",),
        ("🚽 Hojatxona musorini to'kish",),
        ("🧹 Xona va oshxonani supurish",)
    ]
    cursor.executemany("INSERT INTO tasks (task_name) VALUES (?)", default_tasks)

cursor.execute("INSERT OR IGNORE INTO status VALUES ('current_index', 0)")
conn.commit()

# --- STATES ---
class AdminStates(StatesGroup):
    waiting_for_broadcast = State()
    waiting_for_task = State()

class UserStates(StatesGroup):
    waiting_for_item = State()

# --- YORDAMCHI FUNKSIYALAR ---
def get_users():
    cursor.execute("SELECT user_id, name FROM users ORDER BY duty_order ASC")
    return cursor.fetchall()

def get_current_duty_user():
    users = get_users()
    if not users:
        return None
    cursor.execute("SELECT value FROM status WHERE key = 'current_index'")
    idx = cursor.fetchone()[0] % len(users)
    return users[idx]

def rotate_duty():
    users = get_users()
    if users:
        cursor.execute("SELECT value FROM status WHERE key = 'current_index'")
        idx = cursor.fetchone()[0]
        cursor.execute("UPDATE status SET value = ? WHERE key = 'current_index'", (idx + 1,))
        conn.commit()

# --- CRON JOB: ERTALABGI 07:00 ESLATMA ---
async def morning_reminder():
    user = get_current_duty_user()
    if not user:
        return

    cursor.execute("SELECT task_name FROM tasks")
    tasks = cursor.fetchall()
    task_text = "\n".join([f"• {t[0]}" for t in tasks])

    hadith = ("✨ *Hadis:* «Albatta, Alloh pokdir, poklikni sevadi; "
              "ozodalikni sevadi, hovli-joylaringizni tozalanglar»")

    msg = (f"☀️ *Xayrli tong! Bugungi navbatchi:* <a href='tg://user?id={user[0]}'>{user[1]}</a>\n\n"
           f"📋 *Bugungi bajarilishi kerak bo'lgan ishlar:*\n{task_text}\n\n{hadith}")

    users = get_users()
    for u in users:
        try:
            await bot.send_message(u[0], msg, parse_mode="HTML")
        except Exception:
            pass

# --- HANDLERS ---
@dp.message(Command("start"))
async def start_cmd(message: types.Message):
    cursor.execute("INSERT OR IGNORE INTO users VALUES (?, ?, (SELECT COUNT(*) FROM users))", 
                   (message.from_user.id, message.from_user.full_name))
    conn.commit()

    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🛒 Xaridlar ro'yxati", callback_data="show_shopping")],
        [InlineKeyboardButton(text="➕ Xarid kerak", callback_data="add_shopping")]
    ])
    
    await message.answer(
        f"Assalomu aleykum, {message.from_user.first_name}!\n"
        f"Xona navbatchilik botiga xush kelibsiz.\n"
        f"Tozalik hisobotini yuborish uchun shunchaki **rasm va izoh** yuboring!",
        reply_markup=kb, parse_mode="Markdown"
    )

# 📢 ADMIN BROADCAST
@dp.message(Command("broadcast"))
async def broadcast_cmd(message: types.Message, state: FSMContext):
    if message.from_user.id != ADMIN_ID:
        await message.answer("⚠️ Siz admin emassiz!")
        return
    await message.answer("📢 Barcha xonadoshlarga yuboriladigan xabarni kiriting:")
    await state.set_state(AdminStates.waiting_for_broadcast)

@dp.message(AdminStates.waiting_for_broadcast)
async def process_broadcast(message: types.Message, state: FSMContext):
    users = get_users()
    count = 0
    for u in users:
        try:
            await bot.send_message(u[0], f"📢 *Admin xabari:*\n\n{message.text}", parse_mode="Markdown")
            count += 1
        except Exception:
            pass
    await message.answer(f"✅ Xabar {count} ta xonadoshga yuborildi.")
    await state.clear()

# ➕ ADMIN VAZIFA QO'SHISH
@dp.message(Command("addtask"))
async def add_task_cmd(message: types.Message, state: FSMContext):
    if message.from_user.id != ADMIN_ID:
        return
    await message.answer("Yangi vazifa nomini kiriting:")
    await state.set_state(AdminStates.waiting_for_task)

@dp.message(AdminStates.waiting_for_task)
async def process_add_task(message: types.Message, state: FSMContext):
    cursor.execute("INSERT INTO tasks (task_name) VALUES (?)", (message.text,))
    conn.commit()
    await message.answer("✅ Yangi vazifa muvaffaqiyatli qo'shildi!")
    await state.clear()

# 📸 HISOBOT TOPShIRISH VA SHERIKLAR ROZILIGI
@dp.message(F.photo)
async def handle_report(message: types.Message):
    duty_user = get_current_duty_user()
    if not duty_user or message.from_user.id != duty_user[0]:
        await message.answer("⚠️ Bugun siz navbatchi emassiz!")
        return

    users = get_users()
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="✅ Tasdiqlayman", callback_data="approve_work")]
    ])

    for u in users:
        if u[0] != message.from_user.id:
            try:
                await bot.send_photo(
                    u[0],
                    photo=message.photo[-1].file_id,
                    caption=f"📸 **Navbatchi ({duty_user[1]}) hisobot topshirdi!**\n\n"
                            f"Izoh: {message.caption or 'Yoq'}\n\n"
                            f"Ish sifatiga rozimisiz?",
                    reply_markup=kb, parse_mode="Markdown"
                )
            except Exception:
                pass
    await message.answer("✅ Hisobot barcha xonadoshlarga yuborildi!")

@dp.callback_query(F.data == "approve_work")
async def approve_callback(callback: types.CallbackQuery):
    await callback.answer("Rahmat, roziligingiz qabul qilindi!", show_alert=True)
    await callback.message.edit_caption(
        caption=callback.message.caption + f"\n\n✅ **{callback.from_user.first_name} tasdiqladi.**",
        parse_mode="Markdown"
    )

# --- 🛒 XARIDLAR TIZIMI ---
@dp.callback_query(F.data == "add_shopping")
async def add_shop_cb(callback: types.CallbackQuery, state: FSMContext):
    await callback.message.answer("Tugayotgan buyum nomini yozing (masalan: Idish yuvish geli):")
    await state.set_state(UserStates.waiting_for_item)
    await callback.answer()

@dp.message(UserStates.waiting_for_item)
async def process_shop_item(message: types.Message, state: FSMContext):
    cursor.execute("INSERT INTO shopping (item) VALUES (?)", (message.text,))
    conn.commit()
    await message.answer("🛒 Xaridlar ro'yxatiga qo'shildi!")
    await state.clear()

@dp.callback_query(F.data == "show_shopping")
async def show_shop_cb(callback: types.CallbackQuery):
    cursor.execute("SELECT item FROM shopping")
    items = cursor.fetchall()
    text = "\n".join([f"• {i[0]}" for i in items]) if items else "Hozircha xaridlar yo'q."
    await callback.message.answer(f"🛒 **Xaridlar ro'yxati:**\n\n{text}", parse_mode="Markdown")
    await callback.answer()

# --- WEBHOOK & LIFECYCLE ---
async def on_startup(app):
    await bot.set_webhook(WEBHOOK_URL)
    scheduler.add_job(morning_reminder, "cron", hour=7, minute=0)
    scheduler.add_job(rotate_duty, "cron", hour=0, minute=0)
    scheduler.start()

async def on_shutdown(app):
    await bot.delete_webhook()

def main():
    logging.basicConfig(level=logging.INFO)
    app = web.Application()
    
    app.router.add_get("/", lambda r: web.Response(text="Bot ishlomoqda!"))
    SimpleRequestHandler(dispatcher=dp, bot=bot).register(app, path=WEBHOOK_PATH)
    
    app.on_startup.append(on_startup)
    app.on_shutdown.append(on_shutdown)
    
    setup_application(app, dp, bot=bot)
    web.run_app(app, host="0.0.0.0", port=PORT)

if __name__ == "__main__":
    main()
