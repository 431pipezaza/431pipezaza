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


bot.run('..token your bot discord..')
