# ALSA device discovery and settings

## Intro
ALSA appears to be the default audio device module & userspace suite for Debian-based distros.

ALSA assigns device names and card numbers to USB audio devices.

## Goals

1. For babybell's proper operation, we need to find the correct devices and card numbers, in order to connect to and use them:
- USB microphone
- USB sound card `line out`

2. We need to control `mic` gain and `line out` volume.

## Discovery
### Get list of device names and card numbers
From the ALSA suite, `arecord -l` lists available devices (that's lower-case L):
```
$ arecord -l
**** List of CAPTURE Hardware Devices ****
card 0: Device [USB PnP Sound Device], device 0: USB Audio [USB Audio]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 3: Device_1 [USB PnP Sound Device], device 0: USB Audio [USB Audio]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
```

### Parse out the interesting bits

The portions we're interested in is covered by:

`r"^card (?P<card_num>\d+): (?P<dev_name>Device_?\d*).*?, device (?P<dev_num>\d+).*$"`

For example:

`card 0: Device [USB PnP Sound Device], device 0: USB Audio [USB Audio]`

yields

```{"card_num": "0", "dev_name": "Device", "dev_num": "0"}```

and

`card 3: Device_1 [USB PnP Sound Device], device 0: USB Audio [USB Audio]`

yields

`{"card_num": "3", "dev_name": "Device_1", "dev_num": "0"}`

Additional detail is available via `arecord -L` (upper-case L). 

## Change device settings

### Get mic sub-device name
Now we need to see sub-device names within a particular USB audio device: 
```
$ amixer -c Device_1 scontrols
Simple mixer control 'Mic',0
Simple mixer control 'Auto Gain Control',0
```
`'Auto Gain Control'` sounds potentially useful, but let's go with what we know works for the moment.

### Check mic gain
Let's see default mic gain, post-boot.
```
$ amixer -c Device_1 sget Mic,0
Simple mixer control 'Mic',0
  Capabilities: cvolume cvolume-joined cswitch cswitch-joined
  Capture channels: Mono
  Limits: Capture 0 - 12
  Mono: Capture 12 [75%] [17.85dB] [on]
```

### Set mic gain
To start, we're going to manually set mic gain to 100%.
```
$ amixer -c Device_1 sset Mic 100%
```

Check the gain again
```
$ amixer -c Device_1 sget Mic,0
Simple mixer control 'Mic',0
  Capabilities: cvolume cvolume-joined cswitch cswitch-joined
  Capture channels: Mono
  Limits: Capture 0 - 16
  Mono: Capture 16 [100%] [23.81dB] [on]
```

## Recording
While we won't be recording via ALSA CLI, here's how it's done:

```
arecord -r 24000 -f S16_LE -D hw:3,0 -t wav filename.wav
```

24000 is what the VAD and ASR models are trained at, iirc.

## How about Python?
Since babybell is a Python app, let's figure out how to do device discovery and change settings there. 
