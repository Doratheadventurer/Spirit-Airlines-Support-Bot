import discord
from discord.ext import commands, tasks
from discord.ui import View, Button
import os
import datetime
import asyncio

intents = discord.Intents.default()
intents.message_content = True
intents.members = True
intents.guilds = True
intents.dm_messages = True

bot = commands.Bot(command_prefix="!", intents=intents)

SUPPORT_ROLE_NAME = "Support Team"
EXEC_ROLE_NAME = "Executive Board Member"
TICKET_CATEGORY_NAME = "Tickets"
TRANSCRIPT_LOG_CHANNEL_NAME = "ticket-logs"
RATING_LOG_CHANNEL_NAME = "ticket-ratings"
user_ticket_channels = {}  # user_id: channel_id
cooldowns = {}  # user_id: timestamp of last ticket open
rating_messages = {}  # message_id: (user_id, support_id)
user_languages = {}  # user_id: language
unrated_tickets = {}  # user_id: (msg_id, timestamp)

LANGUAGES = {
    "🇺🇸": "English",
    "🇪🇸": "Español",
    "🇫🇷": "Français",
    "🇭🇹": "Kreyòl",
    "🇵🇹": "Português"
}

TRANSLATIONS = {
    "ticket_created": {
        "English": "📬 Your ticket has been created. We'll assist you shortly.",
        "Español": "📬 Su ticket ha sido creado. Le ayudaremos pronto.",
        "Français": "📬 Votre ticket a été créé. Nous vous aiderons sous peu.",
        "Kreyòl": "📬 Tikè ou a kreye. Nou pral ede ou byento.",
        "Português": "📬 Seu ticket foi criado. Iremos ajudá-lo em breve."
    },
    "rate_request": {
        "English": "🔒 Your ticket has been closed. Thank you! Please rate our support.",
        "Español": "🔒 Su ticket ha sido cerrado. ¡Gracias! Por favor, califique nuestro soporte.",
        "Français": "🔒 Votre ticket a été fermé. Merci ! Veuillez évaluer notre support.",
        "Kreyòl": "🔒 Tikè ou fèmen. Mèsi! Tanpri evalye sipò a nou ba ou a.",
        "Português": "🔒 Seu ticket foi fechado. Obrigado! Por favor, avalie nosso suporte."
    },
    "bot_offline": {
        "English": "❗ Support is currently offline. Please message again during support hours.",
        "Español": "❗ El soporte está fuera de línea. Por favor, envíe un mensaje durante el horario de atención.",
        "Français": "❗ Le support est hors ligne. Merci de nous contacter pendant les heures d'ouverture.",
        "Kreyòl": "❗ Sipò pa disponib kounye a. Tanpri ekri nou pandan lè sèvis yo.",
        "Português": "❗ O suporte está offline. Por favor, envie mensagem durante o horário de atendimento."
    },
    "choose_language": "Please select your language / Por favor seleccione su idioma / Veuillez sélectionner votre langue / Tanpri chwazi lang ou / Por favor selecione seu idioma"
}

SUPPORT_HOURS = {
    "weekday": (12, 22),
    "weekend": (14, 20)
}

@bot.event
async def on_ready():
    print(f"Logged in as {bot.user}")
    os.makedirs("transcripts", exist_ok=True)
    daily_open_reminder.start()
    rating_reminder_loop.start()
    activity = discord.Activity(type=discord.ActivityType.listening, name="Your Requests!")
    await bot.change_presence(activity=activity)

@bot.command()
async def reset_language(ctx, user: discord.User):
    if user.id in user_languages:
        del user_languages[user.id]
        await ctx.send(f"✅ Language for {user.mention} has been reset.")
    else:
        await ctx.send(f"{user.mention} has no language set.")

@bot.command()
async def view_tickets(ctx):
    open_tickets = [f"<@{uid}> - <#{cid}>" for uid, cid in user_ticket_channels.items()]
    await ctx.send("📂 **Open Tickets:**\n" + ("\n".join(open_tickets) if open_tickets else "None"))

@bot.command()
async def view_pending_ratings(ctx):
    if unrated_tickets:
        lines = [f"<@{uid}> since <t:{int(ts)}:R>" for uid, (_, ts) in unrated_tickets.items()]
        await ctx.send("⏳ **Pending Ratings:**\n" + "\n".join(lines))
    else:
        await ctx.send("✅ No pending ratings!")

@bot.event
async def on_message(message):
    if message.author.bot:
        return

    if isinstance(message.channel, discord.DMChannel):
        if not is_support_open():
            lang = user_languages.get(message.author.id, "English")
            await message.channel.send(TRANSLATIONS["bot_offline"][lang])
            return

        if message.author.id not in user_ticket_channels:
            await ask_language(message.author)
        else:
            guild = bot.guilds[0]
            channel = guild.get_channel(user_ticket_channels[message.author.id])
            if channel:
                await channel.send(f"**{message.author.name}**: {message.content}")

    elif message.guild and message.channel.id in user_ticket_channels.values():
        user_id = next((uid for uid, cid in user_ticket_channels.items() if cid == message.channel.id), None)
        if user_id:
            user = await bot.fetch_user(user_id)
            try:
                if not message.content.startswith('!'):
                    await user.send(f"**Support:** {message.content}")
            except:
                await message.channel.send("Could not send message to user.")

    await bot.process_commands(message)

async def ask_language(user):
    lang_msg = await user.send(TRANSLATIONS["choose_language"])
    for emoji in LANGUAGES:
        await lang_msg.add_reaction(emoji)

    def check(reaction, u):
        return u == user and str(reaction.emoji) in LANGUAGES

    try:
        reaction, _ = await bot.wait_for("reaction_add", timeout=60.0, check=check)
        user_languages[user.id] = LANGUAGES[str(reaction.emoji)]
    except asyncio.TimeoutError:
        user_languages[user.id] = "English"

    await create_ticket(bot.guilds[0], user)

async def create_ticket(guild, user):
    if user.id in user_ticket_channels:
        return

    support_role = discord.utils.get(guild.roles, name=SUPPORT_ROLE_NAME)
    exec_role = discord.utils.get(guild.roles, name=EXEC_ROLE_NAME)
    category = discord.utils.get(guild.categories, name=TICKET_CATEGORY_NAME)
    if not category:
        category = await guild.create_category(TICKET_CATEGORY_NAME)

    lang = user_languages.get(user.id, "English")

    overwrites = {
        guild.default_role: discord.PermissionOverwrite(read_messages=False),
        user: discord.PermissionOverwrite(read_messages=True, send_messages=True),
        support_role: discord.PermissionOverwrite(read_messages=True, send_messages=True),
        exec_role: discord.PermissionOverwrite(read_messages=True, send_messages=True)
    }

    channel = await guild.create_text_channel(f"ticket-{user.name}", category=category, overwrites=overwrites)
    user_ticket_channels[user.id] = channel.id

    embed = discord.Embed(
        title="Support Ticket",
        description=f"Support will be with you shortly.\nLanguage: {lang}\nClick 🔒 to close.",
        color=discord.Color.blurple()
    )
    await channel.send(embed=embed, view=TicketCloseView(user.id))

    try:
        await user.send(TRANSLATIONS["ticket_created"][lang])
    except:
        pass

class TicketCloseView(View):
    def __init__(self, user_id):
        super().__init__(timeout=None)
        self.add_item(CloseButton(user_id))

class CloseButton(Button):
    def __init__(self, user_id):
        super().__init__(label="Close Ticket", style=discord.ButtonStyle.red, emoji="🔒")
        self.user_id = user_id

    async def callback(self, interaction):
        channel = interaction.channel
        await save_transcript(channel, self.user_id)
        del user_ticket_channels[self.user_id]

        lang = user_languages.get(self.user_id, "English")
        user = await bot.fetch_user(self.user_id)
        msg = await user.send(TRANSLATIONS["rate_request"][lang])
        for emoji in ["1️⃣", "2️⃣", "3️⃣", "4️⃣", "5️⃣"]:
            await msg.add_reaction(emoji)
        rating_messages[msg.id] = (self.user_id, interaction.user.id)
        unrated_tickets[self.user_id] = (msg.id, datetime.datetime.utcnow().timestamp())

        await asyncio.sleep(3)
        await channel.delete()

@bot.event
async def on_raw_reaction_add(payload):
    if payload.user_id == bot.user.id:
        return
    if payload.message_id in rating_messages:
        rating = payload.emoji.name
        if rating in ["1️⃣", "2️⃣", "3️⃣", "4️⃣", "5️⃣"]:
            user_id, support_id = rating_messages[payload.message_id]
            guild = bot.guilds[0]
            rating_channel = discord.utils.get(guild.text_channels, name=RATING_LOG_CHANNEL_NAME)
            if not rating_channel:
                overwrites = {
                    guild.default_role: discord.PermissionOverwrite(read_messages=False),
                    discord.utils.get(guild.roles, name=SUPPORT_ROLE_NAME): discord.PermissionOverwrite(read_messages=True),
                    discord.utils.get(guild.roles, name=EXEC_ROLE_NAME): discord.PermissionOverwrite(read_messages=True)
                }
                rating_channel = await guild.create_text_channel(RATING_LOG_CHANNEL_NAME, overwrites=overwrites)

            await rating_channel.send(
                f"📊 **Rating:** {rating}\n👤 From: <@{user_id}>\n🛠️ Handled by: <@{support_id}>"
            )
            del unrated_tickets[user_id]
            del rating_messages[payload.message_id]

async def save_transcript(channel, user_id):
    messages = []
    async for msg in channel.history(limit=None, oldest_first=True):
        timestamp = msg.created_at.strftime("%Y-%m-%d %H:%M")
        author = msg.author.name
        content = msg.content
        messages.append(f"[{timestamp}] {author}: {content}")

    filename = f"transcripts/ticket_{channel.name}_{user_id}.txt"
    with open(filename, "w", encoding="utf-8") as f:
        f.write("\n".join(messages))

    log_channel = discord.utils.get(channel.guild.text_channels, name=TRANSCRIPT_LOG_CHANNEL_NAME)
    if log_channel:
        await log_channel.send(f"📁 Transcript for ticket {channel.name}", file=discord.File(filename))

@tasks.loop(minutes=1)
async def rating_reminder_loop():
    now = datetime.datetime.utcnow().timestamp()
    for user_id, (msg_id, ts) in list(unrated_tickets.items()):
        if now - ts > 86400:
            user = await bot.fetch_user(user_id)
            try:
                await user.send("⏰ Reminder: Please rate your last support ticket!")
            except:
                pass
            del unrated_tickets[user_id]

@tasks.loop(time=datetime.time(hour=12, minute=0))
async def daily_open_reminder():
    for guild in bot.guilds:
        rating_channel = discord.utils.get(guild.text_channels, name=RATING_LOG_CHANNEL_NAME)
        if rating_channel:
            await rating_channel.send("🌞 Good morning! Support is now **open** for the day.")

def is_support_open():
    now = datetime.datetime.now()
    hour = now.hour
    is_weekend = now.weekday() >= 5
    open_hr, close_hr = SUPPORT_HOURS["weekend"] if is_weekend else SUPPORT_HOURS["weekday"]
    return open_hr <= hour < close_hr

bot.run(os.getenv("KEY"))
