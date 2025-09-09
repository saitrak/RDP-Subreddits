import discord
from discord.ext import commands
import asyncio
import asyncpraw
import random

# ====== YOUR CREDENTIALS ======
TOKEN = "MTQxMDY0OTg3MTE5ODcyMDA3Mg.G38pmS.-UcXKdl_1soko2j8X5ScJd1_xsQnQ0xgraW-Us"
REDDIT_CLIENT_ID = "IWIlurvX-dNFuoQt4SNxCA"
REDDIT_CLIENT_SECRET = "KgpkeXA0hX9Wgk5oi5u1A8Ijx4AUug"
REDDIT_USER_AGENT = "discord-bot:v1.0 (by u/Imaginary-Run-2831)"
DEFAULT_INTERVAL = 60  # seconds
# ==============================

intents = discord.Intents.default()
intents.message_content = True
bot = commands.Bot(command_prefix="m$", intents=intents)

reddit = None

# channel_id -> { "subreddit": str, "seen": set, "interval": int, "mode": str }
# mode = "interval" or "new"
channel_settings = {}

# Track running tasks for each channel
channel_tasks = {}

# ===================== REDDIT FETCHING =====================
async def fetch_new_post(subreddit_name, seen_set, reddit_client):
    subreddit = await reddit_client.subreddit(subreddit_name)
    async for post in subreddit.new(limit=50):
        if post.id not in seen_set and not post.stickied and post.url.endswith((".jpg", ".png", ".jpeg", ".gif")):
            seen_set.add(post.id)
            return post
    return None

async def fetch_fallback_post(subreddit_name, seen_set, reddit_client):
    subreddit = await reddit_client.subreddit(subreddit_name)
    candidates = []
    async for post in subreddit.top(limit=100):
        if post.id not in seen_set and not post.stickied and post.url.endswith((".jpg", ".png", ".jpeg", ".gif")):
            candidates.append(post)
    if candidates:
        post = random.choice(candidates)
        seen_set.add(post.id)
        return post
    return None

async def send_meme(channel, subreddit_name, post):
    embed = discord.Embed(
        title=post.title,
        url=f"https://reddit.com{post.permalink}",
        color=random.randint(0, 0xFFFFFF)
    )
    embed.set_image(url=post.url)
    embed.set_footer(text=f"r/{subreddit_name}")
    await channel.send(embed=embed)

# ===================== BACKGROUND LOOP =====================
async def post_loop(channel, settings, reddit_client):
    seen_set = settings["seen"]
    subreddit_name = settings["subreddit"]
    mode = settings["mode"]

    if mode == "new":  # instant mode
        subreddit = await reddit_client.subreddit(subreddit_name)
        async for post in subreddit.stream.submissions(skip_existing=True):
            if not post.stickied and post.url.endswith((".jpg", ".png", ".jpeg", ".gif")):
                if post.id not in seen_set:
                    seen_set.add(post.id)
                    await send_meme(channel, subreddit_name, post)

    else:  # interval mode
        while True:
            try:
                post = await fetch_new_post(subreddit_name, seen_set, reddit_client)
                if not post:
                    post = await fetch_fallback_post(subreddit_name, seen_set, reddit_client)
                if post:
                    await send_meme(channel, subreddit_name, post)
                else:
                    print(f"⚠️ No memes for r/{subreddit_name}")
            except Exception as e:
                print(f"⚠️ Error in loop for r/{subreddit_name}: {e}")
            await asyncio.sleep(settings["interval"])

# ===================== COMMANDS =====================
@bot.command()
async def setchannel(ctx, subreddit_name: str):
    """Subscribe this channel to a subreddit"""
    # Check if channel already has a subreddit
    if ctx.channel.id in channel_settings:
        await ctx.send(f"⚠️ This channel is already linked to r/{channel_settings[ctx.channel.id]['subreddit']}. Use `m$unsetchannel` first.")
        return
    
    channel_settings[ctx.channel.id] = {
        "subreddit": subreddit_name.lower(),
        "seen": set(),
        "interval": DEFAULT_INTERVAL,
        "mode": "interval"
    }
    
    # Cancel any existing task for this channel
    if ctx.channel.id in channel_tasks:
        channel_tasks[ctx.channel.id].cancel()
    
    # Start new task
    channel_tasks[ctx.channel.id] = bot.loop.create_task(
        post_loop(ctx.channel, channel_settings[ctx.channel.id], reddit)
    )
    
    await ctx.send(f"✅ This channel is now linked to r/{subreddit_name}!")

@bot.command()
async def unsetchannel(ctx):
    """Unsubscribe this channel"""
    if ctx.channel.id in channel_settings:
        # Cancel the background task
        if ctx.channel.id in channel_tasks:
            channel_tasks[ctx.channel.id].cancel()
            del channel_tasks[ctx.channel.id]
        
        # Remove channel settings
        del channel_settings[ctx.channel.id]
        await ctx.send("🛑 Meme stream stopped in this channel.")
    else:
        await ctx.send("⚠️ This channel isn't subscribed to any subreddit.")

@bot.command()
async def setchanneltime(ctx, minutes: int):
    """Set post interval in minutes"""
    if ctx.channel.id not in channel_settings:
        await ctx.send("❌ This channel has no subreddit subscription.")
        return
    
    if minutes < 1:
        await ctx.send("⚠️ Please provide a positive number of minutes.")
        return
    
    channel_settings[ctx.channel.id]["interval"] = minutes * 60
    channel_settings[ctx.channel.id]["mode"] = "interval"
    
    # Restart the task with new interval
    if ctx.channel.id in channel_tasks:
        channel_tasks[ctx.channel.id].cancel()
    
    channel_tasks[ctx.channel.id] = bot.loop.create_task(
        post_loop(ctx.channel, channel_settings[ctx.channel.id], reddit)
    )
    
    await ctx.send(f"⏱ Interval updated: {minutes} minutes per post.")

@bot.command()
async def time(ctx, action: str, value: str = None):
    """Change interval or set instant mode"""
    if ctx.channel.id not in channel_settings:
        await ctx.send("❌ This channel has no subreddit subscription.")
        return

    if action == "set":
        try:
            minutes = int(value)
            if minutes < 1:
                await ctx.send("⚠️ Please provide a positive number of minutes.")
                return
                
            channel_settings[ctx.channel.id]["interval"] = minutes * 60
            channel_settings[ctx.channel.id]["mode"] = "interval"
            
            # Restart the task with new interval
            if ctx.channel.id in channel_tasks:
                channel_tasks[ctx.channel.id].cancel()
            
            channel_tasks[ctx.channel.id] = bot.loop.create_task(
                post_loop(ctx.channel, channel_settings[ctx.channel.id], reddit)
            )
            
            await ctx.send(f"⏱ Interval updated: {minutes} minutes per post.")
        except:
            await ctx.send("⚠️ Please provide a valid number of minutes.")

    elif action == "new":
        channel_settings[ctx.channel.id]["mode"] = "new"
        
        # Restart the task with new mode
        if ctx.channel.id in channel_tasks:
            channel_tasks[ctx.channel.id].cancel()
        
        channel_tasks[ctx.channel.id] = bot.loop.create_task(
            post_loop(ctx.channel, channel_settings[ctx.channel.id], reddit)
        )
        
        await ctx.send("⚡ Instant new-post mode enabled (no fallback).")

    else:
        await ctx.send("⚠️ Usage: `m$time set <minutes>` or `m$time new`")

# ===================== BOT READY =====================
@bot.event
async def on_ready():
    global reddit
    reddit = asyncpraw.Reddit(
        client_id=REDDIT_CLIENT_ID,
        client_secret=REDDIT_CLIENT_SECRET,
        user_agent=REDDIT_USER_AGENT
    )
    reddit.read_only = True
    print(f"RDP-Subreddits is online!")

bot.run(TOKEN)
