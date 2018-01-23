+++
title = "OBS Scripting in Python"
description = "Testing out the new python scripting system in OBS"
tags = []
categories = []
date: 2018-01-23T01:34:53-06:00
+++

OBS just released version 21.0.1. This release includes a new subsystem for python and lua scripts.

In order to use the scripting system, we have to recompile (for arch anyways).


Install prereqs(from pkgbuild):

```
# pacman -Sy cmake git libfdk-aac libxcomposite x264 jack vlc

# pacman -Sy ffmpeg jansson libxinerama libxkbcommon-x11 qt5-x11extras curl gtk-update-icon-cache
```


Even the pkgbuild from upstream git doesn't have scripting, even though it has a high version.


```
volt% /usr/bin/obs --version
OBS Studio - 21.0.1.r0.g4eb5be49 (linux)
volt% ~/local/bin/obs --version
OBS Studio - 21.0.1 (linux)

```

Versions from AUR and self-compiled respectively.

From the developer docs ( https://obsproject.com/docs/scripting.html) 

"Scripting can be accessed in OBS Studio via the Tools menu -> Scripts option, which will bring up the scripting dialog. Scripts can be added, removed, and reloaded in real time while the program is running."


This annoyingly means we'll be playing with drop-downs to figure out if scripting is active. Some folks on the #obs-dev channel in irc gave me some breadcrumbs to follow.


```
08:56:51         Dillon | nibalizer - You need to configure CMake to be pointed at Python and Luajit.
08:57:45         Dillon | Python is relatively simple, but Luajit is a bit more of a pain, requiring you to compile Lua and Luajit.
09:07:31          Dilaz | there was also some cmake flag for scripting
11:48:20         sophie | nibalizer: after installing swig, libluajit-5.1-dev, and  python3-dev my linux build now has scripting
```

Thanks!


```
pacman -Sy luajit
pacman -Sy swig
pacman -Sy python
```


volt% cat nibz_cmake.log| grep Script
-- Scripting: Luajit supported
-- Scripting: Python 3 supported

After this I was reasonably sure I had compiled obs correctly, but it still didn't work. I eventually realized (by banging around with ``ldd`` that the system libraries like ``libobs.so.1`` were being loaded before the local libraries stored in ``~/local``. I ran ``pacman -R obs-studio-git`` to remove obs and its libraries from the system entirely. Then I used ``LD_LIBRARY_PATH`` to explicitly set which libraries I wanted involved.


``
$ LD_LIBRARY_PATH=~/local/lib /home/nibz/local/bin/obs
Attempted path: share/obs/obs-studio/locale/en-US.ini
Attempted path: /home/nibz/local/share/obs/obs-studio/locale/en-US.ini
Attempted path: share/obs/obs-studio/locale.ini
Attempted path: /home/nibz/local/share/obs/obs-studio/locale.ini
Attempted path: share/obs/obs-studio/themes/Default.qss
Attempted path: /home/nibz/local/share/obs/obs-studio/themes/Default.qss
Attempted path: share/obs/obs-studio/license/gplv2.txt
Attempted path: /home/nibz/local/share/obs/obs-studio/license/gplv2.txt
info: CPU Name: Intel(R) Core(TM) i5-5200U CPU @ 2.20GHz
info: CPU Speed: 2194.820MHz
info: Physical Cores: 2, Logical Cores: 4
info: Physical Memory: 7859MB Total, 1790MB Free
info: Kernel Version: Linux 4.14.12-1-ARCH
info: Distribution: "Arch Linux" Unknown
info: Portable mode: false
QMetaObject::connectSlotsByName: No matching signal for on_advAudioProps_clicked()
QMetaObject::connectSlotsByName: No matching signal for on_advAudioProps_destroyed()
QMetaObject::connectSlotsByName: No matching signal for on_program_customContextMenuRequested(QPoint)
info: OBS 21.0.1 (linux)
info: ---------------------------------
info: ---------------------------------
info: audio settings reset:
        samples per sec: 44100
        speakers:        2
info: ---------------------------------
info: Initializing OpenGL...
info: OpenGL version: 4.5 (Core Profile) Mesa 17.3.2
info: ---------------------------------
info: video settings reset:
        base resolution:   1920x1080
        output resolution: 1280x720
        downscale filter:  Bicubic

<snip>
``

And just like that, I had scripting enabled. I tried out the example script Jim provided with the source release. It worked. I copied it and made some modifications, loaded that and it worked fine.

My first super trivial script is here: https://gist.github.com/nibalizer/a6649abee758da3f8d08ef5e164b524c
It just cycles a text source through the names of different Weasley children.

The python scripting system is now all ready to go on my laptop, all I need now is a nontrivial goal and I should be set to start really hacking.

