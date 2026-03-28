import sqlite3
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, ContextTypes

# --- Database setup ---
conn = sqlite3.connect("expenses.db")
cursor = conn.cursor()
cursor.execute("""
CREATE TABLE IF NOT EXISTS transactions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER,
    amount REAL,
    category TEXT,
    date TEXT DEFAULT (datetime('now','localtime'))
)
""")
conn.commit()

# --- Bot Commands ---
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "Welcome! I'm your Finance Tracker Bot.\n"
        "Use /add <amount> <category> to log a transaction.\n"
        "Use /summary to see your monthly summary."
    )

async def add(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user_id = update.message.from_user.id
        amount = float(context.args[0])
        category = context.args[1]
        cursor.execute(
            "INSERT INTO transactions (user_id, amount, category) VALUES (?, ?, ?)",
            (user_id, amount, category)
        )
        conn.commit()
        await update.message.reply_text(f"Added {amount} to {category}.")
    except (IndexError, ValueError):
        await update.message.reply_text("Usage: /add <amount> <category>")

async def summary(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.message.from_user.id
    cursor.execute(
        "SELECT category, SUM(amount) FROM transactions WHERE user_id = ? GROUP BY category",
        (user_id,)
    )
    rows = cursor.fetchall()
    if not rows:
        await update.message.reply_text("No transactions found.")
        return
    msg = "📊 Your spending summary:\n"
    for category, total in rows:
        msg += f"{category}: {total}\n"
    await update.message.reply_text(msg)

# --- Main ---
app = ApplicationBuilder().token("YOUR_TELEGRAM_BOT_TOKEN").build()

app.add_handler(CommandHandler("start", start))
app.add_handler(CommandHandler("add", add))
app.add_handler(CommandHandler("summary", summary))

print("Bot is running...")
app.run_polling()
