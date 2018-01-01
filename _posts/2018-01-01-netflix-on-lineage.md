---
layout: post
title: Netflix on LineageOS with a Galaxy Tab 8
---

I've been running [Cyanogenmod 12.1](https://archive.org/download/cmarchive_snapshots) on a Galaxy Tab 3 8.0 (GT-N5110) for quite some time now without issue. However, when *someone* decided they wanted Netflix, it didn't work, apparently due to [SafetyNet](https://www.lineageos.org/Safetynet/). After some fiddling around, it turns out that a fresh install of [LineageOS 14.1](https://forum.xda-developers.com/galaxy-note-8-0/development/rom-unofficial-lineageos-14-1-gt-n51xx-t3533329) got it working. Eh...

<!--more-->

## Signs of trouble
The first red flag was that Netflix didn't appear in the Google Play store. Apparently, if SafetyNet checks fail, then it's automatically hidden from the store. Given that I hadn't yet installed a root, it seems that the unlocked bootloader (TWRP) was the cause of the trip. While it was possible to [sideload](https://help.netflix.com/en/node/57688) the app, it failed on playing videos with: 

> Sorry, we could not reach the Netflix service. Please try again later. If the problem persists, please visit the Netflix website (0013).

## Potential fixes
Several searches later suggested installing [Magisk](https://forum.xda-developers.com/apps/magisk) to make it pass the SafetyNet checks, although I was hesitant to root it, just to get Netflix access.

Taking another track, I had come across user agent spoofing (such as on [the RPi](https://thepi.io/how-to-watch-netflix-on-the-raspberry-pi/)) to get Netflix playing through the browser. While Firefox can spoof user agents (through an add-on), I saw that it lacked the [Widevine DRM](https://support.mozilla.org/en-US/kb/enable-drm) decryption module required to play the videos. I did come across [this thread](https://www.reddit.com/r/firefox/comments/709jtj/is_there_a_plan_to_implement_widevine_for_firefox/), which suggested it was supported on Android 6+. 

I had held off upgrading to 14.1, because it's technically listed as 'alpha' quality. But with little else to lose, I finally decided to just do it.

### Hey, it works!
As luck would have it, after the upgrade, the app was now available in the store, and it worked with no other modifications needed. I'm still don't know why this is; I would have thought that *at least* TWRP would trip SafetyNet.

It was a good thing too that that worked, since I couldn't get the Widevine support in Firefox, even on 14.1 (Android 7.1), which was quite disappointing. I do think however, that this would be a viable alternative *when* (or maybe *if*) this support lands.

## Upgrade details
These are the specifics for my upgrade:

* I already had TWRP installed, although I did upgrade from 3.0.0-0 to [3.0.2-0](https://dl.twrp.me/n5110/).
* I did a backup (in TWRP), including everything but the cache partitions
* I did a standard wipe/factory reset (again in TWRP)
* I used [lineage-14.1-20171228-nightly-n5110-signed](https://mirrorbits.lineageos.org/full/n5110/20171228/lineage-14.1-20171228-nightly-n5110-signed.zip) with [Open GApps](http://opengapps.org/?api=7.1&variant=nano) - [open_gapps-arm-7.1-micro-20171230](https://github.com/opengapps/arm/releases/download/20171230/open_gapps-arm-7.1-micro-20171230.zip) (in hindsight I should have stuck with the nano version, as I dislike the Gmail app)

So far, most of everything works as before. The Telstra Air app however, is now listed as being incompatible, which is strange.
