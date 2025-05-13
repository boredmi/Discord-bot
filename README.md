# Discord-bot
Discord bot
import discord
from discord.ext import commands, tasks
import random
import json
import os

intents = discord.Intents.default()
intents.message_content = True
intents.members = True
bot = commands.Bot(command_prefix="!", intents=intents)

# Load or create user data
def load_data():
    if not os.path.exists("data.json"):
        with open("data.json", "w") as f:
            json.dump({}, f)
    with open("data.json", "r") as f:
        return json.load(f)

def save_data(data):
    with open("data.json", "w") as f:
        json.dump(data, f, indent=4)

data = load_data()

# Helper functions
def get_user(user_id):
    if str(user_id) not in data:
        data[str(user_id)] = {
            "currency": 100,
            "level": 1,
            "xp": 0,
            "inventory": [],
            "city": [],
            "job": None,
            "timeout": 0
        }
    return data[str(user_id)]

# Anime-style embed generator with optional image
anime_images = {
    "interact": "https://i.imgur.com/jr6pf0o.gif",
    "work": "https://i.imgur.com/Np5T4hi.gif",
    "rob": "https://i.imgur.com/sMC2gNn.gif",
    "gamble": "https://i.imgur.com/fkQYzSE.gif",
    "levelup": "https://i.imgur.com/V7gkOeB.gif"
}

def anime_embed(title, description, image_key=None):
    embed = discord.Embed(title=f"🌸 {title} 🌸", description=description, color=discord.Color.purple())
    embed.set_thumbnail(url="https://i.imgur.com/8zQH4hr.gif")
    if image_key and image_key in anime_images:
        embed.set_image(url=anime_images[image_key])
    return embed

@bot.event
async def on_ready():
    print(f'Logged in as {bot.user}')

@bot.command()
async def balance(ctx):
    user = get_user(ctx.author.id)
    await ctx.send(embed=anime_embed("Balance", f"{ctx.author.name} has {user['currency']} coins."))

@bot.command()
async def gamble(ctx, amount: int):
    user = get_user(ctx.author.id)
    if amount <= 0 or amount > user['currency']:
        await ctx.send(embed=anime_embed("Gamble Failed", "Invalid amount to gamble."))
        return

    if random.random() < 0.5:
        user['currency'] -= amount
        await ctx.send(embed=anime_embed("Gamble Result", f"You lost {amount} coins!", image_key="gamble"))
    else:
        winnings = amount * 2
        user['currency'] += winnings
        await ctx.send(embed=anime_embed("Gamble Result", f"You won {winnings} coins!", image_key="gamble"))

    save_data(data)

@bot.command()
async def interact(ctx, member: discord.Member):
    await ctx.send(embed=anime_embed("Interaction", f"{ctx.author.name} interacts with {member.name} in the city!", image_key="interact"))

@bot.command()
async def level(ctx):
    user = get_user(ctx.author.id)
    await ctx.send(embed=anime_embed("Level Info", f"{ctx.author.name} is level {user['level']} with {user['xp']} XP."))

@bot.command()
async def work(ctx):
    user = get_user(ctx.author.id)
    if user['timeout'] > 0:
        await ctx.send(embed=anime_embed("Timeout", "You're in timeout and can't work right now."))
        return

    base_earned = random.randint(10, 50)
    job_bonus = {
        "Farmer": 20,
        "Engineer": 40,
        "Artist": 30
    }
    job = user.get('job')
    bonus = job_bonus.get(job, 0)
    earned = base_earned + bonus
    xp_gain = random.randint(5, 15)

    user['currency'] += earned
    user['xp'] += xp_gain
    if user['xp'] >= user['level'] * 100:
        user['xp'] = 0
        user['level'] += 1
        await ctx.send(embed=anime_embed("Level Up!", f"{ctx.author.name} leveled up to {user['level']}!", image_key="levelup"))
    else:
        await ctx.send(embed=anime_embed("Work Result", f"You worked as a {job or 'freelancer'} and earned {earned} coins and {xp_gain} XP!", image_key="work"))
    save_data(data)

@bot.command()
async def shop(ctx):
    items = {"House": 500, "Car": 1000, "Bike": 250, "Club": 1200, "Zoo": 2000, "Store": 1500}
    shop_text = "\n".join([f"{item}: {price} coins" for item, price in items.items()])
    await ctx.send(embed=anime_embed("City Shop", shop_text))

@bot.command()
async def buy(ctx, *, item):
    user = get_user(ctx.author.id)
    items = {"House": 500, "Car": 1000, "Bike": 250, "Club": 1200, "Zoo": 2000, "Store": 1500}
    if item not in items:
        await ctx.send(embed=anime_embed("Shop Error", "Item not found in shop."))
        return
    if user['currency'] < items[item]:
        await ctx.send(embed=anime_embed("Shop Error", "Not enough coins."))
        return
    user['currency'] -= items[item]
    user['inventory'].append(item)
    user['city'].append(item)
    await ctx.send(embed=anime_embed("Shop", f"You bought a {item}! It's now part of your city."))
    save_data(data)

@bot.command()
async def city(ctx):
    user = get_user(ctx.author.id)
    if user['city']:
        await ctx.send(embed=anime_embed("City View", f"{ctx.author.name}'s city has: {', '.join(user['city'])}"))
    else:
        await ctx.send(embed=anime_embed("City View", "Your city is empty. Buy buildings from the shop to populate it!"))

@bot.command()
async def rob(ctx):
    user = get_user(ctx.author.id)
    if user['timeout'] > 0:
        await ctx.send(embed=anime_embed("Crime Denied", "You're in timeout!"))
        return
    if random.random() < 0.4:
        penalty = 100
        user['timeout'] = 3
        user['currency'] = max(0, user['currency'] - penalty)
        await ctx.send(embed=anime_embed("Caught!", f"You were caught robbing a bank and fined {penalty} coins! Police put you in timeout.", image_key="rob"))
    else:
        loot = random.randint(100, 500)
        user['currency'] += loot
        await ctx.send(embed=anime_embed("Heist Success!", f"You successfully robbed a bank and gained {loot} coins!", image_key="rob"))
    save_data(data)

@bot.command()
async def job(ctx, *, job_name):
    user = get_user(ctx.author.id)
    valid_jobs = ["Farmer", "Engineer", "Artist"]
    if job_name not in valid_jobs:
        await ctx.send(embed=anime_embed("Job Error", f"Valid jobs: {', '.join(valid_jobs)}"))
        return
    user['job'] = job_name
    await ctx.send(embed=anime_embed("New Job", f"You are now working as a {job_name}!"))
    save_data(data)

@bot.command()
async def pvp(ctx, member: discord.Member):
    attacker = get_user(ctx.author.id)
    defender = get_user(member.id)
    if attacker['timeout'] > 0:
        await ctx.send(embed=anime_embed("PvP Error", "You're in timeout and can't fight."))
        return
    if random.random() < 0.5:
        reward = random.randint(50, 150)
        attacker['currency'] += reward
        defender['currency'] = max(0, defender['currency'] - reward)
        await ctx.send(embed=anime_embed("PvP Result", f"{ctx.author.name} defeated {member.name} and stole {reward} coins!"))
    else:
        penalty = random.randint(20, 50)
        attacker['currency'] = max(0, attacker['currency'] - penalty)
        await ctx.send(embed=anime_embed("PvP Result", f"{ctx.author.name} lost the fight and paid {penalty} coins as penalty!"))
    save_data(data)

# Timeout decay
@tasks.loop(minutes=1)
async def reduce_timeout():
    for user_id in data:
        if data[user_id]["timeout"] > 0:
            data[user_id]["timeout"] -= 1
    save_data(data)

reduce_timeout.start()

bot.run("YOUR_BOT_TOKEN")
