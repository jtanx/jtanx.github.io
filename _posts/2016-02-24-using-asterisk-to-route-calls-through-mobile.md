---
layout: post
title: Using a Raspberry Pi, Asterisk and a Bluetooth dongle to route phone calls through a mobile phone
---

Mobile data is a strange thing in Australia. After taking advantage of an Optus 'bonus data' prepaid offer (5GB for $5, although I only got 3GB...), I was left with 'unlimited' calls that I was never going to make the best use of. However, I figured there was a better way to leverage more out of this offer - specifically by routing *all* my home phone calls through my mobile service.

![diagram](https://raw.githubusercontent.com/jtanx/jtanx.github.io/master/_img/2016-02-24-diagram.png)

After some thought, the above diagram is how the system works. As I was already routing home phone calls through an [ATA](https://en.wikipedia.org/wiki/Analog_telephone_adapter) to use VoIP, I only needed to add my own SIP server that somehow routed my calls through my mobile. Having a Windows Phone, writing an app to do this is impossible, given that the API is simply not there to perform remote calls. However, like many other smartphones, it *does* support making and receiving calls through a Bluetooth connection - and this is how, along with an Asterisk instance running the `chan_mobile` module, this setup is possible. The only reason I'm using the Raspberry Pi to do this was because I had one laying around that wasn't doing much of anything, and this seemed like the perfect opportunity to use it.

## Getting Asterisk
For the Raspberry Pi, [RasPBX](http://www.raspberry-asterisk.org/) seems to be the way to go.

1. For a quickstart, download [RasPBX](http://www.raspberry-asterisk.org/downloads/) (at the time of writing, I am using version 25-01-2016). Unzip and write the `.img` file to a microSD card that's at least 4GB large. I like to use [Win32 Disk Imager](https://sourceforge.net/projects/win32diskimager/). 
    * If like me, you don't have (or aren't willing to spare) a 4GB microSD card, but do have a bunch of USB sticks lying around, then you can follow these steps instead (be warned: it's quite tedious and you should know your way around Linux)
        1. You will need a microSD card that is at least 100MB large and a USB stick that is at least 4GB large (you may get away with 2GB, but you're really pushing it). Ensure the microSD card is FAT32 or FAT16 formatted (I prefer FAT32).
        2. Install [Virtualbox](https://www.virtualbox.org/wiki/Downloads) and create a Linux VM. You will need some way of mounting shared folders from within the VM. I already had an Ubuntu VM which served this purpose. To the VM, add a hard drive specifically for the RasPBX image (at least 4GB large)
        3. From inside the VM, I will assume the empty VM hard disk is on `/dev/sdb`, and the microSD card is on `/dev/sdc`. I will also assume that you have setup a shared folder that contains the RasPBX image.
        
          ````bash
          mkdir share
          sudo mount -t vboxsf my_shared_folder share
          
          mkdir microsd
          sudo mount /dev/sdc1 share
          ````

        4. Dump the contents of the image into the empty hard drive:
        
          ````bash
          cd share
          sudo dd if=raspbx-25-01-2016.img of=/dev/sdb
          ````
          
          Or if you want to see progress, install `pv` first, then run
          
          ````bash
          cd share
          pv raspbx-25-01-2016.img | sudo dd of=/dev/sdb
          ````
        
        5. Refresh the partition table:
        
          ````bash
          sudo partprobe /dev/sdb
          ````
          
          You should now see `/dev/sdb1` (the boot partition) and `/dev/sdb2`.
        6. Copy everything from `/dev/sdb1` into the microSD card:
        
          ````bash
          mkdir tmp
          sudo mount /dev/sdb1 tmp
          sudo cp -r tmp/* microsd/*
          
          sudo umount tmp
          sudo umount microsd
          ````
          
        7. From within your host machine, edit `cmdline.txt` on the microSD card: Where you see `root=/dev/mmcblk0p2`, replace this with `/dev/sda1` - this will make it boot from the USB stick.
        8. Back in the VM, boot into [Clonezilla](http://clonezilla.sourceforge.net/) (I use the stable ISO). Choose all the default options. When Clonezilla is started, choose to work `device-device` (not `device-image`) then `part_to_local_part`. The source partition will be `/dev/sdb2` (the main partition of the RasPBX image).
        9. Ensure that your USB stick is connected to the VM. This will be your destination partition. On my VM, this would be `/dev/sdc1`. If you don't see this, check that it's connected (under `Devices->USB`) and re-try Clonezilla.
        10. All the default options should work. **Note**: Double check that the source and destination partitions are correct! If you don't, you may mistakenly wipe one of your other (virtual machine) drives. Once you confirm everything, Clonezilla should dump the main RasPBX partition onto your USB stick. It should also automatically expand the partition to match the size of your USB stick, which is nice. Once done, exit Clonezilla.
        11. Profit! To boot, ensure that the USB stick and your microSD card is connected to your Raspberry Pi before powering on. On bootup, you should see the red LED blink for a few seconds before switching off - then your USB LED (if you have one) should start blinking - it's now booting off the USB stick. Yay (and phew)!

### Configuring RasPBX
The following steps help setup and customise RasPBX to suit your situation. Essentially this follows the stock standard [documentation](http://www.raspberry-asterisk.org/documentation/):

1. SSH into the RPi - using PuTTY to `raspbx` should work. Or just enter the IP address of the RPi.
2. Inside the SSH session (login is root/raspberry), run
  ```bash
  raspbx-upgrade
  ```
  
  This will upgrade all the components. I found this took quite a while (30 mins on a RPi B+).
  
3. Run

  ````bash
  regen-hostkeys
  configure-timezone
  dpkg-reconfigure locales
  passwd         (to change the root password)
  ````

4. I skipped setting up the email. I also didn't use the provided web GUI (FreePBX). I found it easier to just modify the config files and use Asterisk directly. In fact, FreePBX got in my way, so I disabled it:

  ````bash
  update-rc.d freepbx remove
  update-rc.d apache2 remove
  wget http://repo.raspbx.org/download/asterisk -o /etc/init.d/asterisk
  chmod +x /etc/init.d/asterisk
  update-rc.d asterisk defaults
  reboot
  ````

## Adding `chan_mobile` support
1. You will need a USB Bluetooth module that works with the Raspberry Pi. I managed to procure one of the el-cheapo [$1 eBay USB Bluetooth adaptors](http://www.ebay.com.au/sch/i.html?_from=R40&_sacat=0&_nkw=bluetooth+usb&_sop=15), which worked fine. `lsusb` lists it as:

  ~~~~~
  Bus 001 Device 005: ID 0a12:0001 Cambridge Silicon Radio, Ltd Bluetooth Dongle (HCI mode)
  ~~~~~

2. Next up, you will need to install the Bluetooth drivers:

    ~~~~~
    apt-get install bluetooth bluez
    ~~~~~

3. You will then need to pair it with your phone. With Bluetooth turned on in your phone, run:

    ~~~~~
    bluetoothctl
    ~~~~~

  This will provide you with a `bluetooth` console. From here, run
  
    ~~~~~
    list
    ~~~~~
    
    You should see your Bluetooth controller. Take note of it's ID (like a MAC address, e.g. `14:90:E2:1A:AC:80`).
  
    ~~~~~
    scan on
    ~~~~~
  
  After a while, run
  
    ~~~~~
    devices
    ~~~~~
    
  You should see your phone listed, along with its own ID. Take note of the ID. Now enter
  
    ~~~~~
    pair <your-phone-id>
    ~~~~~
    
  And follow the prompts to pair the phone. Once done, you can type `quit` to exit the Bluetooth console.
4. The next step is to enable `chan_mobile` support to Asterisk. Under `/etc/asterisk`, create a file called `chan_mobile.conf`. You can base the contents on this [sample file](http://doxygen.asterisk.org/trunk/chan_mobile.conf.html). My configuration goes along the lines of:

  ````bash
  [general]
  interval=30             ; Number of seconds between trying to connect to devices.
  
  [adapter]
  id=blue
  address=14:90:E2:1A:AC:80
  
  [WP530]
  address=0A:E2:10:C3:75:30       ; the address of the phone
  port=3                          ; the rfcomm port number (from mobile search)
  context=incoming-mobile         ; dialplan context for incoming calls
  adapter=blue                    ; adapter to use
  group=1                         ; this phone is in channel group 1
  ;sms=no                         ; support SMS, defaults to yes
  ;nocallsetup=yes                ; set this only if your phone reports that it supports call progress notification, but does not do it. Motorola L6 for example.
  ````
  
  The `adapter` section contains information on our USB Bluetooth adapter. The `WP530` section defines the mobile phone that I want to connect to. Don't worry about the exact details for the phone as we'll come back to that in a moment.
5. Bring up the Asterisk console by running `asterisk -rvvvv` (the number of v's is how verbose the console will be)
6. In this console, run 

  ````bash
  module load chan_mobile.so
  mobile search
  ````

  Give it a while. Ensure your phone is unlocked and Bluetooth enabled during this process. You should see your phone pop up in the list. Importantly, it should display a `Usable` column and a `Port` column - you want to make sure it says `yes` for usable. Take note of the `Port` value and change the value in `chan_mobile.conf` to match this if necessary.
7. To autoload `chan_mobile`, append to the end of `/etc/asterisk/modules.conf`:

  ````bash
  load = chan_mobile.so
  ````
8. At this point, the mobile configuration should be complete. Restart Asterisk by issuing:

  ````bash
  core restart now
  ````
  
  From the Asterisk terminal. After a short while, re-enter the Asterisk terminal. Once it has finished loading, run
  
  ````bash
  mobile show devices
  ````
  
  If you've set it up correctly, it should show something like:
  
  ````bash
  ID              Address           Group Adapter         Connected State      SMS
  WP530           0A:E2:10:C3:75:30 1     blue            Yes       Free       No
  ````
  
# Configuring call routing
We now need to setup Asterisk to create a SIP server and to also route calls made through this server to the mobile phone. To setup the SIP server, append the following to `/etc/asterisk/sip.conf`:

````bash
[test]
type=friend
host=dynamic
secret=test
dtmfmode=rfc2833
canreinvite=yes
nat=yes
qualify=yes
context=test
````

To connect to this server, from your client, enter the IP address of your Raspberry Pi. The username and password will both be `test`. For quick testing on my computer, I used [MicroSIP](http://www.microsip.org/) which worked quite well.

Now that the SIP server is setup, the routes need to be formed. Edit or create `/etc/asterisk/extensions_custom.conf`, and to it, add:

````bash
[test]
exten => _X.,1,Dial(Mobile/WP530/${EXTEN},45)
_X.,n,Hangup

[incoming-mobile]
exten => s,1,Noop(Accepting mobile call from ${DID})
exten => s,n,Dial(SIP/test)
````

This should route all numbers made to the SIP server through to your mobile phone. It **should** also route any calls made to the phone back to the SIP connected device, although I've had less luck on this side.

Once the configuration is done, restart Asterisk as before. That's it! We're done! After adding this SIP server configuration to my ATA, I can now make home phone calls that get pushed through my mobile phone. :)

# Adding SIP auto fallback
As it turns out, I wanted to slightly extend this a bit. Since I may not always have my phone present to make calls, I'd like it to auto-fallback to use the SIP providers that I'm already using. My ATA is not quite smart enough to do this itself, since it will just appear like the line is busy if Asterisk fails to connect the call. So instead, I've added my SIP providers directly to Asterisk:

1. For each provider, append to `/etc/asterisk/sip.conf`:

  ~~~~
  [telecube-out]
  type=peer
  secret=XXXXXXXXX
  username=YYYYY
  host=sip.telecube.net.au
  fromuser=YYYYY
  fromdomain=192.168.0.10
  canreinvite=no
  insecure=invite,port
  qualify=yes
  nat=yes
  context=from-telecube
  ~~~~
  
  Note: Replace `secret=XXXXXXXXX` with `secret=your-sip-password` and `YYYYY` with your username. I don't know if `fromdomain` does anything, but I set it to the IP address of my Raspberry Pi.
  
2. For routing, I've modified `/etc/asterisk/extensions_custom.conf` to:

  ~~~~~
  [test]
  exten => _183[12].,1,Dial(Mobile/WP530/${EXTEN},45)
  exten => _*2.,1,Dial(SIP/${EXTEN:2}@pennytel-out,30,r)
  exten => _*3.,1,Dial(SIP/${EXTEN:2}@telecube-out,30,r)
  exten => _X.,1,Dial(Mobile/WP530/1831${EXTEN},45)
  
  exten => _04!,2,Dial(SIP/${EXTEN}@telecube-out,30,r)
  exten => _13!,2,Dial(SIP/${EXTEN}@telecube-out,30,r)
  exten => _X.,2,Dial(SIP/${EXTEN}@pennytel-out,30,r)
  
  exten => _X.,3,Dial(SIP/${EXTEN}@telecube-out,30,r)
  _X.,n,Hangup
  
  [incoming-mobile]
  exten => s,1,Noop(Accepting mobile call from ${DID})
  exten => s,n,Dial(SIP/test)
  ~~~~~
  
  At the top level, it diverts all numbers through my mobile phone (line 4). You'll notice that it automatically prepends `1831` to the DID, unless `1831` or `1832` have already been specified by the user (line 1). `1831` (at least in Optus) prevents my caller ID from being displayed, while `1832` forces it to be displayed. Furthermore, if I prefix the DID with either `*2` or `*3` (lines 2 and 3), then these will be redirected to my SIP providers (I have both Telecube and Pennytel) - like a manual override.
  
  At the second level, if my phone isn't available, then mobile numbers (04 prefix) and 13 numbers get routed through Telecube, while everything else gets routed through Pennytel (taking advantage of what's cheaper with which provider). At the third level, if Pennytel fails, then everything gets routed through Telecube. Neat!
