---
page: https://idle.run/baby-raspi
title: "Raspberry Pi Camera Stream to YouTube Live"
tags: ffmpeg raspberry pi youtube stream
date: 2018-11-10
---

## Overview

Forward video and audio from Raspberry PI to Youtube live stream

## Requirements

Install ffmpeg binaries. Possibly from my ffmpeg-raspi project

## Hardware
- Raspberry Pi 3B+
- Pi Camera Module v2 NOIR
- USB microphone

## Camera

Enable `bcm2835-v4l2` and have it loaded on startup

As root:
```
modprobe bcm2835-v4l2
echo bcm2835-v4l2 >> /etc/modules
```

### raspivid preview

Test camera working correctly on raspivid:

*Note*: The camera preview renders directly to the display. It will NOT show up on VNC

```
raspivid -t 0
```

### v4l2-ctl

Customize as preferred.
`--overlay=1` enables display of the video on the monitor connected to the PI. Useful for debugging, but can be set to 0 most of the time.

`-p` set the target FPS

`pixelformat=4` sets the PI to output frames in H264 mode directly

Other params are self explanatory.

```
v4l2-ctl --set-fmt-video=width=1280,height=720,pixelformat=4
v4l2-ctl --overlay=1
v4l2-ctl -p 30
v4l2-ctl --set-ctrl=video_bitrate=1000000
```

### Capture to file

Test recording to file with `ffmpeg`. Copy the file off the device and play it to test it works as expected
```
ffmpeg -y -ac 1 -f alsa -i hw:1 -f h264 -framerate 30 -i /dev/video0 -vcodec copy -acodec aac -ab 128k out.mp4
```

## YouTube

With default encoding

```
ffmpeg -ac 1 -f alsa -i hw:1 -f h264 -framerate 30 -i /dev/video0 -vcodec copy -acodec aac -ab 128k -f flv rtmp://a.rtmp.youtube.com/live2/$ID
```

With ffmpeg encoding
```
ffmpeg -ac 1 -f alsa -i hw:1 -f video4linux2 -framerate 30 -i /dev/video0 -fflags nobuffer -vcodec h264_omx -g 5 -b:v 1000k -tune zerolatency -acodec aac -ab 128k -f flv rtmp://a.rtmp.youtube.com/live2/$ID
```

Replace $ID with an ID generated from a new Youtube Live Event created here:
https://www.youtube.com/my_live_events

Throw that command in an upstart, systemd, or other process manager script and you are good to go.


## Notes

It's possible to skip `bcm2835-v4l2` and instead pipe `raspivid` into ffmpeg. The problem is that audio and video
will be very out of sync. For reference this is the command:

```
raspivid -t 0 -o - -b 1000000 -md 4 -w 1280 -h 720 -fps 30 -awb auto -ex nightpreview -ev -2 -drc med -pts | \
  ffmpeg -thread_queue_size 128 -y -f h264 -r 30 -i - -ac 1 -f alsa -i hw:1  -c:v copy -c:a aac out.mp4
```


## Refs

- https://github.com/DeTeam/webcam-stream/blob/master/Tutorial.md