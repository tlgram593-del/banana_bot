# banana_bot
import sqlite3
import random
import logging

from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup, ReplyKeyboardMarkup, KeyboardButton
from telegram.ext import ApplicationBuilder, CommandHandler, CallbackQueryHandler, MessageHandler, ContextTypes, filters

TOKEN = "8911544076:AAG5pVCCNF4-HjExRNUEGKavYQdqyoTkmVs"

# ================= LOG =================
logging.basicConfig(
    format="%(asctime)s - %(levelname)s - %(message)s",
    level=logging.INFO
)

# ================= DB =================
conn = sqlite3.connect("bot.db", check_same_thread=False)
cur = conn.cursor()

cur.execute("""
CREATE TABLE IF NOT EXISTS users (
    user_id INTEGER PRIMARY KEY,
    referrals INTEGER DEFAULT 0,
    wallet INTEGER DEFAULT 0,
    captcha_done INTEGER DEFAULT 0,
    phone TEXT
)
""")
conn.commit()

captcha_ans = {}

# ================= CAPTCHA =================
def make_captcha():
    a, b = random.randint(1, 9), random.randint(1, 9)
    return f"{a} + {b}", a + b

# ================= PANEL =================
panel = InlineKeyboardMarkup([
    [InlineKeyboardButton("🎁 سرور رایگان", callback_data="server")],
    [
        InlineKeyboardButton("🛒 خرید", callback_data="buy"),
        InlineKeyboardButton("📞 پشتیبانی", callback_data="support")
    ],
    [
        InlineKeyboardButton("👥 ریفرال", callback_data="ref"),
        InlineKeyboardButton("💰 کیف پول", callback_data="wallet")
    ]
])

# ================= START =================
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid = update.effective_user.id

    cur.execute("INSERT OR IGNORE INTO users (user_id) VALUES (?)", (uid,))
    conn.commit()

    cur.execute("SELECT captcha_done FROM users WHERE user_id=?", (uid,))
    row = cur.fetchone()

    if row and row[0] == 1:
        await update.message.reply_text("✅ خوش اومدی", reply_markup=panel)
        return

    q, ans = make_captcha()
    captcha_ans[uid] = ans

    await update.message.reply_text(f"🔐 کپچا:\n{q} = ?")

# ================= CAPTCHA =================
async def captcha_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid = update.effective_user.id

    if uid not in captcha_ans:
        return

    try:
        if int(update.message.text) == captcha_ans[uid]:
            cur.execute("UPDATE users SET captcha_done=1 WHERE user_id=?", (uid,))
            conn.commit()

            del captcha_ans[uid]

            kb = ReplyKeyboardMarkup(
                [[KeyboardButton("📱 ارسال شماره", request_contact=True)]],
                resize_keyboard=True,
                one_time_keyboard=True
            )

            await update.message.reply_text("✅ درست بود، شماره رو بفرست", reply_markup=kb)
        else:
            await update.message.reply_text("❌ اشتباه")
    except:
        pass

# ================= PHONE =================
async def phone_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid = update.effective_user.id
    phone = update.message.contact.phone_number

    cur.execute("UPDATE users SET phone=? WHERE user_id=?", (phone, uid))
    conn.commit()

    await update.message.reply_text("✅ ثبت شد")
    await update.message.reply_text("👇 پنل:", reply_markup=panel)

# ================= BUTTONS =================
async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    q = update.callback_query
    uid = q.from_user.id
    await q.answer()

    cur.execute("SELECT referrals, wallet FROM users WHERE user_id=?", (uid,))
    row = cur.fetchone()

    refs = row[0]
    wallet = row[1]

    if q.data == "server":
        if refs >= 5:
            await q.message.reply_text("🎉 کانفینگ اختصاصی برات فعال شد")
        else:
            await q.message.reply_text(f"❌ باید 5 نفر دعوت کنی\n👥 {refs}/5")

    elif q.data == "buy":
        await q.message.reply_text(
            "🛒 خرید:\n\n"
            "20GB = 200T\n"
            "50GB = 400T\n"
            "100GB = 650T\n\n"
            "برای خرید به پشتیبانی پیام بده"
        )

    elif q.data == "support":
        await q.message.reply_text("📞 پشتیبانی: @support")

    elif q.data == "ref":
        link = f"https://t.me/{context.bot.username}?start={uid}"
        await q.message.reply_text(f"👥 ریفرال:\n{link}")

    elif q.data == "wallet":
        await q.message.reply_text(f"💰 کیف پول: {wallet}T")

# ================= ERROR HANDLER =================
async def error_handler(update, context):
    print("❌ ERROR:", context.error)

# ================= RUN (FIXED FOR PYDROID) =================
app = ApplicationBuilder().token(TOKEN).build()

app.add_handler(CommandHandler("start", start))
app.add_handler(CallbackQueryHandler(button_handler))
app.add_handler(MessageHandler(filters.CONTACT, phone_handler))
app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, captcha_handler))

app.add_error_handler(error_handler)

print("BOT RUNNING...")

# 🔥 سازگار با همه نسخه‌ها
try:
    app.run_polling(drop_pending_updates=True)
except TypeError:
    app.run_polling()
