fauxstream
==========

fauxstream is a wrapper script for ffmpeg to simultaneously record a
desktop video stream and one (or more?) audio streams on OpenBSD.
The goal is to circumvent limitations with other streaming/recording
solutions on OpenBSD, including simpler invocations of ffmpeg(1). It
allows screencasting including monitoring and microphone audio streams.
The license is ISC (see head of fauxstream).

Rationale
---------

To my knowledge, there is no dedicated screen casting solution available
on OpenBSD as of the time of this writing. The port of ffmpeg can
fulfill these functions, but seems to be hampered by apparent lack of or
suboptimal multithreading support, resulting in dropped frames when
recording.

It turns out that the reason for this is apparently lack of (or too
little) parallelism on the side of ffmpeg.

This script circumvents this by running multiple parallel recording
streams communicating via pipes.

Note that this is likely only useful on multiprocess systems/kernels.

In addition, fauxstream simplifies the innumerable possible
configurations of ffmpeg for the purpose of streaming with sane defaults
and a manageable number of options.

Requirements
------------

* ffmpeg (`# pkg_add ffmpeg`)
* OpenBSD/ksh
* sndiod(8) monitoring stream `snd/mon` (see [FAQ 13](https://www.openbsd.org/faq/faq13.html#recordmon))
* enable kern.audio.record (if using the microphone stream):
  ```
  # sysctl kern.audio.record=1
  ```

Preparations
------------

Things to consider before starting a stream on OpenBSD.

```
# sysctl kern.{audio,video}.record=1	# enable microphone and webcam recording
$ apm -H 				# increase hw.setperf performance level
$ ifconfig 				# check connection (prefer ethernet)
$ sndioctl input.level=1.0		# increase microphone input volume
```

Also test the game audio balance; lower game audio volume in the game's options preferably, and avoid using `-vmic`/`-vmon` if possible.

Known Limitations
-----------------

* Higher recording resolutions, non-default video codecs, or
  simultaneously running applications may affect performance,
  to the point of dropping frames and leading to desynchronization.
* The recording with ffmpeg's x11grab doesn't register when an
  application goes fullscreen, and it continues to record it as being
  run in a window.
* Likely significantly worse performance on single-core CPUs/
  single-process systems.

Usage
-----

```
fauxstream [-v] [-p <preset>] [-vmon <factor>] [-m [-vmic <factor>]]
	[-d <mic_device>] [-mf] [-ab <audio_bitrate>] [-vb <video_bitrate]
	[-r <size> [-o <window_offset>] | -fullscreen | -n <name>]
	[-s <scaled_resolution>] [-q qualitypreset]
	[-f <framerate>] [-a <audio_offset>] <target>

-a:	set audio offset (in seconds; can be negative and can be fractions)
-ab:	audio bitrate (default: 96)
-d:	set microphone device (default: snd/default; see `-m`)
-f:	set video framerate (default: 30)
-fullscreen: set video size & offset to root window geometry (supersedes -r and -o)
-m:	enable microphone stream (in addition to monitoring stream)
-mf:	microphone filter - filters out frequencies not in voice spectrum, as well as noise
-n:	set video size to geometry of named window (supersedess -r, -o, and -fullscreen)
-o:	set video offset (from top left; default: +0+0)
-p:	use preset (e.g. `vaapi`)
-q:	set quality preset (for libx264 codec; defaults to `ultrafast`)
-r:	set video size (resolution; default: 1280x720)
-s:	resolution to scale to (e.g. `1600x900`)
-vb:	set video bitrate (default: 3500)
-vmic:	factor to adjust volume of the microphone stream
-vmon:	factor to adjust volume of the monitoring stream

The target can be a file or a remote streaming address; special TCP configuration will be applied to `rtmp://`, `rtmps://`, `rtmpe://`, `rtmpts://`, and `rtmpe://` (`SO_KEEPALIVE`, see getsockopt(2).
```

Examples:
-----------

### Stream to PeerTube:

```
$ fauxstream -m -r 1920x1080 -f 30 -a -0.2 \
	"rtmp://my.peertube.instance:1935/live/<STREAM_KEY>
```

### Stream to Twitch:

```
$ fauxstream -m -r 1920x1080 -f 30 \
	"rtmp://<SERVER>.twitch.tv/app/<STREAM_KEY>
```

### Record to file:

```
$ fauxstream [...] /path/to/file.flv
```

### Record with separate microphone device (e.g. USB microphone):

Note in this example, the microphone device is attached to `/dev/audio1` which is accessed as `snd/1`.

```
$ fauxstream -m -d snd/1 ...
```

### Record with VAAPI hardware encoding

Note this needs the required libraries. On OpenBSD, check for package `intel-media-driver`. Run `vainfo` from package `libva-utils` to check if VAAPI is supported.
```
fauxstream -p vaapi /path/to/file.flv
```

### Record a rectangle of size 1280x720, offset from the upper left corner by 200 pixels to the right, and 100 pixesls down

```
faustream -r 1280x720 -o +200,100 /path/to/file.flv
```

FAQ:
----

**Q: What does the name stand for?**

A: It used to stand for **f**fmpeg + **au**cat **x**11 **stream**, but
since aucat was dropped, the name isn't an abbreviation for anything in
particular anymore.

**Q: How do I stop the recording??**

A: Press Ctrl-C to stop the recording.

**Q: I'm trying to record a specific window, but why does moving it
or covering it not keep recording it?**

A: Unfortunately, it's just recording the screen geometry of where
the window was when you started recording. Any overlapping windows
will be included in the recorded area and moving the window will
result in it moving outside of the recorded area.

Related Links:
--------------

* https://trac.ffmpeg.org/wiki/StreamingGuide
* https://wiki.archlinux.org/index.php/Streaming_to_twitch.tv
