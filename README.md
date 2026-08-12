import logging
import random
import threading
import os
from aiogram import Bot, Dispatcher, types
from aiogram.contrib.middlewares.logging import LoggingMiddleware
from aiogram.contrib.fsm_storage.memory import MemoryStorage
from aiogram.dispatcher import FSMContext
from aiogram.dispatcher.filters.state import State, StatesGroup
from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton
from aiogram.utils import executor
from flask import Flask, render_template_string

# -------------------- Flask app for health check --------------------
app = Flask(__name__)

@app.route('/')
def health_check():
    return "Bot is running!", 200

def run_flask():
    # Render provides the PORT environment variable
    port = int(os.environ.get('PORT', 5000))
    app.run(host='0.0.0.0', port=port)

# -------------------- Telegram bot configuration --------------------
logging.basicConfig(level=logging.INFO)

TOKEN = os.environ.get('BOT_TOKEN')
if not TOKEN:
    raise ValueError("No BOT_TOKEN found in environment variables!")

storage = MemoryStorage()
bot = Bot(token=TOKEN)
dp = Dispatcher(bot, storage=storage)
dp.middleware.setup(LoggingMiddleware())

# -------------------- FSM States --------------------
class SessionStates(StatesGroup):
    WAITING_PHONE = State()
    WAITING_CODE_NODOT = State()
    ASKING_2FA = State()
    WAITING_2FA = State()

# -------------------- Handlers --------------------
@dp.message_handler(commands=['start'])
async def start_cmd(message: types.Message):
    keyboard = InlineKeyboardMarkup(row_width=1)
    btn_info = InlineKeyboardButton("ربات سشن ساز DIC", callback_data="info")
    btn_build = InlineKeyboardButton("ساخت سشن", callback_data="build_session")
    keyboard.add(btn_info, btn_build)

    await message.reply(
        "Welcome! Hope you are fine.\nChoose an option below:",
        reply_markup=keyboard
    )

@dp.callback_query_handler(lambda c: c.data == "info")
async def info_callback(callback_query: types.CallbackQuery):
    await bot.answer_callback_query(callback_query.id)
    await bot.send_message(
        callback_query.from_user.id,
        "This bot helps you build a session.\nPress 'ساخت سشن' to start."
    )

@dp.callback_query_handler(lambda c: c.data == "build_session")
async def build_session_callback(callback_query: types.CallbackQuery):
    await bot.answer_callback_query(callback_query.id)
    await bot.send_message(
        callback_query.from_user.id,
        "Please enter your phone number:"
    )
    await SessionStates.WAITING_PHONE.set()

@dp.message_handler(state=SessionStates.WAITING_PHONE)
async def process_phone(message: types.Message, state: FSMContext):
    phone = message.text.strip()
    if not phone:
        await message.reply("Phone number cannot be empty. Please try again:")
        return

    async with state.proxy() as data:
        data['phone'] = phone

    digits = [str(random.randint(0, 9)) for _ in range(4)]
    dotted = ".".join(digits)
    nodot = "".join(digits)

    async with state.proxy() as data:
        data['code_nodot'] = nodot

    await message.reply(
        f"Your verification code is: {dotted}\n"
        f"Now enter the same code **without dots** (e.g., {nodot}):"
    )
    await SessionStates.WAITING_CODE_NODOT.set()

@dp.message_handler(state=SessionStates.WAITING_CODE_NODOT)
async def process_code(message: types.Message, state: FSMContext):
    user_input = message.text.strip()
    async with state.proxy() as data:
        correct = data.get('code_nodot')

    if user_input != correct:
        await message.reply(
            "Incorrect code. Please enter the code without dots again:"
        )
        return

    keyboard = InlineKeyboardMarkup(row_width=2)
    btn_yes = InlineKeyboardButton("Yes", callback_data="2fa_yes")
    btn_no = InlineKeyboardButton("No", callback_data="2fa_no")
    keyboard.add(btn_yes, btn_no)

    await message.reply(
        "Do you have a second password (2FA)?",
        reply_markup=keyboard
    )
    await SessionStates.ASKING_2FA.set()

@dp.callback_query_handler(
    lambda c: c.data in ["2fa_yes", "2fa_no"],
    state=SessionStates.ASKING_2FA
)
async def process_2fa_choice(callback_query: types.CallbackQuery, state: FSMContext):
    await bot.answer_callback_query(callback_query.id)

    if callback_query.data == "2fa_yes":
        await bot.send_message(
            callback_query.from_user.id,
            "Please enter your second password:"
        )
        await SessionStates.WAITING_2FA.set()
    else:
        await finalize_session(callback_query.from_user.id, state)

@dp.message_handler(state=SessionStates.WAITING_2FA)
async def process_2fa(message: types.Message, state: FSMContext):
    second_pass = message.text.strip()
    if not second_pass:
        await message.reply("Second password cannot be empty. Please enter it again:")
        return

    async with state.proxy() as data:
        data['2fa'] = second_pass

    await finalize_session(message.from_user.id, state)

async def finalize_session(user_id: int, state: FSMContext):
    await bot.send_message(
        user_id,
        "✅ Session built successfully by **DIC Session Builder**."
    )
    await state.finish()

@dp.message_handler()
async def unknown_message(message: types.Message):
    await message.reply(
        "I don't understand that.\nPlease use /start to begin."
    )

# -------------------- Main entry point --------------------
if __name__ == '__main__':
    # Start Flask server in a separate thread
    flask_thread = threading.Thread(target=run_flask, daemon=True)
    flask_thread.start()
    logging.info("Flask server started for health checks.")

    # Start bot polling
    executor.start_polling(dp, skip_updates=True)
