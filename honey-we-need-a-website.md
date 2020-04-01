Honey, we need a website

Being a new member of the [Danish Beekeepers Association](https://www.biavl.dk/) I get 100 free stickers to put on honey that I can actually, legally sell (the insurance of the association will be covering, it's actually a pretty good deal). Nowadays they can put a QR encoded URL on the stickers, and off course I opted for that. Now I _just_ need to build a website. Some might go straight to [one.com](https://www.one.com/) and get a hosted static website. But I can do so much better than that.

# Security

When it comes to websites for honey, you want top class security -- you cannot risk anyone eavesdropping on your honey website browsing.

## Support the right protocols

Site needs to be TLS, really, it does, and I want TLS1.3. This is not as simple as it sounds. My Debian box was running Debian Jessie, and it seemed to be a good idea to move onto Debian Stretch. First I did `apt-get update` and `apt-get upgrade` to make sure the Debian Jessie was as good as it could be, then `/etc/apt/sources.list` was updated to have stretch specified instead of jessie:

    # deb http://ftp.dk.debian.org/debian/ jessie main
    # deb-src http://ftp.dk.debian.org/debian/ jessie main
    
    deb http://ftp.dk.debian.org/debian/ stretch main
    deb-src http://ftp.dk.debian.org/debian/ stretch main
    
    # deb http://security.debian.org/ jessie/updates main
    # deb-src http://security.debian.org/ jessie/updates main

    deb http://security.debian.org/ stretch/updates main
    deb-src http://security.debian.org/ stretch/updates main

With that setup in place, yet another round of `apt-get upgrade` + `apt-get update` could be done. Finally an `apt-get dist-upgrade` and a reboot. I now had Debian Stretch. But I can do so much better than that.

Note: An alternative would be to install Debian Stretch from scratch, wiping the system -- I would prefer that... not only would I be able to repartition the system, but there would probably be much less kipple on the system. That'll have to wait for some time in the not so near future where I have much more time. When reinstalling a home linux box I always make sure to install [fail2ban](https://www.fail2ban.org/) -- it's a nice safeguard.

I'll go for the nginx webserver, since it has a long tradition of being async epoll based, and according to Linux Journal, even [youporn](https://www.linuxjournal.com/article/10108) uses it, so why not me. It also comes highly recommended by [syshero](https://syshero.org/). My use case is serving static files with a bit of logic on figuring out which file should have what headers, but not any server side processes that resembles any significant business logic -- my use case is basically: shove as many bytes over the wire to as many clients as possible, as securely and efficient as possible. For that nginx seems like a fine choice.

The version of nginx that comes with Debian 9 is 1.10.3 -- a fine version indeed, but since it's compiled with OpenSSL in a pre-1.1.0j version, it cannot do the TLS1.3 that I want. Therefore I need to find another version of nginx, luckily the internet is full of guides on that, I chose [Life on the Net, by Luke NG](https://www.lukeng.net/tlsv1-3-with-nginx-and-openssl-on-debian/) -- scroll down to the section that reads 'Current Nginx mainline version [1.15.5] from nginx.org'. It's pretty simple. Create the file `/etc/apt/sources.list.d/nginx.conf` with the contents:

    deb http://nginx.org/packages/mainline/debian/ stretch nginx
    deb-src http://nginx.org/packages/mainline/debian/ stretch nginx

Then add the nginx public key to your trusted apt keys: `wget https://nginx.org/keys/nginx_signing.key` + `apt-key add nginx_signing.key` and install the package:

    apt-get update
    apt-get install nginx

And then, all of a sudden, running `nginx -V` looks something like this:

    jakob@rack1:~$ /usr/sbin/nginx -V
    nginx version: nginx/1.17.0
    built by gcc 6.3.0 20170516 (Debian 6.3.0-18+deb9u1)
    built with OpenSSL 1.1.0j  20 Nov 2018 (running with OpenSSL 1.1.1c  28 May 2019)
    TLS SNI support enabled

Gr8!

## Get a fine certificate

These days a certificate is not that hard to get, [letsencrypt.org](https://letsencrypt.org) makes it fairly easy to provision a certificate for a domain you own. There are many guides online on how to obtain certificates, and many toolboxes to aid you. I've chosen the [acme.sh](https://acme.sh/) solution -- a shell implemented client that covers my needs. I prefer to acknowledge ownership of the domain through TXT records; the process might be a bit tedious, but I hate to spin up the site insecure first to publish the validation token; I'd rather use DNS for that. I have a dedicated `acme` user on the Debian box, and `acme.sh` installed in that user's homedirectory. So, in order to issue the certificate, I run:

    ./.acme.sh/acme.sh --issue -d www.myhoneywebsite.com --dns --yes-I-know-dns-manual-mode-enough-go-ahead-please

In a lot of terminal output I locate the TXT record name and value, I head over to my DNS admin interface and type it in. Now the certificate can be renewed/issued:

    ./.acme.sh/acme.sh --renew -d www.myhoneywebsite.com --dns --yes-I-know-dns-manual-mode-enough-go-ahead-please

This provides me with a certificate and associated private key; readily configurable in nginx.

# Availability

A modern world calls for modern solutions, so off course this website will be running IPv4 and IPv6 dualstack -- since I only have a single IPv4, it'll be available through SNI (Server Name Identification); due to my vast amount of IPv6 space you don't need SNI to reach the site using IPv6. The nginx that I run does not support any encrypted version of SNI, I believe the standards are yet to be fully specified. That said, SNI also only reveals what site you wish to connect to, it does not expose any of the data you transmit over the socket. Since who ever is spying on you can always see the IP you're connecting to, the enencrypted SNI will only make a difference should that IP be serving many, many sites. Mine does not.

## Get traffic forwarded

I only have one IPv4 number for the entire house and my openwrt router owns it. I thus need to forward port 80 and port 443 to it (I also forward port 22... but that one is not necessary to serve webpages). So I ensure I have these two rules in `/etc/config/firewall` on the router:

    config redirect
            option target 'DNAT'
            option src 'wan'
            option dest 'lan'
            option proto 'tcp'
            option dest_ip '192.168.1.60'
            option dest_port '443'
            option name 'https'
            option src_dport '443'

    config redirect
            option target 'DNAT'
            option src 'wan'
            option dest 'lan'
            option proto 'tcp'
            option src_dport '80'
            option dest_ip '192.168.1.60'
            option dest_port '80'
            option name 'http'

My server has the internal IPv4 number `192.168.1.60` configured statically, no DHCP mumbo jumbo here; `/etc/network/interfaces` contains:

    iface eth0 inet static
            address 192.168.1.60
            netmask 255.255.0.0
            gateway 192.168.1.1

Why? because there's no need for this to change.

Any [serious](https://twitter.com/jakobdalsgaard/status/479169734706225152) business needs IPv6 these days, therefore also my honey website.

My ISP does not provide any IPv6 - which is really, really annoying. Therefore I've signed up at [Hurricane Electric](https://he.com/), done their [IPv6 certification](https://ipv6.he.net/certification/) since [kramse](https://www.zencurity.com/) told me to -- now a Sarge, but still accruing points. Hurrican Electric runs a fine IPv6 tunnel [service](https://tunnelbroker.net/), by which I tunnel IPv6 packets over IPv4 from my house to Frankfurt -- works great. I have my own `::/64` net!

Having my own `::/64` net I want to assign multiple addresses to my main inhouse server. The only way I've been able to make multiple static IPv6 addresses appear consistenly on my Debian boxes is by adding a file `/etc/config/if-up.d/ipv6-up` that contains:

    #!/bin/sh
    # filename: ipv6-up

    if [ "$IFACE" = eth1 ]; then
      echo "Setting ipv6"
      ip -6 addr add 2001:db8:1f0b:dfe:0:0:0:400/64 dev eth1
      ip -6 addr add 2001:db8:1f0b:dfe:0:0:0:800/64 dev eth1
      ip -6 route add default via 2001:db8:1f0b:dfe::1
    fi

This works brilliantly, and I can add as many IPv6 addresses in my `2001:db8:1f0b:dfe::/64` block that I want.

For IPv6 we don't need any NAT (Network Address Translation) to happen, IPv6 works as the internet was intended to work. But we need to have the firewall opened. Thus I make sure that `/etc/config/firewall` on my router contains:

    config rule
            option target 'ACCEPT'
            option src 'wan'
            option name 'https IPv6 to honey'
            option family 'ipv6'
            option proto 'tcp'
            option dest_ip '2001:db8:1f0b:dfe::400'
            option dest_port '443'
            option dest '*'

    config rule
            option target 'ACCEPT'
            option name 'http IPv6 to honey'
            option family 'ipv6'
            option proto 'tcp'
            option dest_ip '2001:db8:1f0b:dfe::400'
            option dest_port '80'
            option dest '*'
            option src '*'

Make sure to commit changes, and restart firewall:

    uci commit firewall
    service firewall restart

Voilá, with `nc` on a machine outside the network (through a mobile subscription for example) and `nc` on the Debian box, connectivity can be tested -- that exercise is left for the reader. 

## Getting nginx configured

Now it's time to configure nginx for this fancy site. I created the file `/etc/nginx/sites-enabled/honey` to hold the configuration; many things must be considered.
### Forward all http to https

Security is a very high priority -- the site will have to respond to _any_ http call with a 301 redirect to the same https. This is accomplished for both IPv4 and IPV6 endpoints by this:

    server {
        listen      192.168.1.60:80;
        listen      [2001:db8:1f0b:dfe:0:0:0:400]:80;
        server_name www.myhoneywebsite.com;
        return 301 https://$server_name$request_uri;
        expires max;

        access_log /var/log/nginx/honey-access.log;
        error_log /var/log/nginx/honey-error.log;
    }

Here's a brief explanation of the `expires max` setting (see [here](http://nginx.org/en/docs/http/ngx_http_headers_module.html#expires)):

> The max parameter sets “Expires” to the value “Thu, 31 Dec 2037 23:55:55 GMT”, and “Cache-Control” to 10 years.

As I'll _never_ change strategy on this redirect, this cache control setting seems fine.

### Caching stuff

I'll be running this website on my home xDSL line -- not the most fancy setup, bandwidth is limited, and my media ambitions are skyhigh, therefore I neeed to setup proper Cache-Control headers for my content. A convenient way of doing this, is to map content types into Cache-Control headers. First a map is defined:

    map $sent_http_content_type $cc_header {
      default                    "no-cache";
      text/html                  "public, max-age=7200";
      text/css                   "public, max-age=7200";
      application/javascript     "public, max-age=14400";
      ~image/                    "public, max-age=14400";
      ~video/                    "public, max-age=31536000";
    }

The cache timeouts are: 7200s (2 hours), 14400s (4 days), 28800s (8 days)  and 31536000s (1 year) -- for most imagery and video the cache timeout can be arbitrarily long, as I can just choose other filenames to bypass caching, but 8 days for pictures and 1 year for videos seem fair. The cache timeout for html pages is set to the lowest, 2 hours, they're probably updated the most _and_ their size is most likely small. In my current setup, the `s-maxage` is not really relevant, as I'm serving https traffic straight from my own server and to whoever is viewing my website, anything in between that can see the content and therefore cache it, would be considered a 'man-in-the-middle' that basically breaks the fundamental concept of privacy through the use of private/public key encryption. I _have_ seen corporate setups with decrypting/re-encrypting proxies for the employees to use; but I cannot stress too much how bad such a setup is. Period.

The magic inside a `server` block is now that you can write:

      add_header Cache-Control $cc_header always;

And have nginx set the relevant value. Once this is put in place, do verify with `curl -I`.

Actually talking about caching: A default nginx will make most browsers issue `If-Modified-Since` requests, even on resources served with cache control headers as specified here. There are two ways of prevent that roundtrip:

1. Add `immutable` to the cache control header, thereby saying that this item _will never_ change. Forcing a refresh in the browser will actually make it reload the resource anyway; so much for being immutable.
2. Remove the `etag` from the responses. The `etag` is sort of a hash value for the resource, and is by default shipped along -- in the absence of this header most browser solely rely on the cache control header and will _not_ issue `If-Modified-Since` requests.

Less requests means more bandwidth to all the other eager honey web surfers out there; I opt for version 2 above, since my content is not really immutable. Thus, right before the `add_header` line, I'll add:

    etag off;

Then I get less requests.

Oh, I'll be serving static content, might as well just set a document root and be done with, default file is index.html and default character set is UTF-8 (after all, we no longer live in the 1970's):

    root /home/www-home/honning;
    index index.html
    charset UTF-8;


### Doing https traffic

So, we need TLS -- first, nginx must listen on the right IPs and ports:

    server_name www.myhoneywebsite.com;
    listen 192.168.1.60:443 ssl;
    listen [2001:db8:1f0b:dfe:0:0:0:400]:443 ssl;

All good, then, since this nginx is serving multiple sites, make sure logging is sane:

    access_log /var/log/nginx/honey-access.log;
    error_log /var/log/nginx/honey-error.log;

And now the fun TLS part, the list of allowed protocols -- it's important to have that non-default version of nginx or else you're out of good options on ciphers. I want top grade security, so I write:

    ssl_protocols TLSv1.2 TLSv1.3;

This seems to be a good and security configuration as of June 2019. Then for Diffie Hellmann to work, we configure a shiny new DH key:

    ssl_dhparam /etc/nginx/dhparam.pem;

It was computed with openssl with; be patient, it takes a while:

    openssl dhparam -out dhparams.pem 4096

Now, making a proper cipher list is bit of black magic I don't quite get -- it seems to be passed unaltered to the TLS implementation and the format thus relies on your TLS library you have. For this specific nginx it looks like a cipher list like this will give a really good rating:

    ssl_ciphers 'TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256 kEECDH+ECDSA+AES128 kEECDH+ECDSA+AES256 kEECDH+AES128 kEECDH+AES256 kEDH+AES128 kEDH+AES256 DES-CBC3-SHA +SHA !aNULL !eNULL !LOW !kECDH !DSS !MD5 !RC4 !EXP !CBC !PSK !SRP !CAMELLIA !SEED';
    ssl_prefer_server_ciphers on;

The last line of config makes sure that the server insists on selecting the cipher -- then clients cannot suddenly downgrade for some odd reason -- my server will be in charge of security.

Lets have nginx load the Letsentcrypt certificate and private key:

    ssl_certificate /home/acme/.acme.sh/www.myhoneywebsite.com/fullchain.cer;
    ssl_certificate_key /home/acme/.acme.sh/www.myhoneywebsite.com/www.myhoneywebsite.com.key;

Another fairly important security feature is the `Strict-Transport-Security` header; it makes browsers that _already has_ visited your website only do https for a specified period of time. Not only can that possibly prevent a superfluous server roundtrip, but it also prevents anything from being sent in clear text. This is especially handy if your website has to deal with some sort of authentication cookies. I set the period to one year:

    add_header Strict-Transport-Security "max-age=31536000" always;

# Content

Oh, I need some content as well. I don't have much to say about honey, as I've not really produced any honey yet, but I do need an index.html page. I'll be making it pretty normal straight HTML page. I'll assume that most of the browsers visiting the page are fairly modern and has javascript enabled, thus I might as well inline the stylesheet and javascript for speed. I don't mind responsive design, but I'll aim at making nginx serve a different HTML file for mobile devices, one that references significantly harder compressed media.

So, the HTML is pretty easy to lay out -- simple is better, so a basic structure like this:

    <html>
      <head>
        <title>My Honey Website</title>
        <style>
          h1 {
            font-face: serif;
          }
        </style>
      </head>
      <body>
        <h1>My Honey Website</h1>
        <p>My glourious honey information</p>
        <img src="danish-honey-association.png" width="100" height="100" />
        <script type="text/javascript">
          function onLoad () {
            ..
          }
          document.AddEventListener("DOMContentLoaded", onLoad);
        </script>
      </body>
    </html>

Is what I'm aiming at - as I go along, I'll get more added to this. First, off course, an icon. Since a bee looks really cute, I'm going for a nice little icon of a western honey bee ([Apis mellifera](https://en.wikipedia.org/wiki/Western_honey_bee_)) -- it'll look nice in your bookmark menu and amidst your tabs. In order to infringe any copyrights, I'll be drawing my own bee following this tutorial: [How To Draw a Cute Honey Bee Step By Step Easy For Kids](https://www.youtube.com/watch?v=yY9-m4qurGA) -- my kids have taught me to rely on youtube for almost anything... might as well follow their advise. A 64x64 pixels PNG file showing a cute honey bee with no background (i.e. PNG transparent) is created and put into the document root of my website; then referred to by the HTML like this:

  <link rel="icon" type="image/png" href="/honeybee-favicon.png"/>

Called `favicon` in honor of dark origins of the icon facility itself. Also in my document root I'll put the logo of the Danish Beekeeper Association as a PNG with transparent background. By adding the `width` and `height` attributes of the IMG tag I'm allowing the browser render engine to layout the page before the image is downloaded. This improves the rendering speed -- and speed is important.

    <img src="beekeeper-logo.png" width="113" height="99" alt="[Danish Beekeeper Association logo]"/>

The `alt` attribute is important for accessibility concern -- vision impaired people have automation that will tell them what the picture is about, also, most browsers will show the `alt` text by the mouse cursor as you hover slowly over the image; a nice feature for me, crucial for others. 

Since all the fancy up and coming tech companies seem to have video streams on their websites, I will, off course, have that too. Using my cheap Canon Camcorder I recorded some 20 minutes of bee activity from the bee hive. I pulled out some 15 minutes using [Lightworks](https://www.lwks.com/) -- and this needs to be compressed properly for online usage. The new AV1 codec is much appraised, and well supported in the newer Chrome browser. But the standard ffmpeg on Debian Stretch does not support it. A problem to be solved. Luckily I stumbled upon: [Brainiarc7 / 	install-ffmpeg.sh](https://gist.github.com/Brainiarc7/7b099f98f6b373699aa2f54e5d6ccb58) which downloads and installs a version of ffmpeg that _does_ AV1 encoding. The author has tested it on Ubuntu 16.04 -- I found no big problems running it on Debian Stretch; there was just a single Debian package that I had to install manually:

    apt-get install libfdk-aac-dev

Now, with ffmpeg installed in `$HOME/bin` I can start converting the video. It'll be handy to have AV1, h.265 and h.264 versions around, so my complete convert job looks like this:

    $HOME/bin/ffmpeg -i bee-movie.mp4 -vf crop=1280:272:0:268 -an -c:v libaom-av1 -strict -2 bee-movie-av1.mp4
    $HOME/bin/ffmpeg -y -i bee-movie.mp4 -vf crop=1280:272:0:268 -c:v libx265 -b:v 400k -x265-params pass=1 -an -f mp4 /dev/null && \
    $HOME/bin/ffmpeg -i bee-movie.mp4 -vf crop=1280:272:0:268 -c:v libx265 -b:v 400k -x265-params pass=2 -an bee-movie-h265.mp4
    $HOME/bin/ffmpeg -y -i bee-movie.mp4 -vf crop=1280:272:0:268 -c:v libx264 -b:v 400k -x264-params pass=1 -an -f mp4 /dev/null && \
    $HOME/bin/ffmpeg -i bee-movie.mp4 -vf crop=1280:272:0:268 -c:v libx264 -b:v 400k -x264-params pass=2 -an bee-movie-h264.mp4

I'll need to redo the AV1 in dual-pass; I'll leave that task for later as the encoding process is _really_ slow - as in _days_. Preferably don't do this on your laptop -- it'll most likely get hot and loud -- my server does that too, but in the basement (to an extent that I rather not do this while anyone is trying to sleep). Also, for mobile device that are getting more and more capable, I'll want a lowres version with less bandwidth; I know that modern smartphones have incredible displays of immense resolution -- but to save bandwidth I'll be serving them a file of about 25% the size of what a browser will get.

The lowres version is done by:

    $HOME/bin/ffmpeg -y -i bee-movie.mp4 -vf crop=1280:272:0:268,scale=640:136 -c:v libx264 -b:v 100k -x264-params pass=1 -an -f mp4 /dev/null && \
    $HOME/bin/ffmpeg -i bee-movie.mp4 -vf crop=1280:272:0:268,scale=640:136 -c:v libx264 -b:v 100k -x264-params pass=2 -an bee-movie-small-h264.mp4

I might make more versions (I believe I once read that Netflix has at least 20 different encodings of all their material).

These video files get quite big -- the h.264 version is some 40MiB; since I'll be hosting this site on my home xDSL line, I'd prefer if people watching my site are not exhausting my capacity. Thus I throttle the download. The `video` folder of my website will allow fast download of the first megabyte of a file, after which you're throttled to 50KiB per second. 50KiB is the equivalent of 400 kilobit -- exactly the highest bitrate of my codecs, so I guess it's a bit tight (there's an mp4 container, network noise, etc.) -- but it seems to work. Here's my nginx configuration:

    location /video {
      limit_rate_after 1m;
      limit_rate 50k;
    }

# Mobile support

So, using responsive design it's possible to serve the same document, the same stylesheet, the same code to all devices and have them figure out the layout, the correct media and whatnot. Here, I'll go down a different road, that of serving mobile phones a differnt version of the HTML -- by doing this, I make sure that a client which, by the server, is regarded as being a mobile phone, is never handed any of the URLs to the high definition videos. In the future I might make the response design act differently on phones -- I will, for sure, very soon make adjustments to the stylesheet for mobile devices, as the current stylesheet is not really optimized for smaller screen. Right now I only have one HTML file, the infamous `index.html` -- configured as this:

    index index.html;

Now I'd like to present a different html file for mobile devices, I could opt for an `if` statement, but [If Is Evil](https://www.nginx.com/resources/wiki/start/topics/depth/ifisevil/) -- and thus I'll try something else. From https://gist.github.com/perusio/1326701 I get this mapping `$http_user_agent` that becomes very useful:

    map $http_user_agent $is_desktop {
      default 0;
      ~*linux.*android|windows\s+(?:ce|phone) 0; # exceptions to the rule
      ~*spider|crawl|slurp|bot 1; # bots
      ~*windows|linux|os\s+x\s*[\d\._]+|solaris|bsd 1; # OSes
    }

This I would like to make into a content suffix; like this:

    map $is_desktop $content_suffix {
      1 "";
      0 "-mobile";
    }

Now I can use the `$content_suffix` anywhere I like and have it do nothing at all for desktop browsers, but evaluate to "-mobile" for all other requests; alas, my configuration of `index.html` now looks like this:

    index index$content_suffix.html

Meaning that my browser will get `index.html` as usual, but my phone will be feed `index-mobile.html` when accessing https://www.myhoneywebsite.com/. I tested it, it works. Right now the desktop and the mobile version look very similar, but the mobile version only has one video listed for the background and that is an h.264 encoded lowres video, whereas the desktop version lists three different encodings: av1, h.265 and h.264. Most browsers seem to choose av1 or h.264. I might do some statistics on that.

Serving different content on the same incoming URL is a bit tricky, especially if you ever want to front your website with a CDN. How is a CDN gonna know which clients can be served a given cached version; so I make sure to tell such machinery to include the client `User-Agent` header in whatever cachekey it's building up:

    add_header Vary "User-Agent" always;

The `always` specifies that this header _must_ be sent, also for error pages and the like. This directive will a CDN _less_ efficient in shielding my server; for me that's a tradeoff I can live with. 

# Final site

Not really final, as anything on the internet, it's always a work in progress. But the website is now live at:

[honning.dalsgaard.net](https://honning.dalsgaard.net/)

Danish only for the time being, but I'm working on an international version for when my honey is to be sold globally...

My efforts seem to have not been in vain, according to [Qualys SSL Labs](https://www.ssllabs.com/) my website scores [A+](https://www.ssllabs.com/ssltest/analyze.html?d=honning.dalsgaard.net) -- better than many others, such as:

* [um.dk](http://um.dk/) - The Foreign Ministry of Denmark, that apparently does not run SSL/TLS at [all](https://www.ssllabs.com/ssltest/analyze.html?d=um.dk).
* [nets.eu](https://www.nets.eu/) - NETS, payment provider in Denmark, and Scandinavia, and increasingly the rest of Europe, they have [B](https://www.ssllabs.com/ssltest/analyze.html?d=www.nets.eu).
* [politi.dk](https://politi.dk) - The Danish Police Force, they have [B](https://www.ssllabs.com/ssltest/analyze.html?d=politi.dk).

Now, not having a high score on your frontpage may not be a huge problem. Most interactions with these bodies do not go via their hosted main website. Also, my version of grade A+ security prevents some old clients from viewing it at all - your use case may vary, and A+ might not be for you.

_Update:_ Checked past week, on Friday March 27 2020, and now the Foreign Ministry of Denmark has upgraded their site to use SSL now; good job!



Tags: computer, debian, devops, video, linux
