---
layout: post
title: Transferring My N64 Saves
date: "2015-04-11 16:41"
comments: true
image: 2015/04/DSC09669.JPG
published: true
---

<p class="intro"><span class="dropcap">W</span>hen I was younger, I would spend weekends after school playing through the Legend of Zelda: Ocarina of Time. After a while, I beat it. I remember the fun I had navigating through puzzles and chasing down heart containers. I loved every minute of it.
</p>

About a year ago, I discovered that my Ocarina of Time cartridge was on borrowed time. It turned out that this was one of a handful of cartridges in the Nintendo 64 library that used a battery to maintain its save file. According to the trustworthy Internet, these batteries have a lifespan between 15 and 20 years. So I did what any other dreamy eyed gamer would do. I got out my N64 and started tinkering with it.

I created a project with one goal in mind. To transfer all game saves from my minuscule N64 library into a digital format that I could use on other devices. I had four games that I needed to get saves off of.

- The Legend of Zelda: Ocarina of Time
- Star Wars Episode One: Racer
- Mario Party 3
- Super Mario 64


Here is how I did it.

####EEPROM Saves (Star Wars, Mario Party, and Super Mario):
- Dump ROM
- Transfer save to memory pack using GameShark
- Transfer memory pack data to computer using DexDrive
- Graft a new save file from DexDrive save with a hex editor
- Load in Project64 and play

####Ocarina of Time Save:
- Dump ROM
- Dump contents of active RAM (with my glorious Pentium III)
- Recalculate save checksum
- Graft a new save file from dumped data with a hex editor
- Load in Project64 and play

![](/assets/img/2015/04/DSC09658.JPG)

###Dumping My ROM
Star Wars, Mario Party, and Super Mario 64 use EEPROM to store their game save. This made my project easy. The first thing that I did was dump each game using the GameShark’s parallel port with the open source software [N64RD](https://github.com/parasyte/n64rd) using the command below. This program allowed me to back up each of my games for later use in my project. I used [this wiki](http://doc.kodewerx.org/hacking_n64.html) here to know what addresses and offsets for RAM and ROM space I needed.

`$ ./n64rd -dgame.n64 -a 0xB0000000 -l 0x02000000`

###Transferring Saves
To transfer the EEPROM games, I used a GameShark (I chose v3.2) to transfer the save from my cartridge to the memory pack. Once I transferred every game over to the memory pack, I used my Nintendo 64 DexDrive (Pictured below) to move the save files onto my computer. But each save file was wrapped inside a proprietary container. When examined with a hex editor, I noticed that each file was still there and all I had to do was cut out the bytes that made up the file container. Each one of the save files was structured in a different way.

![DexDrive](/assets/img/2015/04/DSC09655.JPG)

I took known game save copies from the internet for each of my EEPROM games and matched them up to my DexDrive save contents. I simply grafted over the matching bytes from the DexDrive save into the existing save files, loaded them up in my Project 64 emulator, and everything worked without a hitch. The only save that I had an issue with was Zelda Ocarina of Time. That is because I later found out that the SRAM save format was too large for the GameShark to transfer. So GameShark compressed it. Having no idea how it was compressed, I thought I was out of luck. I contacted Interact and they were unable to provide me with information on the algorithm the GameShark used.

###Hitting a Wall
The first thing that I tried was dumping the SRAM by using the GameShark. However, I was unable to find any documentation on how to pull the SRAM data or if it was even accessible from the GameShark. But I did find [someone](https://www.assemblergames.com/forums/showthread.php?31850-Dumping-N64-Game-Saves-with-a-Gameshark-with-LPT-access&p=517929&viewfull=1#post517929) who was trying to dump the contents of their SRAM into the memory pack controller by uploading it with [gsuploader](https://github.com/ppcasm/gsuploader). According to their post, they had some luck with a few games but were unlucky with others.

I was unable to get sram2mpk.bin working with my Ocarina of Time game. I spent a whole day attempting to load the program, but every time it locked up on me and dumped no information. I was able to verify (with the help of Lawrence) that gsuploader was working correctly by loading up an older project named [Neon64](https://github.com/mikeryan/n64dev), an SNES emulator for the N64.

![](/assets/img/2015/04/2015-03-21_19.19.01.jpg)

This led me to conclude that sram2mpk.bin was unlikely to work for my game. I thought I had reached a dead-end. That is until I discovered [this wiki](http://wiki.spinout182.com/w/Ocarina_of_Time:_Save_Format) that documented the Ocarina of Time save file.

###A New Plan

The documentation on the wiki claimed that save files were built by copying over a certain block of memory. After reading through this, I went back to my GameShark and dumped the entire contents of its RAM (using N64RD) while in game using the command below with N64RD.

`$ ./n64rd -dmemory.n64 -a 0xB0000000 -l 0x007FFFFF`

After opening up the RAM dump with my hex editor, I followed up to the address 0x0011A790 (or 0x8011A790 inside my GameShark memory) and copied 0x1450 bytes (the size of the game save file).

![Native N64 (Big Endian)](/assets/img/2015/04/memoryDump.png)

Once I had my save file, I converted it from N64 native to something that an emulator could read (big endian -> little endian).

I used [uCON64](http://ucon64.sourceforge.net/#ucon64) to convert Big Endian->Little Endian (vise versa) with this command:

`ucon64 --n64 --nint --dint --swap2 save.sra`

![Memory Swaped](/assets/img/2015/04/memoryDumpSwap.png)

I tried grafting over my completed save data from RAM into an existing Ocarina of Time save file. However, when I tried to load it up, I discovered that the save file was corrupt. I took a look back at what I had done. I talked it over with Lawrence and we concluded that my save file checksum that was dumped from RAM was probably bad. After reading the [wiki](http://wiki.spinout182.com/w/Ocarina_of_Time:_Save_Format) again, I tried to see if I could calculate and verify my own checksum.

###Discovering the Checksum
After a lot of digging around to see how the checksum is calculated, I was able to figure out the exact algorithm used. I took an existing Ocarina of Time save file, converted it back to N64 native (little endian -> big endian), and tried to see if I could calculate the checksum over the correct bytes (against a correct known checksum) using a list of different algorithms. I found one that matched and it was using [010 Editor](http://www.sweetscape.com/010editor). UShort (16 bit) – Big Endian.

![](/assets/img/2015/04/Checksum.png)

Lawrence and I were unable to find any information online about this algorithm and we wished to leave a way that others could follow these instructions using open source software. Lawrence contacted SweetScape (Company who created 010 Editor) asking if they could make available for us more information on the algorithm used and [Graeme Sweet](http://www.sweetscape.com/companyinfo/) very generously provided us with information on how “UShort (16 bit) – Big Endian” was calculated. Lawrence and I created a software tool ([Ocarina Checksum Checker](https://github.com/Vi1i/OcarinaChecksumChecker)) to calculate the checksum of an Ocarina of Time save file in native N64 format. The working source code has been posted. The instructions on how to run and calculate it can be found on the GitHub page.

###Putting it All Together

Now that I had the correct checksum algorithm, I recalculated my memory dump and discovered that my checksum from RAM was in fact wrong. I converted it from N64 format to something more readable (big endian -> little endian), grafted its contents into a known working Ocarina of Time save file, and loaded it up in my emulator. It worked. I was able to load up my Ocarina of Time save file on the emulator on my computer with the ROM that I had dumped and the save file I made.

![](/assets/img/2015/04/DSC09668.JPG)

###Final Thoughts
I managed to up-convert a younger piece of my childhood (with help) into the present digital age. I had many roadblocks and a few of my ideas were unexplored. If I had to do it all over again, I wouldn’t do it any other way. This was fun.

It didn’t take long before I learned that the community had made a [high definition texture pack]( http://www.emutalk.net/threads/51481-Zelda-Ocarina-of-time-Community-Retexture-Project-V6-Development-Topic) for specific graphic plug-ins. Of course I had to give it a try. This was the result.

![](/assets/img/2015/04/zelda01.jpg)
![](/assets/img/2015/04/zelda02.jpg)
![](/assets/img/2015/04/zelda03.jpg)

I hope others found this post informational and that it may help those with old N64 saves repeat what I have done.


Materials I needed to complete my project:

- N64 & Controller
- 8 MB RAM expansion pack
- Memory pack
- GameShark (with a working parallel port)
- Parallel port cable (and a machine with a parallel port)
- N64 DexDrive (and a machine with a serial port)

Software and resources I used:

- [N64RD](https://github.com/parasyte/n64rd)
- [010 Editor](http://www.sweetscape.com/010editor/)
- [uCON64](http://ucon64.sourceforge.net/#ucon64)
- [Project64](http://www.pj64-emu.com/downloads/project64/binaries/)
- [Zelda high definition texture pack]( http://www.emutalk.net/threads/51481-Zelda-Ocarina-of-time-Community-Retexture-Project-V6-Development-Topic)
- [Our Custom Ocarina Checksum Checker](https://github.com/Vi1i/OcarinaChecksumChecker)
