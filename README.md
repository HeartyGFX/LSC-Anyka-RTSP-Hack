# LSC-Anyka-RTSP-Hack
100% Local RTSP for LSC/Anyka/Tuya cameras


LSC Smart Connect / Anyka / Tuya : RTSP 100% Local
No Cloud · No App · No API Key
By HeartyGFX (Watson) & Holmes | April 2026

⚠️ Disclaimer
This hack is provided for educational purposes and personal use only.
Only use this on cameras you own.
We are not responsible for any damage caused.

🙏 Credits & Prior Work
This hack would not exist without the foundational work of :

Guino/LSCOutdoor1080P
→ Original hack.sh concept
→ SD card execution method
→ Binary patch approach (different firmware version)

MuhammedKalkan/Anyka-Camera-Firmware
→ Anyka firmware documentation
→ Alternative RTSP application

ricardojlrufino/anyka_v380ipcam_experiments
→ Anyka firmware research

Gerge - Anyka AK3918 Hacking Journey
→ Detailed hacking documentation

Our original contribution :

Discovery of /tmp/_ak39_factory.ini factory trigger
RTSP activation without binary patching
Compatible with firmware versions that cannot be patched
📋 Tested Hardware
text

Brand     : LSC Smart Connect Outdoor Camera (Action stores)
Reference : Article No. 3202092.2
Chip      : Anyka AK3918EV330
Firmware  : Tuya IPC SDK 4.9.18
🔑 The Key Discovery
The Anyka/Tuya firmware contains a dormant RTSP server.
It is activated by creating the file /tmp/_ak39_factory.ini at runtime.

This file triggers factory test mode, which starts the RTSP server on port 554.

This works on firmware versions that cannot be patched (different MD5 than Guino's patch requires).

📦 Requirements
text

- LSC Smart Connect camera (configured on LSC/Smart Connect app first)
- SD Card (4GB to 32GB)
- Formatted FAT32 + MBR partition table (⚠️ NOT GPT!)
- Windows PC with Telnet client enabled
- VLC Media Player
⚠️ Important Prerequisites
1. Camera must be initialized first
The camera MUST be configured via LSC Connect or Smart Connect app first.
This connects it to your local WiFi network.
This hack does NOT replace this initial setup step.

2. SD Card MUST be MBR formatted
Sandisk Ultra and other modern SD cards are GPT formatted by default → REJECTED by camera.

Format SD card via Windows diskpart :

cmd

diskpart
list disk
select disk X        (replace X with your SD card number)
clean
create partition primary
format fs=fat32 quick
assign
exit
📁 SD Card File Structure
text

SD:/
├── hack.sh      (original from Guino repo - DO NOT MODIFY)
└── custom.sh    (our script)
⚠️ Both files must be UTF-8 encoded with LF line endings (not CRLF)

📄 hack.sh
Use the original unmodified hack.sh from Guino's repo.

Do not modify it.

📄 custom.sh
Bash

#!/bin/sh

sleep 8

# Permanent telnet via rc.local
if [ ! -f /etc/config/rc.local ]; then
    mount -o remount,rw /etc/config
    printf '#!/bin/sh\ntelnetd -p 24 -l /bin/sh &\n' > /etc/config/rc.local
    chmod +x /etc/config/rc.local
fi

telnetd -p 24 -l /bin/sh &
🚀 Procedure
STEP 1 - Prepare SD Card
Format SD card : FAT32 + MBR
Copy hack.sh (original Guino) and custom.sh to SD root
Check line endings : LF only
STEP 2 - First Boot
Insert SD card into powered-off camera
Power on camera
Wait 60 seconds
STEP 3 - Enable Windows Telnet Client
text

Control Panel → Programs
→ Turn Windows features on or off
→ Check "Telnet Client"
→ OK
STEP 4 - Connect via Telnet
cmd

telnet 192.168.1.100 24
(replace 192.168.1.100 with your camera's IP)

You should get a direct root shell :

text

~ #
STEP 5 - Activate RTSP (Factory Trigger)
Bash

cat > /tmp/_ak39_factory.ini << 'EOF'
[config]
rtsp_enable = 1
rtsp_port = 554
factory_mode = 1
video_enable = 1
audio_enable = 1

[rtsp]
port = 554
enable = 1
main_stream = 1
sub_stream = 1
EOF

killall anyka_ipc
⚠️ The camera will speak and reboot. This is completely normal.

STEP 6 - Reconnect After Reboot
Wait 90 seconds then reconnect :

cmd

telnet 192.168.1.100 24
Verify RTSP is active :

Bash

netstat -tuln | grep 554
Should show :

text

tcp        0      0 0.0.0.0:554             0.0.0.0:*               LISTEN
STEP 7 - View Stream in VLC
text

Media → Open Network Stream
text

rtsp://192.168.1.100:554/main_ch
🎉 Live local video stream - no cloud!

📡 RTSP URLs
text

Main stream (HD) : rtsp://[CAM_IP]:554/main_ch   ✅ Validated
Sub  stream (SD) : rtsp://[CAM_IP]:554/sub_ch    ❓ Not validated yet
🔄 Automatic RTSP on Every Boot
For fully automatic RTSP without manual intervention, use this custom.sh :

Bash

#!/bin/sh

sleep 8

# Permanent telnet
if [ ! -f /etc/config/rc.local ]; then
    mount -o remount,rw /etc/config
    printf '#!/bin/sh\ntelnetd -p 24 -l /bin/sh &\n' > /etc/config/rc.local
    chmod +x /etc/config/rc.local
fi

telnetd -p 24 -l /bin/sh &

# RTSP factory trigger (once per boot)
if [ ! -f /tmp/_ak39_factory_done ]; then
    cat > /tmp/_ak39_factory.ini << 'EOF'
[config]
rtsp_enable = 1
rtsp_port = 554
factory_mode = 1
video_enable = 1
audio_enable = 1

[rtsp]
port = 554
enable = 1
main_stream = 1
sub_stream = 1
EOF
    touch /tmp/_ak39_factory_done
    sleep 3
    killall anyka_ipc
fi
⚠️ Note : /tmp/ is cleared on reboot.
This means the camera will reboot once on every power cycle to activate RTSP.
After that reboot, RTSP is active until next power cycle.

🔍 Technical Details
What we found inside the firmware
text

Binary        : /usr/bin/anyka_ipc
RTSP symbols  : ht_rtsp_start, ht_rtsp_stop
               ht_rtsp_send_video_frame
               ht_rtsp_send_audio_frame
RTSP library  : librtsp (compiled in)
Factory file  : /tmp/_ak39_factory.ini (trigger)
RTSP port     : 554
Memory layout
text

MemTotal  : ~32MB (very constrained)
/usr      : squashfs (read-only)
/etc/config : jffs2 (read-write, persistent)
/tmp      : tmpfs (read-write, lost on reboot)
Why Guino's patch doesn't work here
text

Expected MD5 : 5ac1f462bf039ec3c6c0a31d27ae652a (v2.10.36)
Actual MD5   : 36849ada9f7fc1e6ea27a986cfbee8d0 (older version)
The binary version is different → patch rejected.
Our method bypasses this completely.

✅ Compatible Software
text

VLC Media Player    ✅
Frigate (NVR)       ✅
Home Assistant      ✅ (Generic Camera integration)
Blue Iris           ✅
Shinobi             ✅
FFmpeg              ✅
⚠️ Known Limitations
text

- SD card required for persistence
- One reboot per power cycle to activate RTSP
- Factory mode may disable some Tuya cloud features
- Tested on LSC Article No. 3202092.2 only
- Other LSC/Anyka variants : untested (feedback welcome)
🤝 Contributing
Tested on a different camera variant ?
Found the sub_ch URL ?
Found a way to make RTSP permanent without SD ?

Open an issue or submit a PR !

📜 License
MIT License - Free to use, share, modify.

If this helped you → ⭐ Star the repo

"Vise la lune, même si tu ne l'atteins pas, tu finiras parmi les étoiles"
La Philosophie de l'Oiseau 🐦

HeartyGFX (Watson) & Holmes | 2026 🔍

