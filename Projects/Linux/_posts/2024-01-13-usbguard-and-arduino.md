---
layout: post
tags: bash
---

Today I set up my laptop (the linux side) to program Arduino boards for a school project, but ran into a few hiccups along the way.

I installed the new v2 IDE without a problem, but when I connected my arduino uno board, nothing showed up, and after doing some googling and troubleshooting, I face-palmed; I had installed usbguard and forgotten about it.

What is usbguard, you may ask? Usbguard is a handy dandy framework for protecting your device from malicious usb devices. Essentially, it doesn't allow any new usb device to talk to your computer without explicit permission from you, the user.

However, it can become a little tedious to manually allow each device to connect to your computer every time you plug it in, *especially* when dealing with arduino (lots of plugging in and unplugging), especially if you want to allow it temporarily, as to do this you need the device ID.

To make my life easier, I wrote a quick bash script that searches for blocked devices and allows the first one to connect:

```bash
#!/bin/bash
ID=$(sudo usbguard list-devices --blocked | grep -oE 'id [0-9,a-z]*:[0-9,a-z]*' | grep -m 1 -oE '[0-9,a-z]*:[0-9,a-z]*')

# Check that length of ID is greater than 0.
if [ -n "${ID}" ]
then
        echo "Allowing device with ID ${ID} to connect."
        sudo usbguard allow-device $ID
else
        echo "No blocked devices were found :D"
fi
```


This bash script, which I named `allow-usb` and placed in my `~/bash_scripts` directory (a directory added to `$PATH` in my `.bashrc`), first runs `sudo usbguard list-devices --blocked` to list all devices actively being blocked by usbguard. The output of this after I plug in two arduino boards looks something like this:
```
29: block id 2341:0043 serial "..." name "" hash "..." parent-hash "..." via-port "1-1" with-interface { ... } with-connect-type "hotplug"
30: block id 2341:0062 serial "..." name "" hash "..." parent-hash "..." via-port "1-1" with-interface { ... } with-connect-type "hotplug"
```
Note that there is one line per device, and I've removed some of the irrelevant information and replaced it with '...'.

The script then runs two grep commands, one to match all the IDs, and one to match only the first ID:


```bash
grep -oE 'id [0-9,a-z]*:[0-9,a-z]*
```
Returns:

```ruby
id 2341:0043
id 2341:0062
```

Then we run:
```bash
grep -m 1 -oE '[0-9,a-z]*:[0-9,a-z]*'
```

Which returns:
```ruby
2341:0043
```

Then, we confirm that we did capture something with the if statement; the condition `[ -n "${ID}" ]` is true if the length of `ID` is more than 0. If we did capture an ID, we allow the device temporarily with:
```bash
sudo usbguard allow-device $ID
```

Otherwise, we inform the user that no devices were detected.

This way, when a device is plugged in, all you need to do is type `allow-usb` into the terminal, which, after authenticating with sudo, allows the device to connect to your computer.

Note that this is temporary, and the command must be run again if the device is disconnected and connected again; by adding the `-p` option to `usbguard allow-device`, the change can be made permanent, but for the sake of security, I don't want this to happen every time. I updated the script to plug the first parameter of the script into the `allow-device` command:
```bash
ID=$(sudo usbguard list-devices --blocked | grep -oE 'id [0-9,a-z]*:[0-9,a-z]*' | grep -m 1 -oE '[0-9,a-z]*:[0-9,a-z]*')

# Check that length of ID is greater than 0.
if [ -n "${ID}" ]
then
        echo "Allowing device with ID ${ID} to connect."
        sudo usbguard allow-device $ID $1
else
        echo "No blocked devices were found :D"
fi

```
(the `$1` is the first parameter)

This way, running `allow-usb -p` would permanently allow the device.