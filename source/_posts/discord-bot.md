---
title: A discord bot for Bilibili
date: 2018-06-14 11:09:03
tags:
---

GitHub: [DiscordBilibiliBot](https://github.com/criyle/DiscordBilibiliBot)

After about one month of development, this discord bot finally have the expected behavior, but it still need some polishment.

## Reason for developing such a bot

There are lots of discord bots that plays YouTube wideos as audio in discord audio channel, but there were not decent ones for Bilibili. Since I am a fan of Vocaloid China mainly posted on Bilibili, I started writing this bot.

<!--more-->

Why Python?

~~Python No. 1. LOL~~ Python is a script-based language that can scale easily and there is also HTTP libraries

## Details

I have spend plenty of time to reverse engineering the JavaScript for the `video.js` for the bilibili video site, but it truns out the 40k lines of JavaScript used many anti reverse engineering strategies, such as renaming and virtualization.

After I gave up, I googled for the `APP_KEY` used by bilibili and I found the `APP_SECRET` directly in `youtube_dl` project... Curretly, bilibili is tring to speed up loading time, the video address is embaded into HTML, initialed by `window.__playinfo__`...

The current process to download the video is:

Download HTML -> Get `window.__INITIAL_STATE__` to parse `cid` -> interface api get video url

And to convert video into audio, used `ffmpeg` to decode and `atomicparsley` to add meta data.

All network operations are done by asynchronized calls(`asyncio`, `aiohttp`), and blocking operations are done by multithreading(`threading`)

Last, using what I have learnt in the past, I wrote `fabric` for deployment and `supervisor` for deamon runner

## In the Last

It is much easier than what I have thought. Maybe I need some courage to start...
