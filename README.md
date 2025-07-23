from aiogram import Bot, Dispatcher, executor, types
from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton
import logging

# 🔐 تنظیمات توکن و اطلاعات شما
BOT_TOKEN = "8090569085:AAFGsEOWD0dHPU5aGrDZ5pxJjsh82FK3cdY"
OWNER_ID = 7582765497  # آیدی عددی شما
CHANNEL_ID = -1001234567890  # آی‌دی عددی کانال (مثلاً -100xxxxxxxxxx)

logging.basicConfig(level=logging.INFO)
bot = Bot(token=BOT_TOKEN)
dp = Dispatcher(bot)

filters = set()
admins = {OWNER_ID}
banned_users = set()
chat_locked = False
anti_link = True
anti_forward = True
welcome_enabled = True
anti_spam_enabled = True
spam_users = {}

@dp.message_handler(content_types=types.ContentType.NEW_CHAT_MEMBERS)
async def welcome(message: types.Message):
    if welcome_enabled:
        for user in message.new_chat_members:
            await message.reply(f"👋 خوش آمدی {user.full_name}!")

@dp.message_handler(content_types=types.ContentType.TEXT)
async def filter_messages(message: types.Message):
    user_id = message.from_user.id
    if user_id in banned_users:
        await message.delete()
        return
    if user_id in admins:
        return
    if chat_locked:
        await message.delete()
        return
    if anti_link and any(e.type in ['url', 'text_link'] for e in (message.entities or [])):
        await message.delete()
        return
    if anti_forward and (message.forward_from or message.forward_from_chat):
        await message.delete()
        return
    if any(word in message.text.lower() for word in filters):
        await message.delete()
        return
    if anti_spam_enabled:
        spam_users[user_id] = spam_users.get(user_id, 0) + 1
        if spam_users[user_id] > 5:
            banned_users.add(user_id)
            await message.delete()
            await message.reply(f"⛔ {message.from_user.full_name} بن شد به دلیل اسپم.")

@dp.message_handler(commands=["پنل"])
async def admin_panel(message: types.Message):
    if message.from_user.id not in admins:
        return await message.reply("⛔ شما ادمین نیستید.")
    keyboard = InlineKeyboardMarkup(row_width=2).add(
        InlineKeyboardButton("➕ فیلتر", callback_data="add_filter"),
        InlineKeyboardButton("❌ حذف فیلتر", callback_data="remove_filter"),
        InlineKeyboardButton("📃 لیست فیلترها", callback_data="list_filters"),
        InlineKeyboardButton("🔒 قفل چت", callback_data="lock_chat"),
        InlineKeyboardButton("🔓 باز کردن چت", callback_data="unlock_chat"),
        InlineKeyboardButton("➕ ادمین", callback_data="add_admin"),
        InlineKeyboardButton("🚫 بن", callback_data="ban_user"),
        InlineKeyboardButton("✅ آن‌بن", callback_data="unban_user"),
        InlineKeyboardButton("📢 به کانال", callback_data="send_channel"),
        InlineKeyboardButton("🔗 ضد لینک", callback_data="toggle_link"),
        InlineKeyboardButton("🔁 ضد فوروارد", callback_data="toggle_forward"),
        InlineKeyboardButton("🎉 خوش‌آمد", callback_data="toggle_welcome"),
        InlineKeyboardButton("⚠️ ضد اسپم", callback_data="toggle_spam"),
    )
    await message.reply("🎛 پنل مدیریت:", reply_markup=keyboard)

@dp.callback_query_handler(lambda c: True)
async def panel_callbacks(call: types.CallbackQuery):
    data = call.data
    if call.from_user.id not in admins:
        return await call.answer("⛔ دسترسی ندارید", show_alert=True)

    global chat_locked, anti_link, anti_forward, welcome_enabled, anti_spam_enabled

    if data == "lock_chat":
        chat_locked = True
    elif data == "unlock_chat":
        chat_locked = False
    elif data == "toggle_link":
        anti_link = not anti_link
    elif data == "toggle_forward":
        anti_forward = not anti_forward
    elif data == "toggle_welcome":
        welcome_enabled = not welcome_enabled
    elif data == "toggle_spam":
        anti_spam_enabled = not anti_spam_enabled
    elif data == "add_filter":
        await call.message.answer("🔤 کلمه‌ای برای فیلتر بنویس:")
        @dp.message_handler()
        async def add_filter_message(msg: types.Message):
            filters.add(msg.text.lower())
            await msg.answer("✅ فیلتر اضافه شد.")
    elif data == "remove_filter":
        await call.message.answer("🧹 کلمه‌ای برای حذف فیلتر بنویس:")
        @dp.message_handler()
        async def remove_filter_message(msg: types.Message):
            filters.discard(msg.text.lower())
            await msg.answer("✅ حذف شد.")
    elif data == "list_filters":
        await call.message.answer("📃 لیست فیلترها:\n" + "\n".join(filters) if filters else "❌ فیلتر نداریم.")
    elif data == "ban_user":
        await call.message.answer("🧨 آی‌دی عددی کاربر:")
        @dp.message_handler()
        async def ban_msg(msg: types.Message):
            banned_users.add(int(msg.text))
            await msg.answer("✅ کاربر بن شد.")
    elif data == "unban_user":
        await call.message.answer("🆓 آی‌دی عددی کاربر:")
        @dp.message_handler()
        async def unban_msg(msg: types.Message):
            banned_users.discard(int(msg.text))
            await msg.answer("✅ آزاد شد.")
    elif data == "add_admin":
        await call.message.answer("👑 آی‌دی عددی کاربر:")
        @dp.message_handler()
        async def admin_msg(msg: types.Message):
            admins.add(int(msg.text))
            await msg.answer("✅ ادمین شد.")
    elif data == "send_channel":
        await call.message.answer("📨 متن برای ارسال به کانال:")
        @dp.message_handler()
        async def send_channel_msg(msg: types.Message):
            await bot.send_message(CHANNEL_ID, msg.text)
            await msg.answer("✅ ارسال شد.")

    await call.answer("عملیات انجام شد", show_alert=True)

if __name__ == "__main__":
    executor.start_polling(dp, skip_updates=True)
