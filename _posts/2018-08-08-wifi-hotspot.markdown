---
layout: post
title:  "Command to turn on a wifi hotspot"
date:   2018-08-08 10:01:14 +0000
tail: <a href="http://www.pcninja.us/turn-your-windows-7-laptop-into-a-wifi-hotspot/" class="post-tags">[More]</a>
categories: app
keywords: wifi, hotspot
---

1. Click Start (i.e. the Windows logo at the bottom left hand corner), type Cmd, right-click the Cmd.exe link and select “Run as Administrator”. If you don’t have that option, you aren’t an Administrator to your computer and you can’t proceed unless you’re setup as one.

2. Type the command below:

`netsh wlan set hostednetwork mode=allow ssid=MyNetworkhere key=Password`

and press [Enter]. Replace “MyNetworkhere” with the name you’d like to use for your new wifi network, and “Password” with a password of your choice.

3. While you are still at the command line, type

`netsh wlan start hostednetwork`

4. and press [Enter] to start the virtual adapter.

5. Now click Control Panel > Network and Internet > Network and Sharing Centre > Change Adapter Settings, right-click your internet connection and select Properties. Click the Sharing tab, check “Allow other network users to connect…”, choose your virtual Wi-Fi adapter.

That’s it! From now on, any WiFi capable device will able to pick up your laptop as a Wifi hotspot. Your WiFi network name (or SSID) will be what you defined above. Guess what, you’ve just learnt a trick to bypass having to purchase a wireless router for your house (with some minor downsides of course).

If you don’t want to share your laptop hotspot anymore:

Click Control Panel > Network and Internet > Network and Sharing Centre > Change Adapter Settings, right-click your internet connection and select Properties. Click the Sharing tab, uncheck “Allow other network users to connect…”, click  OK 