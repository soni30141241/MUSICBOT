#
# Copyright (C) 2021-2022 by TheAloneteam@Github, < https://github.com/TheAloneTeam >.
#
# All rights reserved.

import asyncio
import os
import re
import json
import glob
import random
import logging
import urllib.parse
import time
from typing import Union

import httpx
import yt_dlp

# --- Environment Configuration ---
API_URL = os.getenv("API_URL", "http://localhost:8000").rstrip("/")
API_KEY = os.getenv("API_KEY", "")

# --- Dynamic Compatibility / Fallbacks for Environment Safety ---
try:
    from pyrogram.enums import MessageEntityType
    from pyrogram.types import Message
except ImportError:
    class MessageEntityType:
        URL = "url"
        TEXT_LINK = "text_link"
    class Message:
        pass

try:
    from youtubesearchpython.__future__ import VideosSearch, Playlist
except ImportError:
    VideosSearch = None
    Playlist = None

try:
    from AviaxMusic.utils.database import is_on_off
except ImportError:
    async def is_on_off(*args, **kwargs):
        return True

try:
    from AviaxMusic.utils.formatters import time_to_seconds
except ImportError:
    def time_to_seconds(time_str: str) -> int:
        if not time_str:
            return 0
        try:
            parts = list(map(int, time_str.split(":")))
            if len(parts) == 3:
                return parts[0] * 3600 + parts[1] * 60 + parts[2]
            elif len(parts) == 2:
                return parts[0] * 60 + parts[1]
            elif len(parts) == 1:
                return parts[0]
        except Exception:
            pass
        return 0


# --- Original Local Helper Functions (Preserved for compatibility and fallback) ---

def cookie_txt_file():
    folder_path = f"{os.getcwd()}/cookies"
    filename = f"{os.getcwd()}/cookies/logs.csv"
    txt_files = glob.glob(os.path.join(folder_path, '*.txt'))
    if not txt_files:
        raise FileNotFoundError("No .txt files found in the specified folder.")
    cookie_txt_file = random.choice(txt_files)
    with open(filename, 'a') as file:
        file.write(f'Choosen File : {cookie_txt_file}\n')
    return f"cookies/{str(cookie_txt_file).split('/')[-1]}"


async def check_file_size(link):
    async def get_format_info(link):
        proc = await asyncio.create_subprocess_exec(
            "yt-dlp",
            "--cookies", cookie_txt_file(),
            "-J",
            link,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE
        )
        stdout, stderr = await proc.communicate()
        if proc.returncode != 0:
            print(f'Error:\n{stderr.decode()}')
            return None
        return json.loads(stdout.decode())

    def parse_size(formats):
        total_size = 0
        for format in formats:
            if 'filesize' in format:
                total_size += format['filesize']
        return total_size

    info = await get_format_info(link)
    if info is None:
        return None
    
    formats = info.get('formats', [])
    if not formats:
        print("No formats found.")
        return None
    
    total_size = parse_size(formats)
    return total_size


async def shell_cmd(cmd):
    proc = await asyncio.create_subprocess_shell(
        cmd,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
    )
    out, errorz = await proc.communicate()
    if errorz:
        if "unavailable videos are hidden" in (errorz.decode("utf-8")).lower():
            return out.decode("utf-8")
        else:
            return errorz.decode("utf-8")
    return out.decode("utf-8")


# --- Utility Functions ---

def clean_link(link: str) -> str:
    if not link:
        return ""
    link = str(link)
    if "&" in link:
        link = link.split("&")[0]
    if "?si=" in link:
        link = link.split("?si=")[0]
    elif "&si=" in link:
        link = link.split("&si=")[0]
    return link


def extract_vidid(query: str) -> str:
    if not query:
        return None
    if re.match(r"^[a-zA-Z0-9_-]{11}$", query):
        return query
    regex = r"(?:youtube\.com\/(?:[^\/]+\/.+\/|(?:v|e(?:mbed)?)\/|shorts\/|.*[?&]v=)|youtu\.be\/)([^\"&?\/\s]{11})"
    match = re.search(regex, query)
    return match.group(1) if match else None


async def download_assistant(query: str, dl_type: str) -> str:
    """Helper to get stream URL from the API"""
    safe_query = urllib.parse.quote(query)
    ext = "mp3" if dl_type == "audio" else "mp4"
    if API_KEY:
        url = f"{API_URL}/downloads/{API_KEY}/{safe_query}.{ext}"
    else:
        url = f"{API_URL}/downloads/stream?query={safe_query}&dl_type={dl_type}"
    return url


async def _api_get(endpoint: str, params: dict = None, timeout: float = 30.0) -> Union[dict, list, None]:
    if not API_URL:
        return None
    url = f"{API_URL}/{endpoint.lstrip('/')}"
    if params is None:
        params = {}
    if API_KEY:
        params["api_key"] = API_KEY
    try:
        async with httpx.AsyncClient(timeout=timeout) as client:
            response = await client.get(url, params=params)
            if response.status_code == 200:
                return response.json()
    except Exception as e:
        logging.warning(f"API call to {endpoint} failed: {e}")
    return None


class YouTubeAPI:
    def __init__(self):
        self.base = "https://www.youtube.com/watch?v="
        self._recent_prefetches = {} # vidid -> timestamp
        self.regex = r"(?:youtube\.com|youtu\.be)"
        self.status = "https://www.youtube.com/oembed?url="
        self.listbase = "https://youtube.com/playlist?list="
        self.reg = re.compile(r"\x1B(?:[@-Z\\-_]|\[[0-?]*[ -/]*[@-~])")

    async def exists(self, link: str, videoid: Union[bool, str] = None):
        if videoid:
            link = self.base + link
        return bool(re.search(self.regex, link))

    async def url(self, message_1: Message) -> Union[str, None]:
        messages = [message_1]
        if message_1.reply_to_message:
            messages.append(message_1.reply_to_message)
        for message in messages:
            if getattr(message, "entities", None):
                for entity in message.entities:
                    if entity.type == MessageEntityType.URL:
                        text = message.text or message.caption
                        return text[entity.offset: entity.offset + entity.length]
            elif getattr(message, "caption_entities", None):
                for entity in message.caption_entities:
                    if entity.type == MessageEntityType.TEXT_LINK:
                        return entity.url
        return None

    async def details(self, link: str, videoid: Union[bool, str] = None):
        if videoid:
            link = self.base + link
        link = clean_link(link)
        # API first
        data = await _api_get("details", {"link": link})
        if data:
            return (
                data.get("title"),
                data.get("duration_min"),
                data.get("duration_sec", 0),
                data.get("thumbnail"),
                data.get("vidid")
            )
        # Fallback to local
        if VideosSearch:
            try:
                results = VideosSearch(link, limit=1)
                res = await results.next()
                if res and res.get("result"):
                    result = res["result"][0]
                    title = result["title"]
                    duration_min = result["duration"]
                    thumbnail = result["thumbnails"][0]["url"].split("?")[0]
                    vidid = result["id"]
                    duration_sec = int(time_to_seconds(duration_min)) if duration_min and duration_min != "None" else 0
                    return title, duration_min, duration_sec, thumbnail, vidid
            except Exception as e:
                logging.warning(f"Local VideosSearch fallback failed in details: {e}")
        return None, None, 0, None, None

    async def title(self, link: str, videoid: Union[bool, str] = None):
        if videoid:
            link = self.base + link
        link = clean_link(link)
        # API first
        data = await _api_get("details", {"link": link})
        if data and data.get("title"):
            return data["title"]
        # Fallback to local
        if VideosSearch:
            try:
                results = VideosSearch(link, limit=1)
                res = await results.next()
                if res and res.get("result"):
                    return res["result"][0]["title"]
            except Exception as e:
                logging.warning(f"Local VideosSearch fallback failed in title: {e}")
        return None

    async def duration(self, link: str, videoid: Union[bool, str] = None):
        if videoid:
            link = self.base + link
        link = clean_link(link)
        # API first
        data = await _api_get("details", {"link": link})
        if data and data.get("duration_min"):
            return data["duration_min"]
        # Fallback to local
        if VideosSearch:
            try:
                results = VideosSearch(link, limit=1)
                res = await results.next()
                if res and res.get("result"):
                    return res["result"][0]["duration"]
            except Exception as e:
                logging.warning(f"Local VideosSearch fallback failed in duration: {e}")
        return None

    async def thumbnail(self, link: str, videoid: Union[bool, str] = None):
        if videoid:
            link = self.base + link
        link = clean_link(link)
        # API first
        data = await _api_get("details", {"link": link})
        if data and data.get("thumbnail"):
            return data["thumbnail"]
        # Fallback to local
        if VideosSearch:
            try:
                results = VideosSearch(link, limit=1)
                res = await results.next()
                if res and res.get("result"):
                    return res["result"][0]["thumbnails"][0]["url"].split("?")[0]
            except Exception as e:
                logging.warning(f"Local VideosSearch fallback failed in thumbnail: {e}")
        return None

    async def video(self, link: str, videoid: Union[bool, str] = None):
        if videoid:
            link = self.base + link
        link = clean_link(link)
        # API first: construct API streaming URL
        try:
            vidid_extracted = extract_vidid(link) or link
            if API_URL:
                if API_KEY:
                    stream_url = f"{API_URL}/downloads/{API_KEY}/youtube.com/{vidid_extracted}.mp4"
                else:
                    stream_url = f"{API_URL}/downloads/youtube.com/{vidid_extracted}.mp4"
                return 1, stream_url
        except Exception as e:
            logging.warning(f"Constructing API video url failed: {e}")
        
        # Fallback to local yt-dlp -g
        try:
            cookies = cookie_txt_file()
            cmd = ["yt-dlp", "--cookies", cookies, "-g", "-f", "best[height<=?720][width<=?1280]", link]
            proc = await asyncio.create_subprocess_exec(
                *cmd,
                stdout=asyncio.subprocess.PIPE,
                stderr=asyncio.subprocess.PIPE,
            )
            stdout, stderr = await proc.communicate()
            if stdout:
                return 1, stdout.decode().split("\n")[0]
            else:
                return 0, stderr.decode()
        except Exception as e:
            return 0, str(e)

    async def playlist(self, link, limit, user_id, videoid: Union[bool, str] = None):
        if videoid:
            link = self.listbase + link
        link = clean_link(link)
        # API first
        data = await _api_get("playlist", {"link": link, "limit": limit})
        if data and isinstance(data, dict) and "videos" in data:
            videos = data["videos"]
            return [v.get("vidid") for v in videos if v.get("vidid")]
        
        # Fallback to local shell_cmd
        try:
            cookies = cookie_txt_file()
            playlist_out = await shell_cmd(
                f"yt-dlp -i --get-id --flat-playlist --cookies {cookies} --playlist-end {limit} --skip-download {link}"
            )
            result = playlist_out.split("\n")
            result = [k for k in result if k != ""]
            return result
        except Exception as e:
            logging.warning(f"Local playlist fallback failed: {e}")
        return []

    async def track(self, link: str, videoid: Union[bool, str] = None):
        if videoid:
            link = self.base + link
        link = clean_link(link)
        # API first
        data = await _api_get("details", {"link": link})
        if data:
            track_details = {
                "title": data.get("title"),
                "link": data.get("link"),
                "vidid": data.get("vidid"),
                "duration_min": data.get("duration_min"),
                "thumb": data.get("thumbnail"),
            }
            return track_details, data.get("vidid")
        
        # Fallback to local
        if VideosSearch:
            try:
                results = VideosSearch(link, limit=1)
                res = await results.next()
                if res and res.get("result"):
                    result = res["result"][0]
                    track_details = {
                        "title": result["title"],
                        "link": result["link"],
                        "vidid": result["id"],
                        "duration_min": result["duration"],
                        "thumb": result["thumbnails"][0]["url"].split("?")[0],
                    }
                    return track_details, result["id"]
            except Exception as e:
                logging.warning(f"Local track fallback failed: {e}")
        return None, None

    async def formats(self, link: str, videoid: Union[bool, str] = None):
        if videoid:
            link = self.base + link
        link = clean_link(link)
        # Local formats extraction
        def _extract():
            ytdl_opts = {"quiet": True}
            try:
                ytdl_opts["cookiefile"] = cookie_txt_file()
            except Exception:
                pass
            ydl = yt_dlp.YoutubeDL(ytdl_opts)
            with ydl:
                return ydl.extract_info(link, download=False)

        try:
            r = await asyncio.to_thread(_extract)
            formats_available = []
            for format in r.get("formats", []):
                try:
                    if "dash" not in str(format.get("format", "")).lower():
                        formats_available.append(
                            {
                                "format": format.get("format"),
                                "filesize": format.get("filesize"),
                                "format_id": format.get("format_id"),
                                "ext": format.get("ext"),
                                "format_note": format.get("format_note"),
                                "yturl": link,
                            }
                        )
                except Exception:
                    continue
            return formats_available, link
        except Exception as e:
            logging.warning(f"Formats extraction failed: {e}")
            return [], link

    async def slider(self, link: str, query_type: int, videoid: Union[bool, str] = None):
        if videoid:
            link = self.base + link
        link = clean_link(link)
        # API first
        data = await _api_get("search", {"query": link, "limit": 10})
        if data and isinstance(data, dict) and "result" in data:
            result = data["result"]
            if result and len(result) > query_type:
                target = result[query_type]
                title = target["title"]
                duration_min = target["duration"]
                vidid = target["id"]
                thumbnail = target["thumbnails"][0]["url"].split("?")[0] if target.get("thumbnails") else None
                return title, duration_min, thumbnail, vidid
        
        # Fallback to local VideosSearch
        if VideosSearch:
            try:
                a = VideosSearch(link, limit=10)
                res = await a.next()
                result = res.get("result")
                if result and len(result) > query_type:
                    title = result[query_type]["title"]
                    duration_min = result[query_type]["duration"]
                    vidid = result[query_type]["id"]
                    thumbnail = result[query_type]["thumbnails"][0]["url"].split("?")[0]
                    return title, duration_min, thumbnail, vidid
            except Exception as e:
                logging.warning(f"Local slider fallback failed: {e}")
        return None, None, None, None

    async def prefetch(self, link: str, video: bool = False):
        """Triggers background pre-fetching on the API"""
        if not API_URL:
            return False
        dl_type = "video" if video else "audio"
        link = clean_link(link)

        # Avoid redundant prefetches within 30 seconds
        now = time.time()
        vidid = extract_vidid(link) or link

        cache_key = f"{vidid}_{dl_type}"
        if cache_key in self._recent_prefetches:
            if now - self._recent_prefetches[cache_key] < 30:
                return True

        self._recent_prefetches[cache_key] = now

        # Cleanup old prefetches (keep cache small)
        if len(self._recent_prefetches) > 100:
            self._recent_prefetches = {k: v for k, v in self._recent_prefetches.items() if now - v < 300}

        params = {"query": link, "dl_type": dl_type, "prefetch": "true"}
        if API_KEY:
            params["api_key"] = API_KEY
        try:
            async with httpx.AsyncClient(timeout=10.0) as client:
                await client.get(f"{API_URL}/download", params=params)
                return True
        except Exception as e:
            logging.warning(f"Prefetch failed for {link}: {e}")
        return False

    async def prefetch_queue(self, queries: list, video: bool = False):
        """Triggers bulk background pre-fetching on the API for a queue"""
        if not API_URL or not queries:
            return False
        dl_type = "video" if video else "audio"
        payload = {"queries": queries, "dl_type": dl_type}
        params = {}
        if API_KEY:
            params["api_key"] = API_KEY

        try:
            async with httpx.AsyncClient(timeout=15.0) as client:
                await client.post(f"{API_URL}/prefetch_bulk", json=payload, params=params)
                return True
        except Exception as e:
            logging.warning(f"Bulk prefetch failed: {e}")
        return False

    async def download(
        self,
        link: str,
        mystic,
        video: Union[bool, str] = None,
        videoid: Union[bool, str] = None,
        songaudio: Union[bool, str] = None,
        songvideo: Union[bool, str] = None,
        format_id: Union[bool, str] = None,
        title: Union[bool, str] = None,
    ) -> Union[str, tuple]:
        if videoid:
            link = self.base + link
        
        # Helper to download from API
        async def download_from_api(query_link: str, dl_type: str, filepath: str) -> bool:
            if not API_URL:
                return False
            vidid_extracted = extract_vidid(query_link) or query_link
            params = {"query": vidid_extracted, "dl_type": dl_type}
            if API_KEY:
                params["api_key"] = API_KEY
            os.makedirs(os.path.dirname(filepath), exist_ok=True)
            try:
                async with httpx.AsyncClient(timeout=600.0) as client:
                    async with client.stream("GET", f"{API_URL}/download", params=params) as resp:
                        if resp.status_code != 200:
                            return False
                        with open(filepath, "wb") as f:
                            async for chunk in resp.aiter_bytes(131072):
                                f.write(chunk)
                return os.path.exists(filepath) and os.path.getsize(filepath) > 0
            except Exception as e:
                logging.warning(f"API download failed for {vidid_extracted}: {e}")
                if os.path.exists(filepath):
                    try: os.remove(filepath)
                    except Exception: pass
                return False

        # API-first Streaming Strategy (Works exactly like youtube.py)
        if API_URL:
            dl_type = "video" if (video or songvideo) else "audio"
            link = clean_link(link)
            vidid_extracted = extract_vidid(link)

            # Local download required only for local song uploads (songvideo / songaudio)
            if songvideo:
                fpath = f"downloads/{title}.mp4"
                success = await download_from_api(link, "video", fpath)
                if success:
                    return fpath
            elif songaudio:
                fpath = f"downloads/{title}.mp3"
                success = await download_from_api(link, "audio", fpath)
                if success:
                    return fpath
            else:
                # General audio / video play: return the direct stream URL from API!
                if vidid_extracted:
                    ext = "mp4" if dl_type == "video" else "mp3"
                    # Background prefetch to warm cache
                    asyncio.create_task(self.prefetch(link, video=bool(dl_type == "video")))
                    if API_KEY:
                        stream_url = f"{API_URL}/downloads/{API_KEY}/youtube.com/{vidid_extracted}.{ext}"
                    else:
                        stream_url = f"{API_URL}/downloads/youtube.com/{vidid_extracted}.{ext}"
                else:
                    stream_url = await download_assistant(link, dl_type)
                
                return stream_url, True

        # Local Fallbacks (user's original implementation)
        loop = asyncio.get_running_loop()
        def audio_dl():
            ydl_optssx = {
                "format": "bestaudio/best",
                "outtmpl": "downloads/%(id)s.%(ext)s",
                "geo_bypass": True,
                "nocheckcertificate": True,
                "quiet": True,
                "cookiefile" : cookie_txt_file(),
                "no_warnings": True,
            }
            x = yt_dlp.YoutubeDL(ydl_optssx)
            info = x.extract_info(link, False)
            xyz = os.path.join("downloads", f"{info['id']}.{info['ext']}")
            if os.path.exists(xyz):
                return xyz
            x.download([link])
            return xyz

        def video_dl():
            ydl_optssx = {
                "format": "(bestvideo[height<=?720][width<=?1280][ext=mp4])+(bestaudio[ext=m4a])",
                "outtmpl": "downloads/%(id)s.%(ext)s",
                "geo_bypass": True,
                "nocheckcertificate": True,
                "quiet": True,
                "cookiefile" : cookie_txt_file(),
                "no_warnings": True,
            }
            x = yt_dlp.YoutubeDL(ydl_optssx)
            info = x.extract_info(link, False)
            xyz = os.path.join("downloads", f"{info['id']}.{info['ext']}")
            if os.path.exists(xyz):
                return xyz
            x.download([link])
            return xyz

        def song_video_dl():
            formats = f"{format_id}+140"
            fpath = f"downloads/{title}"
            ydl_optssx = {
                "format": formats,
                "outtmpl": fpath,
                "geo_bypass": True,
                "nocheckcertificate": True,
                "quiet": True,
                "no_warnings": True,
                "cookiefile" : cookie_txt_file(),
                "prefer_ffmpeg": True,
                "merge_output_format": "mp4",
            }
            x = yt_dlp.YoutubeDL(ydl_optssx)
            x.download([link])

        def song_audio_dl():
            fpath = f"downloads/{title}.%(ext)s"
            ydl_optssx = {
                "format": format_id,
                "outtmpl": fpath,
                "geo_bypass": True,
                "nocheckcertificate": True,
                "quiet": True,
                "no_warnings": True,
                "cookiefile" : cookie_txt_file(),
                "prefer_ffmpeg": True,
                "postprocessors": [
                    {
                        "key": "FFmpegExtractAudio",
                        "preferredcodec": "mp3",
                        "preferredquality": "192",
                    }
                ],
            }
            x = yt_dlp.YoutubeDL(ydl_optssx)
            x.download([link])

        if songvideo:
            await loop.run_in_executor(None, song_video_dl)
            fpath = f"downloads/{title}.mp4"
            return fpath
        elif songaudio:
            await loop.run_in_executor(None, song_audio_dl)
            fpath = f"downloads/{title}.mp3"
            return fpath
        elif video:
            if await is_on_off(1):
                direct = True
                downloaded_file = await loop.run_in_executor(None, video_dl)
            else:
                proc = await asyncio.create_subprocess_exec(
                    "yt-dlp",
                    "--cookies",cookie_txt_file(),
                    "-g",
                    "-f",
                    "best[height<=?720][width<=?1280]",
                    f"{link}",
                    stdout=asyncio.subprocess.PIPE,
                    stderr=asyncio.subprocess.PIPE,
                )
                stdout, stderr = await proc.communicate()
                if stdout:
                    downloaded_file = stdout.decode().split("\n")[0]
                    direct = False
                else:
                   file_size = await check_file_size(link)
                   if not file_size:
                     print("None file Size")
                     return
                   total_size_mb = file_size / (1024 * 1024)
                   if total_size_mb > 250:
                     print(f"File size {total_size_mb:.2f} MB exceeds the 100MB limit.")
                     return None
                   direct = True
                   downloaded_file = await loop.run_in_executor(None, video_dl)
            return downloaded_file, direct
        else:
            direct = True
            downloaded_file = await loop.run_in_executor(None, audio_dl)
            return downloaded_file, direct


YouTube = YouTubeAPI()
