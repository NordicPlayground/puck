---
layout: post
title: This is a puck
---

Say you're sitting in your lounge at home, thinking how relaxing some smooth jazz would be right now.

![](/mbed-pucks/images/smooth_jazz.jpg)

What if you could have smooth jazz at your choice of location, without the hassle of turning the music on yourself?

Luckily you've recently accuired one of the new Nordic Semiconductor mBed BLE-Smart enabled ARM chips, and find yourself equipped with a laptop with the mBed online compiler open in a tab. You've heard of iBeacons, maybe you could program your smartphone to automatically start playing Take Five ([Dave Brubeck – Take Five](http://open.spotify.com/track/5p6me2mwQrGfH30eExHn6v)) whenever your phone enters the lounge area, identified by a part of the iBeacon broadcast identifier.

Lucky for our dear readers, me and the rest of a crack-team of summer interns (The Nordic Semis) have during our stay at Nordic amongst others developed a bluetooth LE (aka bluetooth smart) library to ease the pain of quickly turning an mBed chip into a BLE hub. These hubs are lovingly called 'Pucks', a name that lingers on from some early prototype casing design involving 'trolldeig' <http://en.wikipedia.org/wiki/Salt_dough>.

And that's all that's required for now. If you wish to have a look at what happens behind the curtains, either take a look at the puck library, or look at this wonderful tutorial written by the mBed BLE team where they do iBeacon setup manually <http://mbed.org/teams/Bluetooth-Low-Energy/wiki/Homepage>.

And profit! Now your lounge will be filled by the smoothest jazz whenever your path takes you closeby.
