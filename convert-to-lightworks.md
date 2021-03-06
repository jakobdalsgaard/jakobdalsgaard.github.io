Convert to Lightworks

The [Lightworks](https://www.lwks.com/) application is a very nice [non-linear video editing system](https://en.wikipedia.org/wiki/Non-linear_editing_system) -
and, when 720p is sufficient, free of charge! However, it's a bit picky on import
formats. I've found the easiest approach to this is installing [FFMpeg](https://ffmpeg.org/download.html)
and batch converting everything you want to import to mpegts, FullHD, 46kHz stereo.

![Lightworks red shark](media/lightworks-redshark.png)

To do so, I've created a script, should work on all unices, available here: [convert-to-lightworks](http://convert-to-lightworks.dalsgaard.net/) 
-- and also a .bat file for Windows lusers, available here: [convert-to-lightworks.bat](http://convert-to-lightworks-bat.dalsgaard.net/).
They should both support just dropping files onto them from a file manager.

To installl ffmpeg on Elementary OS Freya release, do this:

    sudo cat "deb http://www.deb-multimedia.org wheezy main" > /etc/apt/sources.list.d/deb-multimedia.list
    sudo apt-get update
    sudo apt-get install deb-multimedia-keyring
    sudo apt-get install ffmpeg

Tags: computer, video, software
