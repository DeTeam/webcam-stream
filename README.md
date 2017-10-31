# Webcam streaming with raspberry pi

Below you can find steps I took in order to setup by old raspberry pi (original, Model B) as a webcam HLS streaming service. Both the sreaming and the index.html are protected with basic authentication and Fail2ban on top.

Worth mentioning, even though the solution seem rather stable, it is DYI thing and I would not recommend relying on it for life-critical use-cases. If your goal is just to satisfy your curiousity — go ahead.

We can split our journey in several blocks, each one adds up to our end-goal: having secure enough HLS webcam streaming service with your raspberry.

## Prerequisites

Of course, first of all you'll need a raspberry pi together with some accessories: power/usb-cable, usb wifi module or LAN-cable and a webcam.

I recommend you going through basic raspberry pi and ssh tutorials so that you can familiarize yourself before going into more advance topics. Take your time.

In terms of OS, I have Raspbian "Jessie", though using either "Wheezy" or "Stretch" should also be fine.

Mind, that all of these steps we're going to perform on the raspberry pi itself. You'll need to have ssh properly configured and familiarize yourself with a text editor (being it nano, vim, emacs or something else).

## Components explained

To give you some overview from the beginning what we'll need, here's the setup:

We're going to have our webcam available in raspbian at `/dev/video0`.

Afterwards, we configure nginx with nginx-rtmp-module in order to stream our HLS video.

Once it's setup we need FFmpeg to capture video from our webcam device `/dev/video0` and publish it to the rtmp stream manged by our nginx setup.

Then we configure some basic authentication and protect it with some Fail2ban in order to mitigate some possible naive bruteforce attempts.

As an optional item (not covered here), you could also setup some dynamic dns in order to access your video stream from the internet, using your mobile device, in a convenient way.


## Bulding nginx

Because we need nginx-rtmp-module we can't just install nginx as usual from the repository. We need to build nginx. [This article](https://docs.peer5.com/guides/setting-up-hls-live-streaming-server-using-nginx/) from peer5 turned very handy explaining all the necessary steps. I'll post some of them here, modifying it slightly.

First let's grab some dependencies for building nginx:

```sh
sudo apt-get install build-essential libpcre3 libpcre3-dev libssl-dev
```

Few steps just to get ourself comfortable:

```sh
cd ~
mkdir buliding-nginx
cd building-nginx
```

Then let's download nginx (current latest version, you can look up if there's a more recent one by the time you read it):

```sh
wget http://nginx.org/download/nginx-1.12.2.tar.gz
tar -xf nginx-1.12.2.tar.gz
```

Now we have nginx sources, let's also grab nginx-rtmp-module:

```sh
git clone https://github.com/sergey-dryabzhinsky/nginx-rtmp-module.git
```

Now it's time to prepare ourself for building our very own nginx

```sh
cd nginx-1.12.2
./configure --prefix=/usr/share/nginx \
    --sbin-path=/usr/sbin/nginx \
    --conf-path=/etc/nginx/nginx.conf \
    --pid-path=/var/run/nginx.pid \
    --lock-path=/var/lock/nginx.lock \
    --error-log-path=/var/log/nginx/error.log \
    --http-log-path=/var/log/access.log \
    --user=www-data \
    --group=www-data \
    --without-mail_pop3_module \
    --without-mail_imap_module \
    --without-mail_smtp_module \
    --without-http_fastcgi_module \
    --without-http_uwsgi_module \
    --without-http_scgi_module \
    --without-http_memcached_module \
    --with-ipv6 \
    --with-http_ssl_module \
    --with-http_gzip_static_module \
    --add-module=../nginx-rtmp-module
```

Once it's done we want to go an interesting way. Normally one'd run `make` command and wait for results. We're gonna do it differently. Since we're running things via ssh and `make` for nginx (and ffmpeg, below) will take quite some time (could be more than an hour, depending on your raspbery CPUs), we're gonna run `make` using nohup in the background.

**If you have more than one CPU on your raspberry, benefit from it and add it with `-j` parameter (second example).**

```sh
# on one core
nohup make < /dev/null > nginx-build.log 2>&1 &
```

```sh
# on 4 cores
nohup make -j 4 < /dev/null > nginx-build.log 2>&1 &
```

No you can run `exit` and do some other business in a meanwhile. You can monitor your progress by doing:

```sh
tail -n 20 ./nginx-build.log
```

and checking if the build is done, there would be a message like this:

```
make[1]: Leaving directory
```

One we're done, let's install our nginx so we can finally use it globally:

```sh
sudo make install
```

## Configuring nginx

Now we have nginx up and running. We can manipulate it using:

```sh
sudo service nginx start
sudo service nginx stop
sudo service nginx restart
```

Let's create a few directories we'll need for our webserver:

```sh
mkdir -p /home/pi/webcam-stream/hls
mkdir -p /home/pi/webcam-stream/www
```

It's time to adjust our configuration. Below you'll find what I used in my particular case. Make it content of `/etc/nginx/nginx.conf` using a text editor of your choice.

```nginx
worker_processes 1;
events {
    worker_connections  1024;
}

# RTMP configuration
rtmp {
    server {
        listen 1935; # Listen on standard RTMP port
        chunk_size 4000;

        application show {
            allow publish 127.0.0.1;
            deny publish all;
            allow play all;
            on_play http://localhost:8345/check-auth.txt;
            live on;
            # Turn on HLS
            hls on;
            hls_path /home/pi/webcam-stream/hls/;
            hls_fragment 3;
            hls_playlist_length 60;
            # disable consuming the stream from nginx as rtmp
            deny play all;
        }
    }
}

http {
    sendfile off;
    tcp_nopush on;
    directio 512;
    default_type application/octet-stream;

    # we'll uncomment it in auth step
    # auth_basic "webcam";
    # auth_basic_user_file /etc/nginx/.htpasswd;

    server {
        listen 8080;

        location = /check-auth.txt {
            return 200 'ok';
        }

        location /hls {
	        rewrite /hls/(.*) /$1 break;
            # Disable cache
            add_header 'Cache-Control' 'no-cache';

            # CORS setup
            add_header 'Access-Control-Allow-Origin' '*' always;
            add_header 'Access-Control-Expose-Headers' 'Content-Length';

            # allow CORS preflight requests
            if ($request_method = 'OPTIONS') {
                add_header 'Access-Control-Allow-Origin' '*';
                add_header 'Access-Control-Max-Age' 1728000;
                add_header 'Content-Type' 'text/plain charset=UTF-8';
                add_header 'Content-Length' 0;
                return 204;
            }

            types {
                application/dash+xml mpd;
                application/vnd.apple.mpegurl m3u8;
                video/mp2t ts;
            }

            root /home/pi/webcam-stream/hls;
        }

	location / {
            root /home/pi/webcam-stream/www;
        }
    }
}
```

It's pretty similar to the example given in peer5 article, however I removed the aio part and added some auth for the rtmp stream: `on_play` directive would ask our normal webserver url for status (success or failure) and thus trigger basic auth for anyone trying to play HLS stream directly (you can actually use VLC or some other player to connect to it).

Config itself consist of two part: rtmp and www. First one is about our video stream (note, that there's nothing playing until we get to the ffmpeg part). Second — a webserver.

Now when we have the nginx setup almost ready, let's place an `index.html` file in `/home/pi/webcam-stream/www`:

```html
<!DOCTYPE html>
<html>
    <head>
        <title>webcam-stream</title>
        <meta charset="utf-8" />
        <script type="text/javascript" src="https://cdn.jsdelivr.net/npm/clappr@latest/dist/clappr.min.js"></script>
    </head>
    <body>
        <div id="player">

        </div>
        <script>
            var player = new Clappr.Player({
                source: "/hls/stream.m3u8",
                parentId: "#player",
                maxBufferLength: 10,
                width: 640,
                height: 360
            });
        </script>
    </body>
</html>
```

With this done, starting nginx should make the `index.html` available on port `8080` on your raspberry pi. Try to find the IP address of it in your local network and access the `index.html`. It should display a player created using [Clappr](https://github.com/clappr/clappr), but so far it should not work. We also don't have auth enabled yet (there's a separate step for it later on).

## Building FFmpeg

Alright, now we need to actually grab some video. The tool we're going to use for it is [FFmpeg](https://www.ffmpeg.org/). Try running `sudo apt-get install ffmpeg` in your terminal. If you're lucky and it's available, you can completely skip this section, otherwise — bare with me and let's build it from the sourcecode.

I used [this article](https://www.jungledisk.com/blog/2017/07/03/live-streaming-mpeg-dash-with-raspberry-pi-3/) from jungledisk and [this reddit post](https://www.reddit.com/r/raspberry_pi/comments/5677qw/hardware_accelerated_x264_encoding_with_ffmpeg/) while building ffmpeg, here's the summary of commands you'd have to run.

\* *in the article there's a different streaming format used at the end — MPEG-DASH. I also tried it out, though it doesn't seem to work on iOS devices, which is an issue for me — thus, swirtched to HLS.*

```sh
sudo apt-get update
sudo apt-get install autoconf automake build-essential libass-dev \
    libfreetype6-dev libsdl1.2-dev libtheora-dev libtool \
    libva-dev libvdpau-dev libvorbis-dev libxcb1-dev libxcb-shm0-dev \
    libxcb-xfixes0-dev pkg-config texinfo zlib1g-dev
cd ~
git clone https://github.com/ffmpeg/FFMpeg --depth 1
cd ~/FFMpeg
./configure --enable-gpl --enable-nonfree --enable-mmal --enable-omx-rpi
```

In the last command, mind that I excluded `--enable-omx`, since on my raspberry pi it caused `configure` to fail, plus `--enable-omx-rpi` already does everything required (and more, since it's made for raspberry pi)!

Now, we're ready to kick-ff `make` for ffmpeg, let's do it the same way we did for nginx, using nohup:

```sh
nohup make < /dev/null > ffmpeg-build.log 2>&1 &
```

Once it's done, make sure that this command (run inside ffmpeg directory) returns us something useful:

```sh
./ffmpeg -encoders | grep h264_omx
```

If all good — let's install ffmpeg:

```sh
sudo make install
```

Done, `ffmpeg` command should be available on your `$PATH` now. To check if everything is ok, try run a command below and then stop it using Ctrl-C:

```sh
ffmpeg -i /dev/video0 -c:v h264_omx -c:a copy -b:v 1500k test.mp4
```

Once this done, there should be a `test.mp4` file in your directory available. You can retreive it using `scp` or `rsync` (covered in various raspberry pi tutorials). Play it and you should see what your camera recorded during few seconds you let it run.


## Configuring systemd service for FFmpeg

Now we're going to make sure our webcam is streaming what we need and does it everytime you boot your raspberry pi. For that we're going to create a systemd service.

Simply create a file named `webcam-stream.service` in `/etc/systemd/system` and will it with the following content:

```conf
[Unit]
Description=Video service for the webcam

[Service]
User=pi
ExecStart=/home/pi/stream.sh
Restart=always
RestartSec=30s

[Install]
WantedBy=multi-user.target
```

As you can see it points to `/home/pi/stream.sh`, let's also create it and fill with the following ffmpeg command:

```sh
#!/bin/sh

ffmpeg -re \
	-f video4linux2 \
	-i /dev/video0 \
	-framerate 25 \
	-vcodec h264_omx \
	-map 0:v \
	-b:v 800k \
	-vprofile baseline \
	-vf "scale=640:360" \
	-f flv \
	rtmp://localhost/show/stream
```

Now we just need to enable our new service:

```
sudo systemctl enable webcam-stream
sudo systemctl daemon-reload
```

Now few more commands are available:

```sh
sudo systemctl stop webcam-stream
sudo systemctl start webcam-stream
sudo systemctl status webcam-stream

# acessing the logs
sudo journalctl -u webcam-stream
```

From this point, the whole basic setup should be working. Try accessing the website and turn on the player — it should show content from your webcam. With the given setup, there will be a delay, which I don't know how to remove.

Further steps are needed if you want to allow access to your webcam outside of your local network.

## Basic auth

I was using [this article](https://www.digitalocean.com/community/tutorials/how-to-set-up-password-authentication-with-nginx-on-ubuntu-14-04) from digitalocean to setup users and password for nginx basic auth (section: *"Create the Password File Using Apache Utilities"*).

```sh
sudo apt-get install apache2-utils
sudo htpasswd -c /etc/nginx/.htpasswd some_login
```
Once done, uncomment the following lines in `/etc/nginx/nginx.conf`:

```nginx
auth_basic "webcam";
auth_basic_user_file /etc/nginx/.htpasswd;
```

Now if you run `sudo service nginx restart`, going to your website would trigger auth window to popup, log in to access the player & the video.

## Fail2ban configuration

We're still not ready to open up our website: by default basic auth is not the most secure way of protecting resources, so what we're going to do is to extend it using a tool called p[fail2ban](https://www.fail2ban.org/wiki/index.php/Main_Page) (thanks to my friend [Dan Peddle](https://flarework.com/) for recommending it).

Again, digitalocean comes in handy with [another article](https://www.digitalocean.com/community/tutorials/how-to-protect-an-nginx-server-with-fail2ban-on-ubuntu-14-04) explaining how to set things up, let's take a few nice commands from there and do the rest on our own:

```sh
sudo apt-get install fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

We're ready to apply some changes. When I was first following the article and going for the most basic nginx-http-auth filter and "jail" setup. It wasn't working so I browsed the network a bit and decided to use things from some gists, here's what we need to do:

Create a file called `nginx-auth.conf` in `/etc/fail2ban/filter.d`:

```conf
[Definition]

failregex = no user/password was provided for basic authentication.*client: <HOST>
            user .* was not found in.*client: <HOST>
            user .* password mismatch.*client: <HOST>

ignoreregex =
```

Perfect, next step — make it available, let's edit `/etc/fail2ban/jail.local` file and _add_ a few liens:

```conf
[nginx-auth]

enabled = true
filter = nginx-auth
port  = 8080,http,https
logpath = /var/log/nginx/error.log
```

This will create a "jail" called `nginx-auth` and it will use the corresponding filter.

Let's also adjust some default (find the corresponding variables in the `jail.local` under the `[DEFAULT]` section and adjust them to the values below):

```conf
findtime = 3600
maxretry = 10
bantime = 3600
ignoreip = 127.0.0.1/8 your_home_IP # put your home IP or range
```

Now we just need to:


```sh
sudo service fail2ban restart
```

Few more useful commands:

```sh
sudo fail2ban-client status
sudo fail2ban-client status nginx-auth
sudo fail2ban-client set nginx-http-auth unbanip 1.1.1.1 # some IP
```

Try failing to login more than 10 times — your IP should be banned (no worries, you should be able to SSH and unban yourself).

## Port-forwarding

Finally, to make things available you might need setup port forwarding for the port 8080 in your home router to your raspberry pi address. **Make sure to only forward port 8080**.