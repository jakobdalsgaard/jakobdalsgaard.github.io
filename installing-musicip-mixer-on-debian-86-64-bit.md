Installing MusicIP Mixer on Debian 8.6 64-bit

In order to have automatic music mixing on a Logitech Media Server (former
Slimserver...), now a days you'll need the Music IP Mixer x86 installed. Running 
that beast on a 64-bit Debian 8.6 requires a bit of configuration.

First; make your platform support x86 (32-bit) binaries:

~~~~
sudo dpkg --add-architecture i386
sudo apt-get update
sudo apt-get install libgcc1:i386
~~~~

Now, download the MusicIP software, version 1.8 from the Spicefly Sugarcube homepage:

[Spicefly MusicIP Software](http://www.spicefly.com/article.php?page=musicip-software).

I then created a user for the MusicIP software, to keep things separated:

~~~~~
sudo user-add -m --system musicip
~~~~~

Now, in the software bundle for Linux, there is an init script called `mmserver` -- 
just copy that one to `/etc/init.d` make it owned by root, and change the username therein
to `musicip` and the music home to `/home/musicip/MusicMagicMixer/` (remember the trailing slash).

You can now start the MusicMagicMixer with:

~~~~
/etc/init.d/mmserver start
~~~~

It prints a wee bit on the terminal, but all is good. On all interfaces MusicMagicMixer is
now responding to port 10002. You can point your browser to it, and you'll have a fine
web UI. You can make it even finer if you download and install the Spicefly Sugarcube MusicIP
replacement UI: [MusicIP Replacement UI](http://www.spicefly.com/article.php?page=musicip-headless-interface).

Please support software, buy your license to Spicefly Sugarcube: [SugarCube Licensing](http://www.spicefly.com/licensing/).

Tags: slimserver, squeezebox, logitech, software, debian, linux

