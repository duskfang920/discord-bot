
def run():
    app.run(host="0.0.0.0", port=8080)

Thread(target=run).start()

TOKEN = os.getenv("DISCORD_TOKEN")

intents = discord.Intents.default()
intents.message_content = True
bot = commands.Bot(command_prefix="!", intents=intents)

# Kết nối SQLite
conn = sqlite3.connect("casino.db")
cursor = conn.cursor()
cursor.execute(
    """CREATE TABLE IF NOT EXISTS users (
        id TEXT PRIMARY KEY,
        money INTEGER DEFAULT 50000,
        last_daily TEXT
    )"""
)
conn.commit()

# Hàm lấy số tiền
def get_money(user_id):
    cursor.execute("SELECT money FROM users WHERE id=?", (user_id,))
    result = cursor.fetchone()
    return result[0] if result else 50000

# Hàm cập nhật số tiền
def update_money(user_id, amount):
    cursor.execute("INSERT INTO users (id, money) VALUES (?, ?) ON CONFLICT(id) DO UPDATE SET money=?",
                   (user_id, amount, amount))
    conn.commit()

@bot.event
async def on_ready():
    print(f"✅ Bot {bot.user} đã hoạt động!")

@bot.command(name="daily")
async def daily(ctx):
    user_id = str(ctx.author.id)
    cursor.execute("SELECT last_daily FROM users WHERE id=?", (user_id,))
    last_daily = cursor.fetchone()
    now = datetime.now()

    if last_daily and last_daily[0]:
        last_claimed = datetime.strptime(last_daily[0], "%Y-%m-%d %H:%M:%S")
        if now - last_claimed < timedelta(days=1):
            return await ctx.send(f"⏳ {ctx.author.mention}, bạn đã nhận hôm nay rồi!")

    update_money(user_id, get_money(user_id) + 10000)
    cursor.execute("UPDATE users SET last_daily=? WHERE id=?", (now.strftime("%Y-%m-%d %H:%M:%S"), user_id))
    conn.commit()
    await ctx.send(f"🎁 {ctx.author.mention}, bạn đã nhận **10.000💵** hôm nay!")

@bot.command(name="tiền")
async def money(ctx):
    balance = get_money(str(ctx.author.id))
    await ctx.send(f"{ctx.author.mention} có **{balance:,}💵**")

@bot.command(name="tx")
async def taixiu(ctx, bet: int):
    user_id = str(ctx.author.id)
    balance = get_money(user_id)

    if bet > balance or bet <= 0:
        return await ctx.send(f"❌ {ctx.author.mention}, Số tiền cược không hợp lệ!")

    view = discord.ui.View()

    async def bet_choice(interaction, choice):
        if interaction.user != ctx.author:
            return await interaction.response.send_message("❌ Bạn không phải người chơi!", ephemeral=True)

        dice = [random.randint(1, 6) for _ in range(3)]
        total = sum(dice)
        result = "TÀI" if total >= 11 else "XỈU"
        payout = int(bet * 1.92) if choice.lower() == result.lower() else -bet
        update_money(user_id, balance + payout)

        msg = f"🎲 {dice[0]} + {dice[1]} + {dice[2]} = **{total}** → **{result}**!\n"
        msg += f"✅ Thắng **{payout:,}💵**!" if payout > 0 else f"❌ Thua **{abs(payout):,}💵**!"
        await interaction.response.edit_message(content=msg, view=None)

    button_tai = discord.ui.Button(label="TÀI", style=discord.ButtonStyle.green)
    button_tai.callback = lambda i: bet_choice(i, "TÀI")

    button_xiu = discord.ui.Button(label="XỈU", style=discord.ButtonStyle.red)
    button_xiu.callback = lambda i: bet_choice(i, "XỈU")

    view.add_item(button_tai)
    view.add_item(button_xiu)

    await ctx.send(f"{ctx.author.mention}, chọn Tài hoặc Xỉu!", view=view)

@bot.command(name="bc")
async def baucua(ctx, bet: int):
    user_id = str(ctx.author.id)
    balance = get_money(user_id)

    if bet > balance or bet <= 0:
        return await ctx.send(f"❌ {ctx.author.mention}, Số tiền cược không hợp lệ!")

    view = discord.ui.View()
    animals = ["🐔", "🦀", "🦐", "🐟", "🍐", "🦌"]

    async def choose_animal(interaction, animal):
        if interaction.user != ctx.author:
            return await interaction.response.send_message("❌ Bạn không phải người chơi!", ephemeral=True)

        result = random.choices(animals, k=3)
        payout = bet * result.count(animal) if animal in result else -bet
        update_money(user_id, balance + payout)

        msg = f"🎰 Kết quả: {' '.join(result)}\n"
        msg += f"✅ Thắng **{payout:,}💵**!" if payout > 0 else f"❌ Thua **{abs(payout):,}💵**!"
        await interaction.response.edit_message(content=msg, view=None)

    for animal in animals:
        button = discord.ui.Button(label=animal, style=discord.ButtonStyle.blurple)
        button.callback = lambda i, a=animal: choose_animal(i, a)
        view.add_item(button)

    await ctx.send(f"{ctx.author.mention}, chọn con vật!", view=view)

@bot.command(name="db")
async def danh_bai(ctx, bet: int):
    user_id = str(ctx.author.id)
    balance = get_money(user_id)

    if bet > balance or bet <= 0:
        return await ctx.send(f"❌ {ctx.author.mention}, Số tiền cược không hợp lệ!")

    view = discord.ui.View()
    cards = ["🂠", "🂡", "🂢", "🂣", "🂤", "🂥", "🂦", "🂧", "🂨", "🂩"]

    async def choose_card(interaction, card):
        if interaction.user != ctx.author:
            return await interaction.response.send_message("❌ Bạn không phải người chơi!", ephemeral=True)

        bot_card = random.choice(cards)
        payout = int(bet * 2) if card > bot_card else -bet
        update_money(user_id, balance + payout)

        msg = f"🃏 Bạn chọn: {card} | Bot chọn: {bot_card}\n"
        msg += f"✅ Thắng **{payout:,}💵**!" if payout > 0 else f"❌ Thua **{abs(payout):,}💵**!"
        await interaction.response.edit_message(content=msg, view=None)

    for card in cards:
        button = discord.ui.Button(label=card, style=discord.ButtonStyle.gray)
        button.callback = lambda i, c=card: choose_card(i, c)
        view.add_item(button)

    await ctx.send(f"{ctx.author.mention}, chọn lá bài!", view=view)

@bot.event
async def on_command_error(ctx, error):
    if isinstance(error, commands.CommandNotFound):
        await ctx.send("❌ Lệnh không tồn tại! Gõ `!giúp` để xem danh sách lệnh.")

bot.run(TOKEN)
