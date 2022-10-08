import async_timeout
import discord
from discord.channel import VocalGuildChannel, VoiceChannel
from discord.ext.commands.core import guild_only
from discord.message import DeletedReferencedMessage
from discord.utils import get
from discord import FFmpegPCMAudio
import youtube_dl
import asyncio
from functools import partial
from async_timeout import timeout
from discord.ext import commands
import itertools
import os
from discord.ext import commands, tasks
from pprint import pformat
from discord.ext import tasks, commands
from discord.ext import tasks
import os
import random
from dotenv import load_dotenv
from discord.ext import commands
from keep_alive import keep_alive
from discord.ext.commands import Bot
import discord
from discord.ext import commands
import asyncio
from discord.ext import tasks, commands

client = discord.Client()

bot = commands.Bot(command_prefix='!', help_command=None)


###########################################################################################################
#คำสั่งคิวเพลง

youtube_dl.utils.bug_reports_message = lambda: ''

ytdl_format_options = {
    'format': 'bestaudio/best',
    'outtmpl': '%(extractor)s-%(id)s-%(title)s.%(ext)s',
    'restrictfilenames': True,
    'noplaylist': False,
    'nocheckcertificate': True,
    'ignoreerrors': False,
    'logtostderr': False,
    'quiet': True,
    'no_warnings': True,
    'default_search': 'auto',
    'source_address':
    '0.0.0.0'  # bind to ipv4 since ipv6 addresses cause issues sometimes
}

ffmpeg_options = {
    'options': '-vn',
    "before_options":
    "-reconnect 1 -reconnect_streamed 1 -reconnect_delay_max 5"  ## song will end if no this line
}

ytdl = youtube_dl.YoutubeDL(ytdl_format_options)


class YTDLSource(discord.PCMVolumeTransformer):
    def __init__(self, source, *, data, requester):
        super().__init__(source)
        self.requester = requester

        self.title = data.get('title')
        self.web_url = data.get('webpage_url')

        # YTDL info dicts (data) have other useful information you might want
        # https://github.com/rg3/youtube-dl/blob/master/README.md

    def __getitem__(self, item: str):
        """Allows us to access attributes similar to a dict.
        This is only useful when you are NOT downloading.
        """
        return self.__getattribute__(item)

    @classmethod
    async def create_source(cls, ctx, search: str, *, loop, download=False):
        loop = loop or asyncio.get_event_loop()

        to_run = partial(ytdl.extract_info, url=search, download=download)
        data = await loop.run_in_executor(None, to_run)

        if 'entries' in data:
            # take first item from a playlist
            data = data['entries'][0]

        await ctx.send(f'```ini\n[Added {data["title"]} to the Queue.]\n```'
                       )  #delete after can be added

        if download:
            source = ytdl.prepare_filename(data)
        else:
            return {
                'webpage_url': data['webpage_url'],
                'requester': ctx.author,
                'title': data['title']
            }

        return cls(discord.FFmpegPCMAudio(source, **ffmpeg_options),
                   data=data,
                   requester=ctx.author)

    @classmethod
    async def regather_stream(cls, data, *, loop):
        """Used for preparing a stream, instead of downloading.
        Since Youtube Streaming links expire."""
        loop = loop or asyncio.get_event_loop()
        requester = data['requester']

        to_run = partial(ytdl.extract_info,
                         url=data['webpage_url'],
                         download=False)
        data = await loop.run_in_executor(None, to_run)

        return cls(discord.FFmpegPCMAudio(data['url'], **ffmpeg_options),
                   data=data,
                   requester=requester)


class MusicPlayer:
    """A class which is assigned to each guild using the bot for Music.
    This class implements a queue and loop, which allows for different guilds to listen to different playlists
    simultaneously.
    When the bot disconnects from the Voice it's instance will be destroyed.
    """

    __slots__ = ('bot', '_guild', '_channel', '_cog', 'queue', 'next',
                 'current', 'np', 'volume')

    def __init__(self, ctx):
        self.bot = ctx.bot
        self._guild = ctx.guild
        self._channel = ctx.channel
        self._cog = ctx.cog

        self.queue = asyncio.Queue()
        self.next = asyncio.Event()

        self.np = None  # Now playing message
        self.volume = .5
        self.current = None

        ctx.bot.loop.create_task(self.player_loop())

    async def player_loop(self):
        """Our main player loop."""
        await self.bot.wait_until_ready()

        while not self.bot.is_closed():
            self.next.clear()

            try:
                # Wait for the next song. If we timeout cancel the player and disconnect...
                async with timeout(300):  # 5 minutes...
                    source = await self.queue.get()
            except asyncio.TimeoutError:
                return await self.destroy(self._guild)

            if not isinstance(source, YTDLSource):
                # Source was probably a stream (not downloaded)
                # So we should regather to prevent stream expiration
                try:
                    source = await YTDLSource.regather_stream(
                        source, loop=self.bot.loop)
                except Exception as e:
                    await self._channel.send(
                        f'There was an error processing your song.\n'
                        f'```css\n[{e}]\n```')
                    continue

            source.volume = self.volume
            self.current = source

            self._guild.voice_client.play(source,
                                          after=lambda _: self.bot.loop.
                                          call_soon_threadsafe(self.next.set))
            self.np = await self._channel.send(
                f'**Now Playing:** `{source.title}` requested by '
                f'`{source.requester}`')
            await self.next.wait()

            # Make sure the FFmpeg process is cleaned up.
            source.cleanup()
            self.current = None

            try:
                # We are no longer playing this song...
                await self.np.delete()
            except discord.HTTPException:
                pass

    async def destroy(self, guild):
        """Disconnect and cleanup the player."""
        del players[self._guild]
        await self._guild.voice_client.disconnect()
        return self.bot.loop.create_task(self._cog.cleanup(guild))


###########################################################################################################
#ส่วนเสริม




class songAPI:
    def __init__(self):
        self.players = {}






@bot.command()
async def test(ctx, *, par):
    await ctx.channel.send("You typed {0}".format(par))




@bot.event
async def on_ready():
    print("Bot is ready!")



@bot.command()
async def ping(ctx):
    ping_ = bot.latency
    ping =  round(ping_ * 1000)
    await ctx.send(f"my ping is {ping}ms")


###########################################################################################################
#คำลั่งพูดคุย


@bot.event
async def on_message(message):
    if message.content == 'code':
        await message.channel.send('')
        print(message.channel)
    elif message.content == 'สวัสดีค่ะ':
        await message.channel.send('สวัสดีค่ะ ' + str(message.author.name))
        print(message.channel)
    elif message.content == '':
        await message.channel.send('')
        print(message.channel)
    elif message.content == '':
        await message.channel.send('')
        print(message.channel)
    elif message.content == 'นอนละ':
        await message.channel.send('ฝันดีค่ะ')
        print(message.channel)
    elif message.content == '':
        await message.channel.send('')
        print(message.channel)
    elif message.content == 'สวัสดีครับ':
        await message.channel.send('สวัสดีค่ะ ' + str(message.author.name))
        print(message.channel)
    elif message.content == 'มิ้น':
        await message.channel.send('คะ?')
        print(message.channel)
    elif message.content == 'เบื่อจัง':
        await message.channel.send('มาเล่นกับมิ้นก็ได้นะคะ')
        print(message.channel)
    elif message.content == 'Happy New Year!':
        await message.channel.send('???')
    elif message.content == '...':
        await message.channel.send('-.-')
        print(message.channel)
    elif message.content == 'สบายดีไหม':
        await message.channel.send('มิ้นสบายดีค่ะ')
    elif message.content == 'สวัสดีมิ้น':
        await message.channel.send(str(message.author.name) + ' สวัสดีค่ะ')
        print(message.channel)
    elif message.content == '!logout':
        await bot.logout()    
    await bot.process_commands(message)



###########################################################################################################
#คำสั่งแนะนำ




@bot.command()
async def help(ctx):
    # help
    # test
    # send
    emBed = discord.Embed(title="คำสั่งทั้งหมดของมิ้นค่ะ",
                          description="",
                          color=0x00FFFF)
    emBed.add_field(name="!play", value="มิ้นจะเปิดเพลงให้ฟังค่ะ", inline=False)
    emBed.add_field(name="!pause",
                    value="มิ้นจะหยุดเพลงชั่วคราวให้ค่ะ",
                    inline=False)
    emBed.add_field(name="!resume",
                    value="มิ้นจะเปิดเพลงต่อจากที่หยุดไว้ค่ะ",
                    inline=False)
    emBed.add_field(name="!queuelist",
                    value="มิ้นจะเปิดคิวเพลงทั้งหมดให้ดูค่ะ",
                    inline=False)
    emBed.add_field(name="!clear",
                     value="มิ้นจะลบคิวเพลงทั้งหมดให้ค่ะ", inline=False)
    emBed.add_field(name="!skip", value="มิ้นจะเปิดเพลงถัดไปทันทีค่ะ", inline=False)
    emBed.add_field(name="!leave หรือ !stop",
                    value="มิ้นจะออกจากห้องทันทีค่ะ",
                    inline=False)
    emBed.add_field(name="!join",
                     value="มิ้นจะเข้าไปอยู่เป็นเพื่อนให้ค่ะ", inline=False)                
    emBed.set_thumbnail(
        url=
        'https://scontent.xx.fbcdn.net/v/t1.15752-9/s206x206/248000378_578412716606282_1833479855126826773_n.png?_nc_cat=109&ccb=1-5&_nc_sid=aee45a&_nc_eui2=AeFm-Jp_s9uFlhqfHsU1je69xWZKVMzr4aPFZkpUzOvho0lJzT6W2UdUm7cfV7I1nWzYv_06TOABT4uscM-jyHnG&_nc_ohc=XVnyAm3bf0gAX9q_9_k&_nc_ad=z-m&_nc_cid=0&_nc_ht=scontent.xx&oh=02193f9f9b12f62b1bd89047865eb2eb&oe=619E1F15'
    )
    emBed.set_footer(
        text='by Mint',
        icon_url=
        'https://scontent.xx.fbcdn.net/v/t1.15752-9/s206x206/248000378_578412716606282_1833479855126826773_n.png?_nc_cat=109&ccb=1-5&_nc_sid=aee45a&_nc_eui2=AeFm-Jp_s9uFlhqfHsU1je69xWZKVMzr4aPFZkpUzOvho0lJzT6W2UdUm7cfV7I1nWzYv_06TOABT4uscM-jyHnG&_nc_ohc=XVnyAm3bf0gAX9q_9_k&_nc_ad=z-m&_nc_cid=0&_nc_ht=scontent.xx&oh=02193f9f9b12f62b1bd89047865eb2eb&oe=619E1F15'
    )
    await ctx.channel.send(embed=emBed)
    


###########################################################################################################
#คำลั่งเข้าร่วม
  


@bot.command()
async def join(ctx):
    channel = ctx.author.voice.channel
    voice_client = get(bot.voice_clients, guild=ctx.guild)
    
    if voice_client == None:
        await ctx.channel.send("-w-")
        await channel.connect()
        voice_client = get(bot.voice_clients, guild=ctx.guild)



###########################################################################################################
#คำลั้งเล่นเพลง



@bot.command()
async def p(ctx, *, search: str):
    channel = ctx.author.voice.channel
    voice_client = get(bot.voice_clients, guild=ctx.guild)
    
    if voice_client == None:
        await ctx.channel.send("```music start!```")
        await channel.connect()
        voice_client = get(bot.voice_clients, guild=ctx.guild)


    
    _player = get_player(ctx)
    source = await YTDLSource.create_source(ctx,
                                            search,
                                            loop=bot.loop,
                                            download=False)

    await _player.queue.put(source)





@bot.command()
async def play(ctx, *, search: str):
    channel = ctx.author.voice.channel
    voice_client = get(bot.voice_clients, guild=ctx.guild)


  
    if voice_client == None:
        await ctx.channel.send("```music start!```")
        await channel.connect()
        voice_client = get(bot.voice_clients, guild=ctx.guild)
        print(message.channel)

  
    
    _player = get_player(ctx)
    source = await YTDLSource.create_source(ctx,
                                            search,
                                            loop=bot.loop,
                                            download=False)

    await _player.queue.put(source)







players = {}


def get_player(ctx):
    try:
        player = players[ctx.guild.id]
    except:
        player = MusicPlayer(ctx)
        players[ctx.guild.id] = player

    return player







  
###########################################################################################################
#คำสั้งเสริมสำหรับเล่นเพลง



@bot.command()
async def skip(ctx):
    voice_client = get(bot.voice_clients, guild=ctx.guild)
    if voice_client == None:
        await ctx.channel.send("skip now!")
        return

    if voice_client.channel != ctx.author.voice.channel:
        await ctx.channel.send("skip Error".format(
            voice_client.channel))
        return
    voice_client.stop()


@bot.command()
async def pause(ctx):
    voice_client = get(bot.voice_clients, guild=ctx.guild)
    if voice_client == None:
        await ctx.channel.send("pôz")
        return

    if voice_client.channel != ctx.author.voice.channel:
        await ctx.channel.send("pôz Error".format(
            voice_client.channel))
        return
    voice_client.pause()


@bot.command()
async def resume(ctx):
    voice_client = get(bot.voice_clients, guild=ctx.guild)
    if voice_client == None:
        await ctx.channel.send("not responding")
        return

    if voice_client.channel != ctx.author.voice.channel:
        await ctx.channel.send("not responding".format(
            voice_client.channel))
        return

    voice_client.resume()


@bot.command()
async def queue(ctx):
    voice_client = get(bot.voice_clients, guild=ctx.guild)

    if voice_client == None or not voice_client.is_connected():
        await ctx.channel.send("not responding")
        return

    player = get_player(ctx)
    if player.queue.empty():
        return await ctx.send('....')

    upcoming = list(
        itertools.islice(player.queue._queue, 0, player.queue.qsize()))
    fmt = '\n'.join(f'**`{_["title"]}`**' for _ in upcoming)
    embed = discord.Embed(title=f'Upcoming - Next {len(upcoming)}',
                          description=fmt)
    await ctx.send(embed=embed)


@bot.command()
async def stop(ctx):
    await ctx.voice_client.disconnect()
    await ctx.channel.send("!stop!")
@bot.command()
async def leave(ctx):
    await ctx.voice_client.disconnect()
    await ctx.channel.send("!leave!")


@bot.command()
async def wakeup(ctx):
    del players[ctx.guild.id]
    await ctx.channel.send("*-*")



@bot.command()
async def clear(ctx):
    del players[ctx.guild.id]
    await ctx.channel.send("clear!")










  

###########################################################################################################
#แสดงกิจกรรม


@bot.event
async def on_ready():
    await bot.change_presence(activity=discord.Game(name=f"osu!"))

async  def ch_pr():
  await client.wait_until_ready()

  statuses = ["a game" , f"osu!"]

  while not client.is_closed():

    status = random.choice(statuses)

    await client  .change_presence(activity=discord.Game(name=status))

    await asyncio.slepp(10)

client.loop.create_task(ch_pr())


###########################################################################################################


keep_alive()
token = os.environ.get("DISCORD_BOT_SECRET")


bot.run('OTAyMTc5ODkyOTUzNzcyMDkz.Gp0xkv._QX__usxjxlzOE1EdmUOd4UuYt1Ah2JFOLHSjE')
