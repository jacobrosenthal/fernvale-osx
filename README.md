#fernvale-osx

Fernvale (Mediatek) OSX serial driver source. The device shows up at as /dev/tty.usbmodem1410 for me. I haven't had success usb loading yet so I can't say for sure the drivers are fully working, but the board acts just like it does under linux with the udev changes. Also I've altered the match for an Arduino serial interface which worked perfectly. Timing may be an issue. @thesourcerer8 added make test which uses hub-ctrl which doesn’t appear to have an OSX equivalent.

##build
You need to disable signing which became mandatory at some point recently
https://apple.stackexchange.com/questions/163059/how-can-i-disable-kext-signing-in-mac-os-x-10-10-yosemite
```
sudo nvram boot-args=kext-dev-mode=1
```
and reboot.

Then load up in AppleUSBCDC.xcodeproj in Xcode and build. Expand Products in the file navigator and right click to show in finder. open a terminal at that location:
```
sudo cp -R AppleUSBCDCData.kext /tmp/
sudo kextutil -t /tmp/AppleUSBCDCData.kext
```
if you need to unload it for some reason
```
sudo kextunload -vt /tmp/AppleUSBCDCData.kext
```

If you want to make it permanent. (careful with this, if you install a bad driver permanently it’ll be a real pain to fix)
```
sudo cp -R AppleUSBCDCData.kext /System/Library/Extensions/
sudo chown root:wheel /System/Library/Extensions/AppleUSBCDCData.kext
```

##info

Fernvale wouldn’t create a /dev/tty for me under OSX Yosemite.

dmesg reported:
```
AppleUSBCDCACMData: Version number - 4.2.1b5, Input buffers 8, Output buffers 16
AppleUSBCDC: Version number - 4.2.1b5
```

At first I hoped to make a codeless kext but I'm currently under the impression thats impossible for serial devices. 

I was able to take the [darwin sources](https://opensource.apple.com/tarballs/AppleUSBCDCDriver/) apart and enable logging to find this much:
```
Jan 24 03:14:48 jacob-2 kernel[0]: USB (XHCI Root Hub USB 2.0 Simulation):Port 1 on bus 0xa connected or disconnected: portSC(0xe0206e1)
Jan 24 03:14:49 jacob-2 kernel[0]: 0        0 AppleUSBCDCECMData: start - Find CDC driver for ECM data interface failed
Jan 24 03:14:49 jacob-2 kernel[0]: e6005000        0        0 AppleUSBCDCACMData: start
Jan 24 03:14:49 jacob-2 kernel[0]: e6005000        0        0 AppleUSBCDCACMData: initStructure
Jan 24 03:14:49 jacob-2 kernel[0]: e6005000        0        0 AppleUSBCDCACMData: findCDCDriverAD
Jan 24 03:14:49 jacob-2 kernel[0]: e6005000 0 0xffffff8035b43a00 AppleUSBCDCACMData: findCDCDriverAD - CDC driver candidate
Jan 24 03:14:49 jacob-2 kernel[0]: e6005000 0 0xffffff8035b43a00 AppleUSBCDCACMData: findCDCDriverAD - Found our CDC driver
Jan 24 03:14:49 jacob-2 kernel[0]: e6005000        0        0 AppleUSBCDCACMData: start: Found the CDC device driver
Jan 24 03:14:49 jacob-2 kernel[0]: e6005000        0        8 AppleUSBCDCACMData: start - Number of input buffers requested
Jan 24 03:14:49 jacob-2 kernel[0]: e6005000        0       10 AppleUSBCDCACMData: start - Number of output buffers requested
Jan 24 03:14:49 jacob-2 kernel[0]: e6005000        8       10 AppleUSBCDCACMData: start - Buffer pools (input, output)
Jan 24 03:14:49 jacob-2 kernel[0]: e6005000 0 0xffffff8035861400 AppleUSBCDCACMData: createSerialStream
Jan 24 03:14:49 jacob-2 kernel[0]: e6005000        0        0 AppleUSBCDCACMData: createSerialStream - NON WAN CDC Device
Jan 24 03:14:49 jacob-2 kernel[0]: e6005000        0        0 AppleUSBCDCACMData: getPortNameForInterface
Jan 24 03:14:49 jacob-2 kernel[0]: e6005000        0        0 AppleUSBCDCACMData: findControlDriverAD
Jan 24 03:14:49 jacob-2 kernel[0]: e6005000        0        0 AppleUSBCDCACMData: findControlDriverAD - No AppleUSBCDCACMControl drivers found (iterator)
```

After a weekend of testing and studying, with credit to [OS X and iOS Kernel Programming by Ole Henry Halvorsen](http://www.apress.com/9781430235361-4892)  I was actually able to alter the plist to match against the Mediatek data/comm interface while making no additional source code changes. One theory I currently have is that the Mediatek chip has its data/comm interface on 0 instead of 1 where Arduino/etc seem to have it and that whats throwing off the default matching.

##todo
* confirm it works - successfully fernvale-usb-load
* ideally we'd just be able use a codeless kext pointing at the existing USBCDC driving.. 
