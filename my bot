import os
import asyncio
import logging
from aiogram import Bot, Dispatcher, types, executor
from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton, ReplyKeyboardMarkup
from telethon import TelegramClient, errors

# ══════════════════════════════════════
#         الإعدادات (بياناتك جاهزة)
# ══════════════════════════════════════
API_ID = '32583989' 
API_HASH = '5c64789b232f538443ad38a4aa998599'
BOT_TOKEN = '8776893445:AAH2V13AgX7Vy6GSZBBl8IsGVcHAEmTKl30'

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher(bot)
logging.basicConfig(level=logging.INFO)

# مخازن البيانات (في الذاكرة)
user_sessions = {}  # العمليات الجارية
saved_accounts = [] # الحسابات اللي اتفعلت

if not os.path.exists('sessions'):
    os.makedirs('sessions')

# ══════════════════════════════════════
#          الواجهة الرئيسية
# ══════════════════════════════════════

def main_kb():
    kb = ReplyKeyboardMarkup(resize_keyboard=True, row_width=2)
    kb.add("➕ تفعيل رقم جديد", "📂 حساباتي")
    return kb

@dp.message_handler(commands=['start'])
async def start(message: types.Message):
    await message.answer("🚀 **مرحباً بك في المنظومة الذكية**\nجاهز لسحب وتفعيل الحسابات بأعلى كفاءة.", reply_markup=main_kb())

# ══════════════════════════════════════
#        قسم إدارة الحسابات المحفوظة
# ══════════════════════════════════════

@dp.message_handler(lambda m: m.text == "📂 حساباتي")
async def my_accounts(message: types.Message):
    if not saved_accounts:
        return await message.answer("❌ لا يوجد حسابات مفعلة حالياً.")
    
    kb = InlineKeyboardMarkup(row_width=1)
    for phone in saved_accounts:
        kb.add(InlineKeyboardButton(f"📱 {phone}", callback_data=f"manage:{phone}"))
    await message.answer("📂 قائمة حساباتك الجاهزة للسحب:", reply_markup=kb)

@dp.callback_query_handler(lambda c: c.data.startswith('manage:'))
async def manage_opt(callback: types.CallbackQuery):
    phone = callback.data.split(":")[1]
    kb = InlineKeyboardMarkup(row_width=1).add(
        InlineKeyboardButton("📩 سحب كود الواتساب (آخر رسالة)", callback_data=f"pull:{phone}"),
        InlineKeyboardButton("🔄 تحديث", callback_data=f"pull:{phone}"),
        InlineKeyboardButton("⬅️ رجوع", callback_data="back_list")
    )
    await bot.edit_message_text(f"⚙️ إدارة الحساب: `{phone}`", callback.from_user.id, callback.message.message_id, reply_markup=kb, parse_mode="Markdown")

@dp.callback_query_handler(lambda c: c.data == "back_list")
async def back_list(callback: types.CallbackQuery):
    kb = InlineKeyboardMarkup(row_width=1)
    for phone in saved_accounts:
        kb.add(InlineKeyboardButton(f"📱 {phone}", callback_data=f"manage:{phone}"))
    await bot.edit_message_text("📂 قائمة حساباتك الجاهزة للسحب:", callback.from_user.id, callback.message.message_id, reply_markup=kb)

# ══════════════════════════════════════
#        عملية التفعيل (الخطوات المنظمة)
# ══════════════════════════════════════

@dp.message_handler(lambda m: m.text == "➕ تفعيل رقم جديد")
async def start_activation(message: types.Message):
    user_sessions[message.from_user.id] = {'step': 'get_phone'}
    await message.answer("📞 أرسل رقم الهاتف الدولي (مثال: +2010...):")

@dp.message_handler(lambda m: user_sessions.get(m.from_user.id, {}).get('step') == 'get_phone')
async def step_phone(message: types.Message):
    phone = message.text.strip()
    client = TelegramClient(f"sessions/{phone}", API_ID, API_HASH)
    await client.connect()
    user_sessions[message.from_user.id] = {'client': client, 'phone': phone, 'step': 'get_email'}
    await message.answer("📧 أرسل بريد التسجيل (Gmail) اللي ظهرلك في تليجرام:")

@dp.message_handler(lambda m: user_sessions.get(m.from_user.id, {}).get('step') == 'get_email')
async def step_email(message: types.Message):
    uid = message.from_user.id
    email = message.text.strip()
    data = user_sessions[uid]
    try:
        sent_code = await data['client'].send_code_request(data['phone'], email=email)
        user_sessions[uid].update({'hash': sent_code.phone_code_hash, 'step': 'get_email_code'})
        await message.answer(f"📩 تم إرسال الكود إلى `{email}`\nأرسل الكود المكون من 5 أرقام هنا:", parse_mode="Markdown")
    except Exception as e: await message.answer(f"❌ خطأ: {e}")

@dp.message_handler(lambda m: user_sessions.get(m.from_user.id, {}).get('step') == 'get_email_code')
async def step_email_code(message: types.Message):
    uid = message.from_user.id
    user_sessions[uid].update({'email_code': message.text.strip(), 'step': 'timer'})
    msg = await message.answer("⏳ كود البريد صحيح! انتظر 90 ثانية لفتح بوابة الـ SMS...")
    
    for i in range(90, 0, -15):
        await asyncio.sleep(15)
        try: await msg.edit_text(f"⏳ متبقي {i} ثانية لطلب كود SMS...")
        except: pass
    
    kb = InlineKeyboardMarkup().add(InlineKeyboardButton("📲 طلب كود SMS الآن", callback_data="req_sms"))
    await message.answer("✅ البوابة جاهزة! اضغط لطلب الكود على الموبايل:", reply_markup=kb)

@dp.callback_query_handler(lambda c: c.data == "req_sms")
async def step_req_sms(callback: types.CallbackQuery):
    uid = callback.from_user.id
    data = user_sessions[uid]
    try:
        new_code = await data['client'].send_code_request(data['phone'], force_sms=True)
        user_sessions[uid].update({'hash': new_code.phone_code_hash, 'step': 'get_sms_code'})
        await bot.send_message(uid, "💬 كود الـ SMS في الطريق.. أرسله هنا فور وصوله:")
    except Exception as e: await bot.send_message(uid, f"❌ فشل طلب SMS: {e}")

@dp.message_handler(lambda m: user_sessions.get(m.from_user.id, {}).get('step') == 'get_sms_code')
async def step_name(message: types.Message):
    user_sessions[message.from_user.id].update({'sms_code': message.text.strip(), 'step': 'get_name'})
    await message.answer("📛 أرسل الاسم الذي تريده للحساب (الأول والأخير):")

@dp.message_handler(lambda m: user_sessions.get(m.from_user.id, {}).get('step') == 'get_name')
async def step_finalize(message: types.Message):
    uid = message.from_user.id
    data = user_sessions[uid]
    name = message.text.strip().split(" ", 1)
    f_name = name[0]
    l_name = name[1] if len(name) > 1 else ""
    
    try:
        await data['client'].sign_up(data['phone'], data['sms_code'], first_name=f_name, last_name=l_name)
    except:
        try: await data['client'].sign_in(data['phone'], data['sms_code'], phone_code_hash=data['hash'])
        except Exception as e: return await message.answer(f"❌ خطأ نهائي: {e}")

    if data['phone'] not in saved_accounts: saved_accounts.append(data['phone'])
    await message.answer(f"✅ تم تفعيل الحساب `{data['phone']}` بنجاح!\nموجود الآن في قائمة 'حساباتي'.", parse_mode="Markdown")
    user_sessions[uid]['step'] = None

# ══════════════════════════════════════
#           منطق سحب الرسائل
# ══════════════════════════════════════

@dp.callback_query_handler(lambda c: c.data.startswith('pull:'))
async def pull_logic(callback: types.CallbackQuery):
    phone = callback.data.split(":")[1]
    client = TelegramClient(f"sessions/{phone}", API_ID, API_HASH)
    try:
        await client.connect()
        async for msg in client.iter_messages(777000, limit=1):
            kb = InlineKeyboardMarkup().add(
                InlineKeyboardButton("🔄 تحديث الكود", callback_data=f"pull:{phone}"),
                InlineKeyboardButton("⬅️ رجوع", callback_data=f"manage:{phone}")
            )
            await bot.send_message(callback.from_user.id, f"🆕 **آخر رسالة للحساب** `{phone}`:\n\n`{msg.text}`", parse_mode="Markdown", reply_markup=kb)
        await client.disconnect()
    except Exception as e: await bot.send_message(callback.from_user.id, f"❌ خطأ في السحب: {e}")

if __name__ == '__main__':
    executor.start_polling(dp, skip_updates=True)
