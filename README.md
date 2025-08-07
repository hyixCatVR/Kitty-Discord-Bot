# Kitty Bot — Discord Treat & Combat Bot

import discord
import json
import random
import os
import asyncio
from datetime import datetime, timezone, timedelta
from discord import app_commands
from discord.ui import Select, View, Button
import time
import threading

PING_FILE = "/tmp/kitty_ping"
PING_TIMEOUT = 20

def monitor_pings():
    while True:
        try:
            with open(PING_FILE, "r") as f:
                last_ping = float(f.read().strip())
        except:
            last_ping = 0
        if time.time() - last_ping > PING_TIMEOUT:
            print("❌ No ping received for 20s. Kitty Bot shutting down.")
            os._exit(0)
        time.sleep(5)

#########################
# ── Bot & Intents Setup ──
#########################
intents = discord.Intents.default()
intents.members = True
kitty = discord.Client(intents=intents)
tree = app_commands.CommandTree(kitty)
tree._global_commands.clear()
active_battles = set()

#########################
# ── Persistent Storage ──
#########################
def load_json(path):
    try:
        return json.load(open(path, "r"))
    except:
        return {}

def save_json(path, data):
    json.dump(data, open(path, "w"), indent=4)

user_treats        = load_json("Kitty_Bot/treats.json")
user_inventory     = load_json("Kitty_Bot/inventory.json")
leaderboard_data   = load_json("Kitty_Bot/leaderboard.json")
command_count      = load_json("Kitty_Bot/counter.json").get("command_counter", 0)
pet_command_count  = load_json("Kitty_Bot/pet_counter.json").get("pet_command_counter", 0)
user_health = {}
boss_health = {}

def save_counts():
    save_json("Kitty_Bot/counter.json", {"command_counter": command_count})
    save_json("Kitty_Bot/pet_counter.json", {"pet_command_counter": pet_command_count})

def save_all():
    save_json("Kitty_Bot/treats.json", user_treats)
    save_json("Kitty_Bot/inventory.json", user_inventory)
    save_json("Kitty_Bot/leaderboard.json", leaderboard_data)

def update_leaderboard(gid, uid):
    leaderboard_data.setdefault(gid, {})[uid] = user_treats.get(uid, 0)
    save_all()

#########################
# ── Cooldown Mechanism ──
#########################
user_last = {}
async def check_cd(inter):
    uid = inter.user.id
    now = datetime.now(timezone.utc)
    if uid in user_last and (now - user_last[uid]) < timedelta(seconds=10):
        await inter.response.send_message("🐢 Slow down a bit!", ephemeral=True)
        return True
    user_last[uid] = now
    return False

async def periodic_leaderboard_update():
    while True:
        for guild in kitty.guilds:
            for member in guild.members:
                update_leaderboard(str(guild.id), str(member.id))
        await kitty.change_presence(activity=discord.Activity(
            type=discord.ActivityType.watching,
            name=f"{len(kitty.guilds)} servers"
        ))
        await asyncio.sleep(300)

######################### 
# ── Leaderboard ──
#########################
@tree.command(name="leaderboard", description="Show server or global leaderboard.")
@app_commands.describe(global_lb="True = global leaderboard")
async def leaderboard(inter, global_lb: bool = False):
    await inter.response.defer()
    if global_lb:
        aggregate = {}
        for guild_map in leaderboard_data.values():
            for uid, t in guild_map.items():
                aggregate[uid] = aggregate.get(uid, 0) + t
        data = sorted(aggregate.items(), key=lambda x: x[1], reverse=True)
        title = "🌍 Global Leaderboard"
    else:
        gid = str(inter.guild.id)
        data = sorted(leaderboard_data.get(gid, {}).items(), key=lambda x: x[1], reverse=True)
        title = f"🏆 {inter.guild.name} Leaderboard"

    lines = []
    for i, (u, t) in enumerate(data[:10], 1):
        m = inter.guild.get_member(int(u))
        display = m.display_name if m and not m.bot else f"<@{u}>"
        lines.append(f"`#{i}` {display} — **{t} Treats**")

    await inter.followup.send(f"**{title}**\n" + "\n".join(lines))

#########################
# ── Shop ──
#########################
class ShopDrop(Select):
    def __init__(self):
        opts = [
            discord.SelectOption(label=x, value=x, description=f"Cost: {c}")
            for x, c in [
                ("Wooden Sword",10),("Stone Sword",30),("Iron Sword",65),
                ("Gold Sword",100),("Diamond Sword",190),("Netherite Sword",300)
            ]
        ]
        super().__init__(placeholder="Choose an item…", options=opts)

    async def callback(self, inter):
        item = self.values[0]
        costs = {"Wooden Sword":10, "Stone Sword":30, "Iron Sword":65,
                 "Gold Sword":100, "Diamond Sword":190, "Netherite Sword":300}
        price = costs[item]
        view = PurchaseView(item, price, inter.user.id)
        embed = discord.Embed(title="Purchase?", description=f"Buy **{item}** for **{price} Treats**?", color=discord.Color.blue())
        await inter.response.send_message(embed=embed, view=view, ephemeral=True)

class ShopView(View):
    def __init__(self, uid):
        super().__init__(timeout=None)
        self.add_item(ShopDrop())

class PurchaseView(View):
    def __init__(self, item, price, uid):
        super().__init__(timeout=None)
        self.item, self.price, self.uid = item, price, str(uid)

    async def interaction_check(self, inter):
        return str(inter.user.id) == self.uid

    @discord.ui.button(label="Confirm", style=discord.ButtonStyle.green)
    async def confirm(self, inter, _):
        if user_treats.get(self.uid, 0) >= self.price:
            user_treats[self.uid] -= self.price
            user_inventory.setdefault(self.uid, []).append(self.item)
            save_all()
            await inter.response.edit_message(content=f"✅ Purchased **{self.item}**!", view=None)
        else:
            await inter.response.edit_message(content="❌ Not enough Treats!", view=None)

    @discord.ui.button(label="Cancel", style=discord.ButtonStyle.red)
    async def cancel(self, inter, _):
        await inter.response.edit_message(content="❌ Cancelled.", view=None)

@tree.command(name="invite", description="Get the bot's invite link.")
async def invite(inter):
    link = "https://discord.com/oauth2/authorize?client_id=1367206381173735545&permissions=1101659162630&integration_type=0&scope=bot+applications.commands"
    await inter.response.send_message(link, ephemeral=True)

@tree.command(name="shop", description="Buy items from shop.")
async def shop(inter):
    if await check_cd(inter): return
    uid = str(inter.user.id)
    t = user_treats.get(uid, 0)
    await inter.response.send_message(f"You have **{t}** Treats. Select an item:", view=ShopView(uid), ephemeral=True)

#########################
# ── Combat ──
#########################
def dmg_roll(name):
    mapping = {
        "Wooden Sword":(1,7),"Stone Sword":(5,13),"Iron Sword":(12,20),
        "Gold Sword":(20,35),"Diamond Sword":(35,50),"Netherite Sword":(50,75)
    }
    mn, mx = mapping.get(name,(1,1))
    return random.randint(mn,mx)

class BattleView(View):
    def __init__(self, uid):
        super().__init__(timeout=None)
        self.uid = str(uid)

    async def interaction_check(self, inter):
        return str(inter.user.id) == self.uid

    async def end_battle(self, inter, message, win=False):
        active_battles.discard(self.uid)
        self.disable_all_items()
        await inter.response.edit_message(content=message, view=self)

    def disable_all_items(self):
        for item in self.children:
            item.disabled = True

    @discord.ui.button(label="Attack⚔️", style=discord.ButtonStyle.red)
    async def attack(self, inter, _):
        uid = self.uid
        sword = max(user_inventory.get(uid, []), key=dmg_roll, default=None)
        d = dmg_roll(sword)
        boss_health[uid] -= d
        msg = f"🗡️ You hit with **{sword or 'fists'}**, dealing **{d}** damage."

        if boss_health[uid] <= 0:
            reward = 35
            user_treats[uid] = user_treats.get(uid, 0) + reward
            save_all()
            await self.end_battle(inter, msg + f"\n🏆 Boss defeated! +{reward} Treats")
            return

        user_health[uid] -= 15
        if user_health[uid] <= 0:
            await self.end_battle(inter, msg + "\n💀 You were defeated!")
            return

        await inter.response.edit_message(
            content=msg + f"\nBoss HP: {boss_health[uid]} | Your HP: {user_health[uid]}",
            view=self
        )

    @discord.ui.button(label="Defend🛡️", style=discord.ButtonStyle.blurple)
    async def defend(self, inter, _):
        uid = self.uid
        if random.random() < 0.65:
            heal = random.randint(1, 12)
            user_health[uid] = min(100, user_health[uid] + heal)
            msg = f"🛡️ You defended and healed **{heal}** HP."
        else:
            user_health[uid] -= 12
            msg = f"🛡️ Defense failed — you took **12** damage."

        if user_health[uid] <= 0:
            await self.end_battle(inter, msg + "\n💀 You died!")
            return

        await inter.response.edit_message(
            content=msg + f"\nBoss HP: {boss_health[uid]} | Your HP: {user_health[uid]}",
            view=self
        )

@tree.command(name="attack", description="Fight the boss!")
async def attack(inter: discord.Interaction):
    if await check_cd(inter): return
    uid = str(inter.user.id)

    if uid in active_battles:
        await inter.response.send_message("⚔️ You're already fighting a boss!", ephemeral=True)
        return

    active_battles.add(uid)
    user_health[uid], boss_health[uid] = 100, 300
    update_leaderboard(str(inter.guild.id), uid)

    fp = discord.File("Kitty_Bot/catboss.png")
    await inter.response.send_message(
        "🐱 A wild boss appears!\nBoss HP: 300 | Your HP: 100",
        file=fp,
        view=BattleView(uid)
    )

    async def cleanup():
        await asyncio.sleep(300)
        active_battles.discard(uid)

    asyncio.create_task(cleanup())

#########################
# ── Pet & Feed ──
#########################
@tree.command(name="pet", description="Pet the cat to earn a treat.")
async def pet(inter):
    global pet_command_count
    if await check_cd(inter): return
    pet_command_count += 1
    save_counts()
    uid = str(inter.user.id)
    user_treats[uid] = user_treats.get(uid, 0) + 1
    update_leaderboard(str(inter.guild.id), uid)
    save_all()
    await inter.response.send_message(f"You got a Treat! <:treat:1382769479964033144>. Command used **{pet_command_count}** times.", file=discord.File("Kitty_Bot/catbeingpet.gif"))

@tree.command(name="feed", description="Feed the cat to earn a treat.")
async def feed(inter):
    global command_count
    if await check_cd(inter): return
    command_count += 1
    save_counts()
    uid = str(inter.user.id)
    user_treats[uid] = user_treats.get(uid, 0) + 1
    update_leaderboard(str(inter.guild.id), uid)
    save_all()
    await inter.response.send_message(f"You got a Treat! <:treat:1382769479964033144>. Command used **{command_count}** times.", file=discord.File("Kitty_Bot/cateatingtreat.gif"))

#########################
# ── Inventory ──
#########################
@tree.command(name="inv", description="Show your or another member's inventory.")
async def inv(inter, user: discord.User = None):
    if await check_cd(inter): return
    target = user or inter.user
    uid = str(target.id)
    invs = user_inventory.get(uid, [])
    t = user_treats.get(uid, 0)
    txt = "\n".join(invs) or "Nothing bought yet."
    await inter.response.send_message(f"**{target.display_name}** has **{t}** Treats<:treat:1382769479964033144>\nInventory:\n{txt}")

#########################
# ── Member Events ──
#########################
@kitty.event
async def on_member_join(member):
    update_leaderboard(str(member.guild.id), str(member.id))

@kitty.event
async def on_member_remove(member):
    gid, uid = str(member.guild.id), str(member.id)
    if uid in leaderboard_data.get(gid, {}):
        del leaderboard_data[gid][uid]
        save_all()

#########################
# ── Launch ──
#########################
@kitty.event
async def on_ready():
    kitty.loop.create_task(periodic_leaderboard_update())
    await tree.sync()
    print(f"Logged in as {kitty.user}")

kitty.run("[CENSORED]")
