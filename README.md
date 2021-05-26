# webcam-stream

Here you will find a [tutorial](Tutorial.md) on how to setup your Raspberry Pi to stream video from a webcam in a nice and secure enough manner.

While trying to DIY such a thing, I didn't find a nice and complete set of steps, this tutorial is an attempt to fix it.
 
If you find any kind of errors or mistakes - please [fill in an issue](https://github.com/DeTeam/webcam-stream/issues/new) or, if you have time, contribute a PR right away.


It's important that:

* Information stays up to date
* Security is covered so that private info is not exposed
* Steps are straightforward enough and don't require very deep knowledge of underlying tools

## Credits

* Arigato to peer5 for a [nice article](https://docs.peer5.com/guides/setting-up-hls-live-streaming-server-using-nginx/) on streaming hls
* Gracias to Jason Kölker from JungleDisk for [posting some info](https://www.jungledisk.com/blog/2017/07/03/live-streaming-mpeg-dash-with-raspberry-pi-3/) on MPEG-DASH streaming
* [Dan Peddle](https://flarework.com/) suggested using fail2ban, merci!
* Thanks to [Denis Rechkunov](http://pragmader.me/) for doing some sanity checks
* [DigitalOcean](https://www.digitalocean.com/community/) came in super handy with some articles on basic nginx setup, thanks
* Thanks to [@ishansheth](https://github.com/ishansheth) for adding a section on systemd part for nginx
