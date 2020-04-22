I switched over to using a Mac in 2013 and back then, the development experience, from my perspective, was great. The hardware was fast and reliable and the operating system got out of the way and provided everything I needed.

Unfortunately, since I got my new 2016 Macbook (the one with the touchbar), the quality of both the hardware and software, especially in terms of robustness and stability has somewhat degraded.

This situation led me to investigate if it's possible to run Linux on a Macbook - mostly to try it out. The last time I tried a Linux distro as my main development OS was quite a while ago and back then it was exhausting to set up and maintain.

First, I was put off, because for my model Audio doesn't work (there are no working drivers for it). So the first time I investigated this, I jumped ship, judging it to be too much effort and likely too much frustration.

However, then a colleague at work actually got Linux to run on his Macbook and didn't have any severe issues. This motivated me to poke at it again.

I investigated USB audio and wifi sticks and saw that they're actually rather cheap and, especially for audio, there are no compatibility issues.

If you're wondering how the driver situation looks for your Macbook model, [here](https://github.com/Dunedan/mbp-2016-linux) is a nice overview of "The State of Linux on 2016 & 2017 models".

## Steps

*Disclaimer: We're going to resize partitions, reformat things and install an operating system. All of this can go wrong and you might lose data, brick your machine or whatever, so backup everything you need first.*

The first step was to partition my disk. This was already a bit annoying and not particularly beginner-friendly. The goal was simple - make a non-apfs partition.

In order to do that, in a default Catalina or later setup, we first need to resize the APFS container. I checked out [this](https://www.macobserver.com/tips/deep-dive/resize-your-apfs-container/) tutorial, however, I ran into an error (49153) because of my time machine backups.
I used [this](https://stackoverflow.com/questions/46424915/apfs-container-resize-error-code-is-49153) answer here to fix this.

Basically, the steps are:

```bash

sudo tmutil disable
tmutil listlocalsnapshots /
tmutil thinlocalsnapshots / 9999999999999999

diskutil list
sudo diskutil apfs resizeContainer disk0s2 750g jhfs+ Extra 250g

sudo tmutil enable
```

You can do this on a live file-system and it took half a day to complete on my machine.

Once the container is resized, you have some unallocated space. You can either format this now to MS DOS (FAT), or during the Ubuntu installation later.

The next step is to download the [ubuntu iso](https://ubuntu.com/download/desktop). I used 18.04 LTS for driver availability and stability.

After the download is complete, create a bootable USB stick with the image. I won't go into any detail how to do this here, but there are plenty of tutorials and tools for this on the web. On OS X I used [etcher](https://www.balena.io/etcher/) to burn the image to a USB.

Alright. Before we install Ubuntu, we need to install `rEFInd` - a bootloader we can use for both Ubuntu and OS X, so we can switch between the two OSes.

Installing rEFInd is installed very well in [this](https://www.maketecheasier.com/install-dual-boot-ubuntu-mac/) article under Section `3. Prepping Your Drive`.

I simply followed the instructions there and it worked perfectly.

Once the bootable USB is done and rEFInd is correctly installed, restart the Macbook and insert the USB stick. It should boot into the rEFInd boot menu automatically and you can select the Linux USB there. 

From here, just follow the Ubuntu installation process, selecting your created partition (or create a new one from the unallocated space) as a location and install it the way you like.

Once the installation is complete, you'll likely boot into Ubuntu automatically. Here we need to reset the boot order to use rEFInd first. This is very well explained [here](https://www.rodsbooks.com/refind/bootcoup.html#efibootmgr).

You basically list your boot order and then try to find rEFInd Boot Manager in there. Then, you can adjust the boot order using the `efib00tmgr -o` command. Once that is done, rebooting will boot us into the rEFInd boot manager again and we'll have dual-boot enabled.

Once we're able to boot up Ubuntu, we can go about installing some necessary drivers. [This](https://github.com/Dunedan/mbp-2016-linux) is a great resource for finding what will work out of the box for your model, what will be missing and how to install it.

Depending on what does and doesn't work for your model, it's possible to use external USB hardware (e.g. for WIFI/Audio) instead.

## Annoyances / Issues

As I noted above, there are some issues I ran into with my new Ubuntu-on-a-Macbook setup. The most notable was, that the internal audio just doesn't work, so I needed to buy a USB audio stick, which costs very little and works nicely.

A bit more annoying is WIFI, which works, or doesn't work, based on the Access Point and on the provided network. So at home, with a 2.4ghz network, it works for me, but when I was on the road, it didn't work with most WIFIs, so I also got an external WIFI stick.
For the external WIFI stick (also not expensive), you have to be a bit careful in terms of compatibility, as not all will work out of the box, or well at all, on Linux.

Battery Life isn't as good. This isn't much of an issue for me, as I almost never work in battery mode. For my model with the useless TouchBar, I also have to issue a certain command at startup, in order to be able to use my F keys, which is a bit annoying, but at least easy to handle.

Sleep Mode is another problem, especially in my setup, where my Macbook is closed most of the time. So if I turn off my screen, or put the Macbook to sleep, it's impossible to recover.

For "normal" sleep with an open lid, there are some solutions, but waking up from sleep seems to be an issue in general.

These things are no deal breakers for me, but they might be for many people. They are also annoying enough that in the mid-term, I plan to move to a more native hardware environment for Linux.

## Conclusion

All in all, this wasn't too hard to do and so far (after 3 months) I'm very happy with Ubuntu as my main development setup. The current setup with the Macbook is still a bit cumbersome, so the next step is to sell it and get a machine where everything works out of the box.

If you're planning to switch as well and didn't know it's possible, or you were afraid to try it because it seemed a bit daunting, I hope this post was helpful to you. :)

#### Resources

* [How to Install and Dual-Boot Ubuntu on Mac](https://www.maketecheasier.com/install-dual-boot-ubuntu-mac/)
* [mbp-2016-linux](https://github.com/Dunedan/mbp-2016-linux)

