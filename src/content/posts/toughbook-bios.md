---
title: AMI Aptio V Bios Password Spelunking
published: 2025-04-03
updated: 2025-04-03
description: 'Removing the BIOS password from a Toughbook CF-54 MK2'
image: ''
tags: [Hardware,BIOS,Soldering,Passwords]
category: 'Hardware'
draft: false 
---

## Background
I purchased a very well used Toughbook CF-54 MK2 from a local classifieds listing, with the description stating that it would not power on after an attempted repair. But for only $35, I couldn't say no, figuring if nothing else I could put a new motherboard in it for relatively cheap off of eBay. 

The story I was told was that an ex had stolen the laptop and put a BIOS password on it so he would no longer be able to use it, and I'm guessing he opened it to attempt to dump the EEPROM. After an apparent unsuccesfull attempt, while putting it back together, the ZIF connector on the motherboard for the serial, VGA, and GPS daughterboards was damaged and shorting something out enough that the laptop would no longer power on.

After getting it home and performing a full autopsy, I found several issues. The most apparent was the fact that several screws were partially stripped and in the wrong place, at least one crossthreaded, and a few missing. The charger I was given with it was technically the right size barrel plug but it was a generic Onn PSU rated at 20v instead of the 15.6v the Toughbook wanted, and not rated for nearly enough amperage. I cut the cable and connected it up to my benchtop PSU to see what happened.

As expected, I got no signs of life, and an amperage draw of 0.08a or so. Opening up the bottom cover, I removed the flex cable that was halfway jammed in the connector, and tried it again - it powered right up! I tried reseating the cable, but the connector was damaged enough that it would not slot all the way in. I could see where the cable was flexed and slightly kinked where the previous owner tried to force it in. I stuck the machine under my microscope and saw that several of the pins under where the plastic had broken away from the connector were bent and pushed down. I was able to push and prod them back in to place, and carefully reseat the connector without the guides on the side. Powering it on again, I had life, but that's when I ran into my next issue.

## BIOS Lock

After the Panasonic splash screen, I was greeted with a wonderful blue Enter Password prompt. After some searching, I found this Github Gist covering removing the password from a CF-U1 MK2. While being a slightly older machine, they do share using an AMI BIOS so I hoped it would give me a good starting point. Unfortunately, as is the case with most projects, it would lead me down a rabbithole to nowhere. 

The basic instructions for the guide are to download the corrosponding [Aptio AMI Firmware Update Utility](https://www.ami.com/support-other/ ) for your platform. For the U1, that would be Aptio 4, for the CF-54, that would be Aptio V (whyyy they would change the naming scheme from numbers to roman numerals I'm not sure). Luckily, despite a BIOS password, it would still boot into a live image no problem. After opening AFUWINGUI, you dump the BIOS, open it in [UEFITool](https://github.com/LongSoft/UEFITool), find the entry for AMITSESetup, and you should be able to XOR it and find the scancodes for each key of the password. In my case, since I'm just trying to remove it, you could just zero this section out, and reflash it using a CH341a programmer.

However, no matter what I tried, I could not get the AMI FUU to open, it would immediately crash due to a driver error. After spending too much time trying to solve that issue, I gave up on that approach, and tried Flashrom in a Linux live distro. This also failed, complaining it couldn't find the device. Hmm.

I found a [post on Level1Techs](https://winraid.level1techs.com/t/guide-usage-of-ami-s-aptiov-uefi-editor-fpt-flash-method/91842) detailing using IntelFPT to dump the bios, which was successful, but did not seem to have the region containing the BIOS password (although maybe I just missed it on this initial dump!).

## Disassembly

A quick sidenote on disassembly of the CF-54, as there aren't many videos on the teardown. It's mostly self explanatory apart from the keyboard, which is actually gued down for waterproofing. Just slide a plastic spudger along the top left corner until you can get it started and work your way across, being especially careful around the bottom right, as that's where the ribbon cables are. There are tabs along the bottom edge of the keyboard to locate it and a metal access panel with a gasket covering the ZIF connectors. Remove that and the keyboard comes right out, then it's just a matter of unrouting the many different antenna connectors that run to the display, and removing the bottom cover. It's not necessary to fully remove the motherboard to access the EEPROM, and if you're leaving it in place, you can leave the keyboard on. I did choose to go ahead and remove the motherboard so I can repaste the CPU though.

## Reading the Chip

My CF-54 uses an MX25L12873F BIOS chip, which runs at 3.3v (lucky for me, because my CH341a did not come with the 1.8v adapter). I used Neoprogrammer in Windows, mostly because I was already booted into Windows on my workstation and I didn't want to reopen all my tabs in Linux. My first attempt at reading using the chip clip failed, it wasn't happy about reading in circuit I guess. I popped it off with my hot air station and soldered it down to the SOIC8 board. It read sucessfully this time, and I was able to open it in UEFITool no problem.

## Modifying the Image

Here's where the CF-U1 instructions came back to bite me again. I figured once I was able to get a clean image in UEFITool, it would be fairly trivial to find and remove the password. Maybe I was just looking in the wrong place, but I just could not find where it was stored. I did see [this blog post](https://davedippenaar.wordpress.com/2020/04/02/aptio_bios_passwords/) which covered Aptio V specifically but it still didn't line up for me. I had to take a different approach.

During my research on this, I had seen a [few](https://www.badcaps.net/forum/troubleshooting-hardware-devices-and-electronics-theory/troubleshooting-laptops-tablets-and-mobile-devices/bios-requests-only/69290-panasonic-cf-54-bios-password) [Badcaps](https://www.badcaps.net/forum/troubleshooting-hardware-devices-and-electronics-theory/troubleshooting-laptops-tablets-and-mobile-devices/bios-requests-only/66415-panasonic-toughbook-cf-53-mk3-bios-password-removal) [threads](https://www.badcaps.net/forum/troubleshooting-hardware-devices-and-electronics-theory/troubleshooting-laptops-tablets-and-mobile-devices/bios-requests-only/79214-panasonic-cf-54-bios-password) where kind souls would remove the password for your dump for you, but I really wanted to learn the actual process. What I decided to do is download a few of the before and after files that were posted and open them up in a hex editor that had a comparison feature (I used [Hex Editor Neo](https://hhdsoftware.com/hex-editor)). I really must have just been looking over it in UEFITool because sure enough, I found the right section immediately. I opened my dump up in Hex Editor Neo and scrolled to the same line, on mine vs the dump I was looking at it was a bit further down, but the structure was nearly identical. I zeroed out that section, saved it, and flashed it back to the chip.

## Flashing the Chip

After my first flash, I put the chip back on the board, put it back together enough to power on, aaaand... Enter password screen again. Damn! At that point I had to turn in for the night and had other things to get to so I posted in the thread asking if someone could fix my dump. Even if I didn't learn exactly how to do it, I would at least have a working laptop. The following day, once I'm able to sit down and work on it again, I pull the chip, put it on my reader, and flash the file kindly provided by the Badcaps user who replied to me. But this time I poked around in the Neoprogrammer write settings and realized I didn't have flash verification turned on. Enabled that, flashed again, but it was failing every time on the same byte. A little searching later and it turns out this chip really likes to be erased before flashing, so I do that, reflash, and success! I soldered it back down, booted it up, and it dropped me straight into the BIOS, no password required.

## Next Steps

Now that the laptop is usable, my next project with it is to rebuild the battery. It did come with one but it lasts all of 30 seconds before hard powering off. I cracked it open to find some seemingly very non-standard cells, and I don't see any identification marks on them to find replacements. I did map out where on the charge board each of them hook up, and verified the cells are in a 3s2p configuration, so if I can find some replacement cells that fit in the housing, I should (hopefully) be good to go. I have yet to find out if the controller board wipes its memory when cells are disconnected ]like what happens with some Thinkpads.](https://forum.thinkpads.com/viewtopic.php?t=136939) 