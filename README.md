# Ulanzi-D200-Linux optimized System
Own created System for using Ulanzi D200 most efficiently under Linux

Prequisites:
Ulanzi D200 WITH root Access. Some critical remarks to the security issue hunters. At the beginning the Ulanzi D200 was sold with no set root password. I personally follow the philosophy that a device I own, should also be fully utilizable by me.
After the finding of that in my eyes quite uncritical issue (we have a device which emulates out of the box  a keyboard). The company sells the Ulanzi D200 with set root password. In that case there is only the sad way about the quite inefficient Ulanzi D200 Protocol (there are some projects for HomeAssistant, which makes the device usable for that and have succeed mostly in implementing the Ulanzi Protocol in python).

After some tries with the python implementation I did find the protocol quite complex and by luck my device s one with no set root password for adb.

So a first look into the filesystems on the Ulanzi Deck:
Partitionen (/dev/block/by-name/)

┌───────────┬────────────┬────────┬────────┬────────────────────────────────┬───────────┐
│ Partition │   Device   │ Blöcke │ Größe  │             Zweck              │   Mount   │
├───────────┼────────────┼────────┼────────┼────────────────────────────────┼───────────┤
│ uboot     │ rkflash0p1 │ 2048   │ 2 MB   │ Bootloader (U-Boot)            │ —         │
├───────────┼────────────┼────────┼────────┼────────────────────────────────┼───────────┤
│ trust     │ rkflash0p2 │ 1024   │ 1 MB   │ ARM TrustZone / Secure         │ —         │
├───────────┼────────────┼────────┼────────┼────────────────────────────────┼───────────┤
│ misc      │ rkflash0p3 │ 1024   │ 1 MB   │ Boot-Commandos (recovery-flag) │ —         │
├───────────┼────────────┼────────┼────────┼────────────────────────────────┼───────────┤
│ recovery  │ rkflash0p4 │ 18432  │ 18 MB  │ Recovery-Image                 │ —         │
├───────────┼────────────┼────────┼────────┼────────────────────────────────┼───────────┤
│ boot      │ rkflash0p5 │ 6144   │ 6 MB   │ Kernel + DTB                   │ —         │
├───────────┼────────────┼────────┼────────┼────────────────────────────────┼───────────┤
│ rootfs    │ rkflash0p6 │ 117760 │ 115 MB │ Root-Filesystem  (ext2)        │ /         │
├───────────┼────────────┼────────┼────────┼────────────────────────────────┼───────────┤
│ oem       │ rkflash0p7 │ 28672  │ 28 MB  │ OEM-Data  (ext2)               │ /oem      │
├───────────┼────────────┼────────┼────────┼────────────────────────────────┼───────────┤
│ userdata  │ rkflash0p8 │ 56303  │ 55 MB  │ User Data   (ext2)             │ /userdata │
└───────────┴────────────┴────────┴────────┴────────────────────────────────┴───────────┘

The usage of the filesystems:

┌────────────┬───────┬────────┬──────┬──────────┐
│   Mount    │ Size  │ Used   │ Free │ Percent  │
├────────────┼───────┼────────┼──────┼──────────┤
│ / (rootfs) │ 95 M  │ 95 M   │ 0    │ 100 % ⚠️ │
├────────────┼───────┼────────┼──────┼──────────┤
│ /oem       │ 27 M  │ 347 K  │ 25 M │ 2 %      │
├────────────┼───────┼────────┼──────┼──────────┤
│ /userdata  │ 52 M  │ 27 M   │ 23 M │ 54 %     |
└────────────┴───────┴────────┴──────┴──────────┘

So the Idea was to place a own created application alongside the original app on /userdata 
(and utilize the rest of space on userdata as file cache). Due to the full root filesystem was the only option to symlink the startup script:
The system uses busybox:
/etc/init.d/S51startApp.sh  ->  /userdata/startapp.sh   (Symlink, on rootfs)
/userdata/startapp.sh                                    (real Script, 148 B)
/userdata/ulanzi-device                                  (Renderer-Binary, 1,1 MB(my own control app)

My System focuses mainly on a desktop usage, therefore the Application is split into several parts:

┌───────────────┬───────────────┬───────┬───────────────────────────────────────────────────┐
│    Binary     │     Crate     │ Where │                        Job                        │
├───────────────┼───────────────┼───────┼───────────────────────────────────────────────────┤
│ ulanzi-device │ ulanzi-device │ Deck  │ Framebuffer/input/HID renderer + screensavers     │
├───────────────┼───────────────┼───────┼───────────────────────────────────────────────────┤
│ ulanzi-host   │ ulanzi-daemon │ PC    │ Main daemon: icons, actions, providers, nav, sync │
├───────────────┼───────────────┼───────┼───────────────────────────────────────────────────┤
│ ulanzi-tray   │ ulanzi-daemon │ PC    │ System-tray status & control                      │
├───────────────┼───────────────┼───────┼───────────────────────────────────────────────────┤
│ ulanzi-daemon │ ulanzi-daemon │ PC    │ Standalone/stock daemon mode (uses the orig app)  │
├───────────────┼───────────────┼───────┼───────────────────────────────────────────────────┤
│ ulanzi-gui    │ ulanzi-gui    │ PC    │ Config editor                                     │
├───────────────┼───────────────┼───────┼───────────────────────────────────────────────────┤
│ ulanzi        │ ulanzi-cli    │ PC    │ CLI control                                       │
└───────────────┴───────────────┴───────┴───────────────────────────────────────────────────┘

The ulanzi-daemon is from my side deprecated, far too complex protocol for a simple use case.

Key limitations vs. ulanzi-host

┌─────────────────────────────────────────┬───────────────────────────────────────┬──────────────────────────────────────┐
│                                         │         ulanzi-daemon (stock)         │       ulanzi-host (fb-channel)       │
├─────────────────────────────────────────┼───────────────────────────────────────┼──────────────────────────────────────┤
│ Talks to                                │ Original Ulanzi firmware (0x7C7C)     │ Our ulanzi-device renderer (0xAB)    │
├─────────────────────────────────────────┼───────────────────────────────────────┼──────────────────────────────────────┤
│ Nav stack                               │ None — Back/Home/Forward → start page │ Full stack + fwd + auto-nav overlays │
├─────────────────────────────────────────┼───────────────────────────────────────┼──────────────────────────────────────┤
│ Providers (Clock/Rules/Text)            │ No                                    │ Yes                                  │
├─────────────────────────────────────────┼───────────────────────────────────────┼──────────────────────────────────────┤
│ Screensavers / night / weather / glyphs │ No                                    │ Yes                                  │
├─────────────────────────────────────────┼───────────────────────────────────────┼──────────────────────────────────────┤
│ Clock                                   │ host heartbeat into small window      │ device-side desk clock               │
├─────────────────────────────────────────┼───────────────────────────────────────┼──────────────────────────────────────┤
│ BrightnessStep                          │ ignored?                              │ supported                            │
├─────────────────────────────────────────┼───────────────────────────────────────┼──────────────────────────────────────┤
│ Requires                                │ firmware ≥ 2.0.3 (per header comment) │ our flashed renderer running         │
└─────────────────────────────────────────┴───────────────────────────────────────┴──────────────────────────────────────┘
