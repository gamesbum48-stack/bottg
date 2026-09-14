import asyncio
import logging
import os
import re
import sqlite3
from datetime import datetime, timedelta
from aiogram import Bot, Dispatcher, F, types
from aiogram.enums import ParseMode
from aiogram.filters import Command

# Отримуємо токен зі змінних оточення (або підставиться твій, якщо змінна не задана)
BOT_TOKEN = os.getenv("BOT_TOKEN", "8959720819:AAGs_z79er5bryEJTiIOUgYhi-EwOve1YjI")
MY_TELEGRAM_ID = 882424834

logging.basicConfig(level=logging.INFO)
bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()

# --- БАЗА ДАНИХ (SQLite) ---
def init_db():
    conn = sqlite3.connect("ipad_pro_biz.db")
    cursor = conn.cursor()
    
    # 1. Товари на складі
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS inventory (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            model TEXT,
            buy_price REAL,
            repair_cost REAL DEFAULT 0,
            date_added TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    
    # 2. Історія продажів iPad
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS sales (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            model TEXT,
            total_cost REAL,
            sell_price REAL,
            profit REAL,
            roi REAL,
            date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    
    # 3. Інвестиції та особисті витрати
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS external_finances (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            category_type TEXT,
            name TEXT,
            amount REAL,
            date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    
    conn.commit()
    conn.close()

# --- ФУНКЦІЇ ДЛЯ РОБОТИ З БАЗОЮ ---
def add_ipad(model: str, buy_price: float):
    conn = sqlite3.connect("ipad_pro_biz.db")
    cursor = conn.cursor()
    cursor.execute("INSERT INTO inventory (model, buy_price, repair_cost) VALUES (?, ?, 0)", (model, buy_price))
    conn.commit()
    conn.close()

def add_repair(ipad_id: int, repair_cost: float):
    conn = sqlite3.connect("ipad_pro_biz.db")
    cursor = conn.cursor()
    cursor.execute("UPDATE inventory SET repair_cost = repair_cost + ? WHERE id = ?", (repair_cost, ipad_id))
    rows = cursor.rowcount
    conn.commit()
    conn.close()
    return rows > 0

def sell_ipad(ipad_id: int, sell_price: float):
    conn = sqlite3.connect("ipad_pro_biz.db")
    cursor = conn.cursor()
    
    cursor.execute("SELECT model, buy_price, repair_cost FROM inventory WHERE id = ?", (ipad_id,))
    item = cursor.fetchone()
    
    if not item:
        conn.close()
        return None

    model, buy_price, repair_cost = item
    total_cost = buy_price + repair_cost
    profit = sell_price - total_cost
    roi = (profit / total_cost * 100) if total_cost > 0 else 0.0

    cursor.execute("INSERT INTO sales (model, total_cost, sell_price, profit, roi) VALUES (?, ?, ?, ?, ?)",
                   (model, total_cost, sell_price, profit, roi))
    cursor.execute("DELETE FROM inventory WHERE id = ?", (ipad_id,))
    
    conn.commit()
    conn.close()
    return {"model": model, "total_cost": total_cost, "profit": profit, "roi": roi}

def add_external_finance(cat_type: str, name: str, amount: float):
    conn = sqlite3.connect("ipad_pro_biz.db")
    cursor = conn.cursor()
    cursor.execute("INSERT INTO external_finances (category_type, name, amount) VALUES (?, ?, ?)",
                   (cat_type, name, amount))
    conn.commit()
    conn.close()

def get_inventory():
    conn = sqlite3.connect("ipad_pro_biz.db")
    cursor = conn.cursor()
    cursor.execute("SELECT id, model, buy_price, repair_cost, date_added FROM inventory")
    rows = cursor.fetchall()
    conn.close()
    return rows

def get_analytics():
    conn = sqlite3.connect("ipad_pro_biz.db")
    cursor = conn.cursor()
    
    cursor.execute("SELECT SUM(profit) FROM sales")
    total_profit = cursor.fetchone()[0] or 0.0
    
    now = datetime.now()
    first_day_of_month = datetime(now.year, now.month, 1).strftime("%Y-%m-%d %H:%M:%S")
    cursor.execute("SELECT SUM(profit) FROM sales WHERE date >= ?", (first_day_of_month,))
    month_profit = cursor.fetchone()[0] or 0.0

    week_ago = (now - timedelta(days=7)).strftime("%Y-%m-%d %H:%M:%S")
    cursor.execute("SELECT SUM(profit) FROM sales WHERE date >= ?", (week_ago,))
    week_profit = cursor.fetchone()[0] or 0.0

    cursor.execute("SELECT SUM(buy_price + repair_cost) FROM inventory")
    invested_in_stock = cursor.fetchone()[0] or 0.0

    cursor.execute("SELECT SUM(amount) FROM external_finances WHERE category_type = 'investment'")
    total_invested = cursor.fetchone()[0] or 0.0

    cursor.execute("SELECT SUM(amount) FROM external_finances WHERE category_type = 'expense'")
    total_expenses = cursor.fetchone()[0] or 0.0

    conn.close()
    
    free_balance = total_profit - total_invested - total_expenses
    
    return {
        "total_profit": total_profit,
        "month_profit": month_profit,
        "week_profit": week_profit,
        "invested_in_stock": invested_in_stock,
        "total_invested": total_invested,
        "total_expenses": total_expenses,
        "free_balance": free_balance
    }

def get_investments_list():
    conn = sqlite3.connect("ipad_pro_biz.db")
    cursor = conn.cursor()
    cursor.execute("SELECT name, SUM(amount) FROM external_finances WHERE category_type = 'investment' GROUP BY name")
    rows = cursor.fetchall()
    conn.close()
    return rows

# --- ПЕРЕВІРКА ДОСТУПУ ---
@dp.message.outer_middleware()
async def check_user_middleware(handler, event: types.Message, data):
    if event.from_user.id != MY_TELEGRAM_ID:
        await event.answer("⛔️ Це приватний бот.")
        return
    return await handler(event, data)

# --- КОМАНДИ ---
@dp.message(Command("start"))
async def start_cmd(message: types.Message):
    await message.answer(
        "📱 **Особистий фінансовий бот-менеджер (у €)**\n\n"
        "**Команди для iPad:**\n"
        "• `купив iPad Air 4 64GB 180` — додати товар\n"
        "• `ремонт 1 25` — додати 25€ ремонту до ID 1\n"
        "• `продав 1 260` — продати ID 1 за 260€\n"
        "• /stock — перегляд складу\n\n"
        "**Інвестиції та витрати:**\n"
        "• `інвест NVIDIA 100` — зафіксувати інвестицію\n"
        "• `витрата продукти 40` — особиста витрата з прибутку\n\n"
        "**Аналітика:**\n"
        "• /stats — повний звіт\n"
        "• /investments — портфель інвестицій",
        parse_mode=ParseMode.MARKDOWN
    )

@dp.message(Command("stock"))
async def stock_cmd(message: types.Message):
    items = get_inventory()
    if not items:
        await message.answer("📦 На складі зараз немає жодного iPad.")
        return

    text = "📦 **iPad у наявності:**\n\n"
    now = datetime.now()
    
    for ipad_id, model, buy_price, repair_cost, date_added in items:
        total = buy_price + repair_cost
        repair_str = f" (+{repair_cost:.2f}€ ремонт)" if repair_cost > 0 else ""
        
        try:
            added_dt = datetime.strptime(date_added.split(".")[0], "%Y-%m-%d %H:%M:%S")
            days_on_stock = (now - added_dt).days
        except Exception:
            days_on_stock = 0
        
        text += (
            f"🔹 **ID: {ipad_id}** | **{model}**\n"
            f"   Собівартість: **{total:.2f}€** (купівля {buy_price:.2f}€{repair_str})\n"
            f"   ⏳ На складі: **{days_on_stock} днів**\n\n"
        )
    
    text += "💡 *Щоб продати: `продав ID ціна` (наприклад: `продав 1 250`)*"
    await message.answer(text, parse_mode=ParseMode.MARKDOWN)

@dp.message(Command("stats"))
async def stats_cmd(message: types.Message):
    data = get_analytics()
    
    await message.answer(
        f"📊 **Повна фінансова аналітика:**\n\n"
        f"📈 **Прибуток від iPad:**\n"
        f"├ За весь час: **+{data['total_profit']:.2f} €**\n"
        f"├ За цей місяць: **+{data['month_profit']:.2f} €**\n"
        f"└ За 7 днів: **+{data['week_profit']:.2f} €**\n\n"
        f"💼 **Розподіл капіталу:**\n"
        f"├ 📦 У товарі на складі: **{data['invested_in_stock']:.2f} €**\n"
        f"├ 🚀 Інвестовано далі: **{data['total_invested']:.2f} €**\n"
        f"└ 💸 Інші витрати: **{data['total_expenses']:.2f} €**\n\n"
        f"💰 **Вільний чистий баланс (на руках):**\n"
        f"👉 **{data['free_balance']:.2f} €**",
        parse_mode=ParseMode.MARKDOWN
    )

@dp.message(Command("investments"))
async def investments_cmd(message: types.Message):
    invs = get_investments_list()
    if not invs:
        await message.answer("📈 Інвестицій поки немає.")
        return
    
    text = "💼 **Твій інвестиційний портфель:**\n\n"
    for name, total_amt in invs:
        text += f"▪️ **{name}**: {total_amt:.2f} €\n"
    await message.answer(text, parse_mode=ParseMode.MARKDOWN)

# --- ОБРОБКА ТЕКСТУ ---
@dp.message(F.text)
async def process_text(message: types.Message):
    text = message.text.strip()
    text_lower = text.lower()

    if text_lower.startswith("купив") or text_lower.startswith("+"):
        match = re.search(r"(?:купив|\+)\s*(.*?)\s*(\d+(?:\.\d+)?)\s*€?$", text, re.IGNORECASE)
        if match:
            model = match.group(1).strip()
            price = float(match.group(2))
            if not model: 
                model = "iPad"
            add_ipad(model, price)
            await message.answer(f"✅ Додано в наявність: **{model}** за **{price:.2f}€**.\nПерегляд складу: /stock", parse_mode=ParseMode.MARKDOWN)
            return

    if text_lower.startswith("ремонт"):
        match = re.search(r"ремонт\s+(\d+)\s+(\d+(?:\.\d+)?)", text_lower)
        if match:
            ipad_id = int(match.group(1))
            cost = float(match.group(2))
            if add_repair(ipad_id, cost):
                await message.answer(f"🔧 До ID {ipad_id} додано ремонт на **{cost:.2f}€**.", parse_mode=ParseMode.MARKDOWN)
            else:
                await message.answer(f"❌ iPad з ID {ipad_id} не знайдено.")
            return

    if text_lower.startswith("продав"):
        match = re.search(r"продав\s+(\d+)\s+(\d+(?:\.\d+)?)", text_lower)
        if match:
            ipad_id = int(match.group(1))
            sell_price = float(match.group(2))
            res = sell_ipad(ipad_id, sell_price)
            if res:
                symbol = "🟢" if res['profit'] >= 0 else "🔴"
                await message.answer(
                    f"🎉 **{res['model']} ПРОДАНО!**\n\n"
                    f"🏷 Ціна продажу: {sell_price:.2f}€\n"
                    f"📉 Собівартість: {res['total_cost']:.2f}€\n"
                    f"{symbol} **Чистий прибуток:** {res['profit']:.2f}€\n"
                    f"📊 **Маржинальність (ROI):** {res['roi']:.1f}%",
                    parse_mode=ParseMode.MARKDOWN
                )
            else:
                await message.answer(f"❌ iPad з ID {ipad_id} не знайдено на складі.")
            return

    if text_lower.startswith("інвест") or text_lower.startswith("інвестиція"):
        match = re.search(r"(?:інвест|інвестиція)\s+(.*?)\s+(\d+(?:\.\d+)?)", text, re.IGNORECASE)
        if match:
            asset_name = match.group(1).strip()
            amount = float(match.group(2))
            add_external_finance("investment", asset_name, amount)
            await message.answer(f"🚀 Записано інвестицію: **{amount:.2f}€** в **{asset_name}**.", parse_mode=ParseMode.MARKDOWN)
            return

    if text_lower.startswith("витрата") or text_lower.startswith("витратив"):
        match = re.search(r"(?:витрата|витратив)\s+(.*?)\s+(\d+(?:\.\d+)?)", text, re.IGNORECASE)
        if match:
            exp_name = match.group(1).strip()
            amount = float(match.group(2))
            add_external_finance("expense", exp_name, amount)
            await message.answer(f"💸 Записано витрату: **{amount:.2f}€** ({exp_name}).", parse_mode=ParseMode.MARKDOWN)
            return

    await message.answer(
        "Не зрозумів команду.\n\n"
        "**Приклади:**\n"
        "• `купив iPad Air 4 180`\n"
        "• `ремонт 1 20`\n"
        "• `продав 1 250`\n"
        "• `інвест NVIDIA 100`\n"
        "• `витрата продукты 30`\n"
        "• /stock чи /stats",
        parse_mode=ParseMode.MARKDOWN
    )

async def main():
    init_db()
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
