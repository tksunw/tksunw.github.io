---
title: "pyTivo on Solaris 11.2"
date: 2015-04-06 12:00:00 -0500
categories: [Solaris]
tags: [solaris, tivo, pytivo, ffmpeg]
---

We are a TiVo household, so a quest has been underway to build a suitable place for long term storage of the family's favorite TV shows and movies. One indisputable requirement is that the shows and movies have to be visible via the TiVo menu. pyTivo (the William McBrine fork) is the logical tool to do this (in my house). William McBrine has been maintaining his fork of pyTivo more regularly than the original package (sourceforge).

To get pyTivo working on Solaris 11.2, only 2 dependencies needed to be resolved:

- I needed to build ffmpeg to support on-the-fly video transcoding, and
- ffmpeg wanted yasm (an open source rewrite of the nasm assembler) or nasm itself.

This is what happened when I tried to build ffmpeg without yasm:

```text
bash-[121]$ ./configure --prefix=/usr/local
yasm/nasm not found or too old. Use --disable-yasm for a crippled build.

If you think configure made a mistake, make sure you are using the latest
version from Git. If the latest version fails, report the problem to the
ffmpeg-user@ffmpeg.org mailing list or IRC #ffmpeg on irc.freenode.net.
Include the log file "config.log" produced by configure as this will help
solve the problem.
```

Both were very simple and straight forward to build and install:

### Yasm

```bash
./configure --prefix=/usr/local
gmake -j4
sudo gmake install
```

### ffmpeg

```bash
./configure --prefix=/usr/local
gmake -j4
sudo gmake install
```

Once ffmpeg was installed, I updated the `pyTivo.conf` file and set the location of the ffmpeg binary, and pyTivo worked beautifully after that.

```ini
# FFmpeg is a required tool but downloaded separately. See pyTivo wiki
# for help.
# Full path to ffmpeg including filename
# For windows: ffmpeg=C:\pyTivo\bin\ffmpeg.exe
# For linux:   ffmpeg=/usr/bin/ffmpeg
#ffmpeg=C:\pyTivo\bin\ffmpeg.exe
ffmpeg=/usr/local/bin/ffmpeg
```

For more information on pyTivo, including installation, configuration, and other tasks outside the purpose of this post:

- The pyTivo Wiki, which has gobs of information, and lots of links, and instructions.
- The pyTivo Discussion Forum on SourceForge, which is still active.
- And, the TiVo Community Forum thread that is still active and being updated.
