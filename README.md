# my-E-book-store-
"""
Python E-book Telegram Bot — v2 (ምናሌ + በእጅ ክፍያ ማረጋገጫ)
pyTelegramBotAPI (telebot) ይጠቀማል።

አጠቃቀም፦
1) ከታች BOT_TOKEN እና ADMIN_ID ይቀይሩ
2) CATALOG ውስጥ ያለውን የመጽሐፍ ዋጋ/ፋይል ይቀይሩ ወይም አዲስ መጽሐፍ ይጨምሩ
3) BANK_INFO ውስጥ ያለውን የባንክ/Telebirr መረጃ ይቀይሩ
4) የመጽሐፍ ፋይሎችን ከ bot.py ጋር በተመሳሳይ ፎልደር ያስቀምጡ
5) python bot.py

⚠️ በአንድ ጊዜ ከአንድ በላይ bot.py ፈጽሞ አያስኪዱ (Error 409 ያስከትላል)።
   ከመጀመርዎ በፊት: pkill -f bot.py  ብለው ነባር ሂደቶችን ያጥፉ።
"""

import json
import os
import telebot
from telebot import types

# ============ ማዋቀሪያ (CONFIG) — እነዚህን ይቀይሩ ============
BOT_TOKEN = "PASTE_YOUR_BOT_TOKEN_HERE"      # ከ @BotFather የሚያገኙት
ADMIN_ID = 123456789                          # የእርስዎ Telegram ID (ከ @userinfobot ያግኙ)

STORE_NAME = "🛍️ My E-book Store"

# 📚 የመጽሐፍት ዝርዝር — ዋጋ ለመጨመር ከታች "price" ያለውን ቁጥር ብቻ ይቀይሩ።
# አዲስ መጽሐፍ ለመጨመር: አዲስ key (ለምሳሌ "forex_ebook") ጨምረው ተመሳሳይ ቅርጽ ይከተሉ።
CATALOG = {
    "python_ebook": {
        "title": "🐍 የPython ፕሮግራሚንግ ሙሉ መመሪያ",
        "description": (
            "📖 ከጀማሪ እስከ ከፍተኛ ደረጃ — 16 ምዕራፍ\n"
            "✅ ተግባራዊ ፕሮጀክቶች እና ምሳሌዎች\n"
            "🗣️ ቋንቋ፦ አማርኛ + English (code examples)"
        ),
        "price": 150,                     # ← ዋጋውን ለመቀየር ይህን ቁጥር ብቻ ይቀይሩ
        "file": "Python_Ebook_COMPLETE_16Chapters.docx",   # ⚠️ ከbot.py ጋር በተመሳሳይ ፎልደር ይኑር
    },
    # ተጨማሪ መጽሐፍ ምሳሌ (ኮሜንቱን አጥፍተው መጠቀም ይችላሉ)፦
    # "forex_ebook": {
    #     "title": "💹 Forex Trading Psychology",
    #     "description": "13 ምዕራፍ — trading psychology እና strategies",
    #     "price": 200,
    #     "file": "Forex_Ebook.docx",
    # },
}

COURSES_MSG = "🎓 Courses\n\nበቅርቡ ይመጣል — ይከታተሉን!"
FREE_LESSONS_MSG = "📖 Free Lessons\n\nነፃ ትምህርቶች በቅርቡ እዚህ ይታከላሉ።"

BANK_INFO_TEMPLATE = """🏦 የክፍያ መረጃ

ባንክ፦ Commercial Bank of Ethiopia (CBE)
የሂሳብ ስም፦ Yacob ...
የሂሳብ ቁጥር፦ 1000xxxxxxxx

ወይም Telebirr፦ 09xxxxxxxx

💵 የሚከፈለው መጠን፦ {price} ብር
📘 ለ፦ {title}

━━━━━━━━━━━━━━━
ክፍያ ከፈጸሙ በኋላ የክፍያ ማረጋገጫ screenshot (ፎቶ) እዚሁ chat ላይ ይላኩልን። ማረጋገጥ እንደተደረገ ወዲያውኑ መጽሐፉ ይላክልዎታል።"""
# =======================================================

BASE_DIR = os.path.dirname(os.path.abspath(__file__))
ORDERS_FILE = os.path.join(BASE_DIR, "orders.json")


def load_orders():
    if not os.path.exists(ORDERS_FILE):
        return {}
    with open(ORDERS_FILE, "r", encoding="utf-8") as f:
        return json.load(f)


def save_orders(data):
    with open(ORDERS_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)


bot = telebot.TeleBot(BOT_TOKEN)
waiting_for_proof = {}  # user_id -> book_id (ግዛ የተጫኑ እና screenshot እየጠበቅን ያለንባቸው)


def main_menu():
    markup = types.InlineKeyboardMarkup()
    markup.add(types.InlineKeyboardButton("📚 E-books", callback_data="menu_ebooks"))
    markup.add(types.InlineKeyboardButton("🎓 Courses", callback_data="menu_courses"))
    markup.add(types.InlineKeyboardButton("📖 Free Lessons", callback_data="menu_lessons"))
    return markup


def ebook_list_menu():
    markup = types.InlineKeyboardMarkup()
    for book_id, info in CATALOG.items():
        markup.add(types.InlineKeyboardButton(info["title"], callback_data=f"view_{book_id}"))
    markup.add(types.InlineKeyboardButton("⬅️ ተመለስ", callback_data="menu_main"))
    return markup


def book_detail_markup(book_id):
    markup = types.InlineKeyboardMarkup()
    markup.add(types.InlineKeyboardButton("🛒 ግዛ", callback_data=f"buy_{book_id}"))
    markup.add(types.InlineKeyboardButton("⬅️ ተመለስ", callback_data="menu_ebooks"))
    return markup


@bot.message_handler(commands=["start"])
def start(message):
    text = f"👋 እንኳን ወደ {STORE_NAME} በደህና መጡ!\n\nምን ይፈልጋሉ?"
    bot.send_message(message.chat.id, text, reply_markup=main_menu())


@bot.callback_query_handler(func=lambda call: call.data == "menu_main")
def menu_main(call):
    bot.edit_message_text(
        f"👋 እንኳን ወደ {STORE_NAME} በደህና መጡ!\n\nምን ይፈልጋሉ?",
        call.message.chat.id, call.message.message_id, reply_markup=main_menu()
    )
    bot.answer_callback_query(call.id)


@bot.callback_query_handler(func=lambda call: call.data == "menu_ebooks")
def menu_ebooks(call):
    bot.edit_message_text(
        "📚 E-books\n\nየፈለጉትን መጽሐፍ ይምረጡ፦",
        call.message.chat.id, call.message.message_id, reply_markup=ebook_list_menu()
    )
    bot.answer_callback_query(call.id)


@bot.callback_query_handler(func=lambda call: call.data == "menu_courses")
def menu_courses(call):
    bot.answer_callback_query(call.id)
    bot.send_message(call.message.chat.id, COURSES_MSG, reply_markup=main_menu())


@bot.callback_query_handler(func=lambda call: call.data == "menu_lessons")
def menu_lessons(call):
    bot.answer_callback_query(call.id)
    bot.send_message(call.message.chat.id, FREE_LESSONS_MSG, reply_markup=main_menu())


@bot.callback_query_handler(func=lambda call: call.data.startswith("view_"))
def view_book(call):
    book_id = call.data.split("_", 1)[1]
    info = CATALOG.get(book_id)
    if not info:
        bot.answer_callback_query(call.id, "አልተገኘም።")
        return
    text = f"{info['title']}\n\n{info['description']}\n\n💵 ዋጋ፦ {info['price']} ብር"
    bot.edit_message_text(text, call.message.chat.id, call.message.message_id, reply_markup=book_detail_markup(book_id))
    bot.answer_callback_query(call.id)


@bot.callback_query_handler(func=lambda call: call.data.startswith("buy_"))
def buy(call):
    book_id = call.data.split("_", 1)[1]
    info = CATALOG.get(book_id)
    if not info:
        bot.answer_callback_query(call.id, "አልተገኘም።")
        return
    waiting_for_proof[call.from_user.id] = book_id
    bank_msg = BANK_INFO_TEMPLATE.format(price=info["price"], title=info["title"])
    bot.send_message(call.message.chat.id, bank_msg)
    bot.answer_callback_query(call.id)


@bot.message_handler(content_types=["photo"])
def receive_proof(message):
    user_id = message.from_user.id
    book_id = waiting_for_proof.get(user_id)

    if not book_id:
        bot.reply_to(message, "ግዢ ለመጀመር እባክዎ /start ብለው ይላኩ፣ መጽሐፍ ይምረጡ እና 🛒 ግዛ የሚለውን ይጫኑ።")
        return

    info = CATALOG.get(book_id, {})
    orders = load_orders()
    order_key = f"{user_id}_{book_id}"
    orders[order_key] = {
        "user_id": user_id,
        "book_id": book_id,
        "username": message.from_user.username or "",
        "first_name": message.from_user.first_name or "",
        "status": "pending",
    }
    save_orders(orders)
    waiting_for_proof.pop(user_id, None)

    bot.reply_to(message, "✅ ደረሰን! ክፍያዎ በአስተዳዳሪ እየተረጋገጠ ነው፣ እባክዎ ትንሽ ይጠብቁ...")

    markup = types.InlineKeyboardMarkup()
    markup.add(
        types.InlineKeyboardButton("✅ አጽድቅ", callback_data=f"approve_{order_key}"),
        types.InlineKeyboardButton("❌ አትቀበል", callback_data=f"reject_{order_key}"),
    )
    caption = (
        f"📩 አዲስ ትዕዛዝ\n"
        f"📘 {info.get('title', book_id)}\n"
        f"👤 {message.from_user.first_name} (@{message.from_user.username})\n"
        f"🆔 {user_id}"
    )
    bot.send_photo(ADMIN_ID, message.photo[-1].file_id, caption=caption, reply_markup=markup)


@bot.callback_query_handler(func=lambda call: call.data.startswith("approve_"))
def approve(call):
    if call.from_user.id != ADMIN_ID:
        return
    order_key = call.data.split("_", 1)[1]
    user_id_str, book_id = order_key.split("_", 1)
    user_id = int(user_id_str)

    orders = load_orders()
    if order_key in orders:
        orders[order_key]["status"] = "approved"
        save_orders(orders)

    info = CATALOG.get(book_id)
    if not info:
        bot.send_message(ADMIN_ID, f"⚠️ CATALOG ውስጥ {book_id} አልተገኘም።")
        bot.answer_callback_query(call.id)
        return

    book_path = os.path.join(BASE_DIR, info["file"])
    try:
        with open(book_path, "rb") as f:
            bot.send_document(
                user_id, f,
                caption=f"🎉 ክፍያዎ ተረጋግጧል! {info['title']} ከታች ነው። መልካም ንባብ! 🐍"
            )
        bot.send_message(ADMIN_ID, f"✅ ለ {user_id} ({info['title']}) ተልኳል።")
    except FileNotFoundError:
        bot.send_message(ADMIN_ID, f"⚠️ የመጽሐፍ ፋይል አልተገኘም: {book_path}")
    bot.answer_callback_query(call.id, "ተልኳል ✅")


@bot.callback_query_handler(func=lambda call: call.data.startswith("reject_"))
def reject(call):
    if call.from_user.id != ADMIN_ID:
        return
    order_key = call.data.split("_", 1)[1]
    user_id_str, book_id = order_key.split("_", 1)
    user_id = int(user_id_str)

    orders = load_orders()
    if order_key in orders:
        orders[order_key]["status"] = "rejected"
        save_orders(orders)

    bot.send_message(
        user_id,
        "❌ የክፍያ ማረጋገጫዎ አልተረጋገጠም። እባክዎ ትክክለኛ screenshot እንደገና ይላኩ ወይም አስተዳዳሪን በቀጥታ ያግኙ።"
    )
    bot.answer_callback_query(call.id, "ተቀባይነት አላገኘም")


@bot.message_handler(commands=["orders"])
def list_orders(message):
    if message.from_user.id != ADMIN_ID:
        return
    orders = load_orders()
    if not orders:
        bot.reply_to(message, "ምንም ትዕዛዝ የለም።")
        return
    lines = [
        f"🆔 {info['user_id']} — {info['status']} — {info.get('book_id')} "
        f"({info.get('first_name', '')}, @{info.get('username', '')})"
        for info in orders.values()
    ]
    bot.reply_to(message, "\n".join(lines))


@bot.message_handler(commands=["help"])
def help_cmd(message):
    if message.from_user.id == ADMIN_ID:
        bot.reply_to(
            message,
            "🔧 የአስተዳዳሪ ትዕዛዞች፦\n"
            "/orders — ሁሉንም ትዕዛዞች ዘርዝር\n"
            "ደንበኛ screenshot ሲልክ ✅/❌ ቁልፎች ይመጣሉ።\n\n"
            "💰 ዋጋ ለመቀየር: bot.py ውስጥ CATALOG ውስጥ ያለውን 'price' ቁጥር ይቀይሩ እና bot.py ን እንደገና ያስኪዱ።"
        )
    else:
        bot.reply_to(message, "/start ብለው ይጀምሩ እና 📚 E-books የሚለውን ይጫኑ።")


if __name__ == "__main__":
    print("🤖 Bot is running... (Ctrl+C to stop)")
    bot.infinity_polling(skip_pending=True
    