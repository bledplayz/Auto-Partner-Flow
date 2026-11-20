

# 🤝 PartnerFlow — Discord Partnership Bot

A Python Discord bot for **mutual partnership ad exchange** between opt-in Discord servers. Built with `discord.py` and SQLite, with slash commands and an interactive setup panel. No paid API or external database needed.

> **Important:** This is an original starter implementation, not the official Auto Partner bot. The current version matches partners on demand with `/find_partner`; it does **not** automatically post on a timer. It never sends `@everyone`, `@here`, or role pings. Ping preferences are stored but requests for actual pings are not matched in this version. Server admins should only use it in channels designated for partnerships.

## ✨ Features

- `/setup` — interactive setup panel with ad-writing modal and settings display
- `/set_ad` — write or paste your advertisement
- `/import_ad` — select a channel and provide a message ID to import an advertisement
- `/set_partner_channel` — select the channel that receives partner ads
- `/requirements` — configure minimum member count and ping preference
- `/partner_toggle` — opt in/out of matching
- `/partner_status` — review configuration and completed match count
- `/find_partner` — exchange ads with a mutually compatible, unmatched server
- Persistent SQLite storage; prevents duplicate pairings; escape mentions in ads
- Restricted configuration commands to admins with Manage Server permission

### Requirements / ping rules

Suggested defaults for server size:

| Members | Suggested ping setting |
| --- | --- |
| 1–200 | No ping |
| 201–500 | Optional (admin chooses) |
| 501–1,000 | Optional (admin chooses) |
| 1,001+ | Optional (admin chooses) |

**These ranges are guidance, not automatically enforced tiers.** Admins choose the minimum member count and ping preference with `/requirements`. The current matching engine supports `none` and `optional` policies only, and always posts without pings. `role` and `everyone` preferences are recorded but excluded from matching until an opt-in, consent-based ping system is built.

## 1. Create a Discord bot

1. Visit [Discord Developer Portal](https://discord.com/developers/applications).
2. Click **New Application**, name it, and create it.
3. Open **Bot** and select **Reset Token** (or copy the available token). **Never share your token or put it in GitHub.**
4. Under **Installation** / OAuth2, configure Guild Install with `bot` and `applications.commands` scopes.
5. Grant permissions **View Channels**, **Send Messages**, **Embed Links**, and **Read Message History**. You can use the installation link to invite the bot.
6. You do **not** need Message Content intent for this bot. Member counts come from guild metadata.

## 2. Upload to GitHub

1. Create a new repository at [github.com/new](https://github.com/new).
2. Upload `main.py`, `requirements.txt`, `.env.example`, `.gitignore`, and `README.md`.
3. **Never upload `.env`, `*.sqlite3`, or your bot token.**
4. Commit your files.

## 3. Test locally (optional)

Requires Python 3.10+.

```bash
python -m pip install -r requirements.txt
```

Linux/macOS:

```bash
export BOT_TOKEN='YOUR_PRIVATE_TOKEN'
python main.py
```

Windows PowerShell:

```powershell
$env:BOT_TOKEN='YOUR_PRIVATE_TOKEN'
python main.py
```

`.env.example` is a template only; the script reads actual environment variables. It does not automatically load `.env` files.

## 4. Host on Bot-Hosting.net

1. Visit [bot-hosting.net](https://bot-hosting.net/) and create an account.
2. Create a **Python** bot instance if that runtime is available on your plan. Hosting interfaces and pricing can change.
3. Upload your repository files using the panel's file manager or Git integration, if available.
4. Set the startup file/command to `main.py` or `python main.py` as required by the host.
5. Install dependencies from `requirements.txt` (often via the panel's Python packages/dependencies feature).
6. In the host's **environment variables / startup variables / secrets** section, create `BOT_TOKEN` with your private token. If the panel does not support private variables, use a private configuration method supported by the host; **never commit the token**.
7. Start the bot and inspect console logs for `Logged in as` and `Synced ... commands`.
8. Keep the SQLite file on persistent storage. If your host resets the filesystem between restarts, configuration and matches will be lost unless you configure persistent storage or an external database.

## 5. Configure your server

1. Invite the bot to **at least two participating servers**.
2. In each server, run `/set_partner_channel` and select a text channel.
3. Run `/set_ad` to write an advertisement, **or** `/import_ad` with a channel and message ID. To copy a message ID, enable Discord Developer Mode in Advanced settings, then right-click/long-press the message and select Copy Message ID.
4. Run `/requirements` (e.g., minimum members `1`, ping policy `No ping`).
5. Run `/partner_toggle enabled:true` in both servers.
6. Run `/find_partner` to exchange ads. Both servers must meet each other's minimum member count and support no-ping ads.

## ⚠️ Known limitations

- This version matches **on command**, not in a background auto-post loop.
- No support for ping delivery, partnership tickets, staff approvals, web dashboard, automatic sync of edited source ads, or timed matchmaking.
- A failure while posting the second advertisement can leave one advertisement posted; the command warns admins to remove it manually.
- SQLite is suitable for a single bot process. Avoid running multiple replicas against the same SQLite file.
- Discord global slash command propagation may take some time.
- Respect Discord's policies and each server's consent; do not use this to spam or send unsolicited DMs.

## 🔐 Security

All outbound ads disable mentions and escape user-entered mention syntax. The bot does not require Administrator permission. Never share the bot token; reset it in the Developer Portal if exposed.

## 📄 License

MIT — see `LICENSE`.

discord.py>=2.4,<3

BOT_TOKEN=PASTE_YOUR_DISCORD_BOT_TOKEN_HERE
DATABASE_PATH=partnerflow.sqlite3

.env
*.sqlite3
*.sqlite3-journal
__pycache__/
*.pyc

MIT License

Copyright (c) 2026 PartnerFlow contributors

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

import os, asyncio, sqlite3, logging, re, time
from contextlib import contextmanager
from typing import Optional
import discord
from discord import app_commands
from discord.ext import commands, tasks

logging.basicConfig(level=logging.INFO)
TOKEN = os.getenv('BOT_TOKEN')
DB = os.getenv('DATABASE_PATH', 'partnerflow.sqlite3')
if not TOKEN:
    raise RuntimeError('Set BOT_TOKEN environment variable. Never commit your token.')

@contextmanager
def db():
    con = sqlite3.connect(DB, timeout=30)
    con.row_factory = sqlite3.Row
    try:
        yield con
        con.commit()
    finally:
        con.close()

def init_db():
    with db() as c:
        c.execute('''CREATE TABLE IF NOT EXISTS servers (guild_id INTEGER PRIMARY KEY, ad TEXT DEFAULT '', ad_channel INTEGER, partner_channel INTEGER, min_members INTEGER DEFAULT 1, ping_rule TEXT DEFAULT 'none', enabled INTEGER DEFAULT 0, last_post INTEGER DEFAULT 0)''')
        c.execute('''CREATE TABLE IF NOT EXISTS partnerships (a INTEGER, b INTEGER, created_at INTEGER, PRIMARY KEY(a,b))''')

def ensure(gid):
    with db() as c:
        c.execute('INSERT OR IGNORE INTO servers(guild_id) VALUES (?)', (gid,))

def get(gid):
    ensure(gid)
    with db() as c:
        return dict(c.execute('SELECT * FROM servers WHERE guild_id=?', (gid,)).fetchone())

def setval(gid, field, value):
    if field not in {'ad','ad_channel','partner_channel','min_members','ping_rule','enabled','last_post'}:
        raise ValueError('invalid field')
    ensure(gid)
    with db() as c:
        c.execute(f'UPDATE servers SET {field}=? WHERE guild_id=?', (value,gid))

def paired(a,b):
    x,y=sorted((a,b))
    with db() as c:
        return bool(c.execute('SELECT 1 FROM partnerships WHERE a=? AND b=?',(x,y)).fetchone())

def mark_paired(a,b):
    x,y=sorted((a,b))
    with db() as c:
        c.execute('INSERT OR IGNORE INTO partnerships(a,b,created_at) VALUES(?,?,?)',(x,y,int(time.time())))

def compatible(a,b):
    return a['enabled'] and b['enabled'] and a['ad'] and b['ad'] and a['partner_channel'] and b['partner_channel']

def clean_ad(text):
    # Advertisements never get to trigger arbitrary mass/role pings.
    return discord.utils.escape_mentions(text.strip())[:1800]

intents=discord.Intents.default()
intents.guilds=True
bot=commands.Bot(command_prefix='!', intents=intents)

class AdModal(discord.ui.Modal, title='Set your partnership advertisement'):
    advertisement=discord.ui.TextInput(label='Advertisement', style=discord.TextStyle.paragraph, max_length=1800, required=True)
    async def on_submit(self, interaction:discord.Interaction):
        setval(interaction.guild_id,'ad',clean_ad(str(self.advertisement)))
        await interaction.response.send_message('✅ Advertisement saved. Mentions are escaped for safety.',ephemeral=True)

class SetupView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=300)
    @discord.ui.button(label='Write advertisement',style=discord.ButtonStyle.primary,emoji='📝')
    async def write(self,interaction:discord.Interaction,button:discord.ui.Button):
        if not interaction.user.guild_permissions.manage_guild:
            return await interaction.response.send_message('Manage Server permission required.',ephemeral=True)
        await interaction.response.send_modal(AdModal())
    @discord.ui.button(label='View settings',style=discord.ButtonStyle.secondary,emoji='⚙️')
    async def settings(self,interaction:discord.Interaction,button:discord.ui.Button):
        if not interaction.user.guild_permissions.manage_guild:
            return await interaction.response.send_message('Manage Server permission required.',ephemeral=True)
        await interaction.response.send_message(embed=settings_embed(interaction.guild),ephemeral=True)

def settings_embed(guild):
    s=get(guild.id)
    e=discord.Embed(title='🤝 PartnerFlow Setup',description='Configure channels and requirements using slash commands below.',color=discord.Color.blurple())
    e.add_field(name='Status',value='🟢 Enabled' if s['enabled'] else '🔴 Disabled',inline=True)
    e.add_field(name='Minimum members',value=str(s['min_members']),inline=True)
    e.add_field(name='Ping policy',value=s['ping_rule'],inline=True)
    e.add_field(name='Partner channel',value=f"<#{s['partner_channel']}>" if s['partner_channel'] else 'Not selected',inline=True)
    e.add_field(name='Ad source',value=f"<#{s['ad_channel']}>" if s['ad_channel'] else ('Written ad' if s['ad'] else 'Not set'),inline=True)
    e.add_field(name='Advertisement',value=(s['ad'][:300] or 'Not set'),inline=False)
    e.set_footer(text='Pings are policy labels only. Bot advertisements never mass-ping users.')
    return e

@bot.tree.command(name='setup',description='Open your partnership setup panel')
@app_commands.guild_only()
@app_commands.default_permissions(manage_guild=True)
async def setup(interaction:discord.Interaction):
    await interaction.response.send_message(embed=settings_embed(interaction.guild),view=SetupView(),ephemeral=True)

@bot.tree.command(name='set_partner_channel',description='Choose where matched partner ads are posted')
@app_commands.guild_only()
@app_commands.default_permissions(manage_guild=True)
async def set_partner_channel(interaction:discord.Interaction,channel:discord.TextChannel):
    me=interaction.guild.me
    perms=channel.permissions_for(me)
    if not (perms.view_channel and perms.send_messages and perms.embed_links):
        return await interaction.response.send_message('Bot needs View Channel, Send Messages, and Embed Links in that channel.',ephemeral=True)
    setval(interaction.guild_id,'partner_channel',channel.id)
    await interaction.response.send_message(f'✅ Partner channel: {channel.mention}',ephemeral=True)

@bot.tree.command(name='set_ad',description='Write an advertisement using a form')
@app_commands.guild_only()
@app_commands.default_permissions(manage_guild=True)
async def set_ad(interaction:discord.Interaction):
    await interaction.response.send_modal(AdModal())

@bot.tree.command(name='import_ad',description='Import ad text from a message in a chosen channel')
@app_commands.guild_only()
@app_commands.default_permissions(manage_guild=True)
@app_commands.describe(channel='Channel containing your advertisement',message_id='ID of the message to import')
async def import_ad(interaction:discord.Interaction,channel:discord.TextChannel,message_id:str):
    if not message_id.isdecimal():
        return await interaction.response.send_message('Message ID must be numeric.',ephemeral=True)
    try:
        message=await channel.fetch_message(int(message_id))
    except (discord.NotFound,discord.Forbidden,discord.HTTPException):
        return await interaction.response.send_message('Cannot read that message. Check the ID and permissions.',ephemeral=True)
    content=message.content or (message.embeds[0].description if message.embeds and message.embeds[0].description else '')
    if not content.strip():
        return await interaction.response.send_message('Message has no importable text.',ephemeral=True)
    setval(interaction.guild_id,'ad',clean_ad(content))
    setval(interaction.guild_id,'ad_channel',channel.id)
    await interaction.response.send_message('✅ Imported advertisement. Re-run this command if the original changes.',ephemeral=True)

@bot.tree.command(name='requirements',description='Set minimum partner size and ping policy')
@app_commands.guild_only()
@app_commands.default_permissions(manage_guild=True)
@app_commands.choices(ping_policy=[app_commands.Choice(name='No ping (default for 1–200)',value='none'),app_commands.Choice(name='Optional ping',value='optional'),app_commands.Choice(name='Role ping requested',value='role'),app_commands.Choice(name='Everyone ping requested',value='everyone')])
async def requirements(interaction:discord.Interaction,min_members:app_commands.Range[int,1,1000000],ping_policy:app_commands.Choice[str]):
    setval(interaction.guild_id,'min_members',min_members)
    setval(interaction.guild_id,'ping_rule',ping_policy.value)
    await interaction.response.send_message(f'✅ Requirements saved: at least **{min_members}** members, ping policy **{ping_policy.name}**. Ping policies are preferences only; bot does not send mass pings.',ephemeral=True)

@bot.tree.command(name='partner_toggle',description='Enable or disable automatic partnership matching')
@app_commands.guild_only()
@app_commands.default_permissions(manage_guild=True)
async def partner_toggle(interaction:discord.Interaction,enabled:bool):
    s=get(interaction.guild_id)
    if enabled and (not s['ad'] or not s['partner_channel']):
        return await interaction.response.send_message('First set an advertisement and partner channel.',ephemeral=True)
    setval(interaction.guild_id,'enabled',int(enabled))
    await interaction.response.send_message('✅ Matching '+('enabled.' if enabled else 'disabled.'),ephemeral=True)

@bot.tree.command(name='partner_status',description='See your partnership settings and count')
@app_commands.guild_only()
async def partner_status(interaction:discord.Interaction):
    with db() as c:
        count=c.execute('SELECT COUNT(*) FROM partnerships WHERE a=? OR b=?',(interaction.guild_id,interaction.guild_id)).fetchone()[0]
    e=settings_embed(interaction.guild)
    e.add_field(name='Completed matches',value=str(count))
    await interaction.response.send_message(embed=e,ephemeral=True)

@bot.tree.command(name='find_partner',description='Find and exchange ads with one compatible participating server')
@app_commands.guild_only()
@app_commands.default_permissions(manage_guild=True)
async def find_partner(interaction:discord.Interaction):
    await interaction.response.defer(ephemeral=True,thinking=True)
    mine=get(interaction.guild_id)
    if not mine['enabled']:
        return await interaction.followup.send('Enable matching first with /partner_toggle.',ephemeral=True)
    if not mine['ad'] or not mine['partner_channel']:
        return await interaction.followup.send('Set an ad and partner channel first.',ephemeral=True)
    with db() as c:
        candidates=[dict(r) for r in c.execute('SELECT * FROM servers WHERE enabled=1 AND guild_id!=?',(interaction.guild_id,))]
    for other in candidates:
        guild=bot.get_guild(other['guild_id'])
        if not guild or paired(interaction.guild_id,guild.id) or not compatible(mine,other):
            continue
        if guild.member_count is None or interaction.guild.member_count is None:
            continue
        if guild.member_count < mine['min_members'] or interaction.guild.member_count < other['min_members']:
            continue
        # Explicit ping requirements must be mutually compatible.
        if mine['ping_rule'] not in ('none','optional') or other['ping_rule'] not in ('none','optional'):
            continue
        a=interaction.guild.get_channel(mine['partner_channel'])
        b=guild.get_channel(other['partner_channel'])
        if not isinstance(a,discord.TextChannel) or not isinstance(b,discord.TextChannel):
            continue
        try:
            # No @everyone, @here or role mentions, even if included in imported ads.
            await a.send(embed=discord.Embed(title=f'🤝 Partner: {guild.name}',description=other['ad'],color=discord.Color.green()),allowed_mentions=discord.AllowedMentions.none())
            await b.send(embed=discord.Embed(title=f'🤝 Partner: {interaction.guild.name}',description=mine['ad'],color=discord.Color.green()),allowed_mentions=discord.AllowedMentions.none())
        except (discord.Forbidden,discord.HTTPException):
            return await interaction.followup.send('Could not post in both channels. Verify bot permissions. If one post succeeded, remove it manually before retrying.',ephemeral=True)
        mark_paired(interaction.guild_id,guild.id)
        return await interaction.followup.send(f'✅ Partnered with **{guild.name}**! Ads posted in both servers.',ephemeral=True)
    await interaction.followup.send('No compatible unmatched partner found. Both servers must allow no-ping ads and meet each other’s member requirements.',ephemeral=True)

@bot.event
async def on_ready():
    logging.info('Logged in as %s, serving %s guilds',bot.user,len(bot.guilds))

async def main():
    init_db()
    async with bot:
        await bot.login(TOKEN)
        # Sync globally; Discord may take time to display new slash commands.
        synced=await bot.tree.sync()
        logging.info('Synced %s commands',len(synced))
        await bot.connect(reconnect=True)

if __name__=='__main__':
    asyncio.run(main())
