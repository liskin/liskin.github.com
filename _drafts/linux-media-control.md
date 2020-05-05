---
layout: default
title: Linux, media keys and multiple players (mpd, chromium, mpv, vlc, …)
comments_issue: 1

---

This post explains how to get [media keys][] (play, pause, …) on [keyboards][]
and Bluetooth headphones work with a bare [X window manager][] (as opposed to
a full [desktop environment][]) and how to make them control multiple media
players including the web browser (YouTube, [bandcamp][], [myNoise][], etc.)
which is something that even majority [operating systems](#windows-10) and
[desktop environments](#gnome-popular-linux-desktop-environment) don't quite
get right out of the box.

[media keys]: https://en.wikipedia.org/wiki/Computer_keyboard#Miscellaneous
[keyboards]: https://en.wikipedia.org/wiki/ThinkPad#/media/File:Lenovo-ThinkPad-Keyboard.JPG
[X window manager]: https://en.wikipedia.org/wiki/X_window_manager
[desktop environment]: https://en.wikipedia.org/wiki/Desktop_environment#Desktop_environments_for_the_X_Window_System
[bandcamp]: https://bandcamp.com/
[myNoise]: https://mynoise.net/
[mpd]: https://www.musicpd.org/

<figure markdown="block">
[![media-keys](https://user-images.githubusercontent.com/300342/81182729-8237d500-8fae-11ea-89eb-81ca7b3598fe.jpg)](https://user-images.githubusercontent.com/300342/81182740-8663f280-8fae-11ea-9b0d-db91eb6febaf.jpg)
<figcaption>ThinkPad 25 media keys</figcaption>
</figure>

{% include toc.md %}

### Goal

My use cases:

* listening to music/myNoise while working (near computer) or reading (away
  from computer)
* listening to podcasts while cooking/cleaning (away from computer, screen
  locked, wet hands)
* discovering new music on YouTube, bandcamp, soundcloud, …
* listening off-line (that why I use [mpd][] and buy music on [bandcamp][])

I'd love to use the same play/pause key/button in all of these scenarios, and
this key/button should control the appropriate application (pause the one
that's playing, play the last paused one, …). It's annoying, bad UX otherwise.

I don't want to think how to pause music when someone needs me in the
meatspace. Walking back from the kitchen to pause a podcast is just plain
silly. Playing YouTube, bandcamp, soundcloud etc. [using mpd is
possible][youtube-dl-mpd] and it's what I used to do back when my play/pause
buttons were hardwired to mpd, but it's a hack and likely against the ToS.

[youtube-dl-mpd]: https://github.com/liskin/dotfiles/blob/15c2cd83ce7297c38830053a9fd2be2f3678f4b0/bin/youtube-dl-mpd

### Solution

There is a standard [D-Bus][dbus] interface for controlling media players on a
modern Linux desktop: [MPRIS][]. It seems to be supported by both
[Chromium][chrome-mpris] and [Firefox][firefox-mpris] these days, and there's
a command line tool [playerctl][] as well, so we just need to write a few
scripts to wire it all together and everything should just work.

[dbus]: https://www.freedesktop.org/wiki/Software/dbus/
[MPRIS]: https://www.freedesktop.org/wiki/Specifications/mpris-spec/
[chrome-mpris]: https://github.com/chromium/chromium/tree/0944c7716afc8b3d8fe2236db79866d4c8a57b6f/components/system_media_controls/linux
[firefox-mpris]: https://github.com/mozilla/gecko-dev/blob/5470b66539234e52e76bc2176d9bec12325fc555/widget/gtk/MPRISServiceHandler.cpp

After some hacking, my setup looks like this (all the icons and some of the
arrows are clickable):

<figure markdown="block">
{% include_relative _svg/linux-media-control.svg %}
<figcaption>diagram of my setup</figcaption>
</figure>

The window manager ([xmonad][]) and screen locker
([xsecurelock][][^xscreensaver]) bind all the keys (and thus also headphone
buttons via [uinput][] and [bluetoothd][bluetoothd-uinput]) and call the
[liskin-media][] script:

[uinput]: https://www.kernel.org/doc/html/v5.4/input/uinput.html
[bluetoothd-uinput]: https://github.com/RadiusNetworks/bluez/blob/e11bfba10cc15cf74f8a657fad018aece4a5bde9/profiles/audio/avctp.c
[xmonad]: https://xmonad.org/
[xsecurelock]: https://github.com/google/xsecurelock
[liskin-media]: https://github.com/liskin/dotfiles/blob/15c2cd83ce7297c38830053a9fd2be2f3678f4b0/bin/liskin-media

[^xscreensaver]:
    I used to use [xscreensaver][], but it doesn't support binding (media)
    keys and [I don't trust it any more][xscreensaver-security].

[xscreensaver]: https://www.jwz.org/xscreensaver/
[xscreensaver-security]: https://news.ycombinator.com/item?id=21224179

<figure markdown="block">
<figcaption markdown="span">[~/.xmonad/xmonad.hs][xmonad.hs-keys]</figcaption>
```haskell
, ((0, xF86XK_AudioPlay ), spawn "liskin-media play")
, ((0, xF86XK_AudioPause), spawn "liskin-media play")
```
</figure>

<figure markdown="block">
<figcaption markdown="span">[~/bin/liskin-xsecurelock][liskin-xsecurelock-keys]</figcaption>
```bash
export XSECURELOCK_KEY_XF86AudioPlay_COMMAND="liskin-media play"
export XSECURELOCK_KEY_XF86AudioPause_COMMAND="liskin-media play"
```
</figure>

[xmonad.hs-keys]: https://github.com/liskin/dotfiles/blob/15c2cd83ce7297c38830053a9fd2be2f3678f4b0/.xmonad/xmonad.hs#L73-L81
[liskin-xsecurelock-keys]: https://github.com/liskin/dotfiles/blob/15c2cd83ce7297c38830053a9fd2be2f3678f4b0/bin/liskin-xsecurelock#L11-L20

A [background service][liskin-media-service] uses [playerctl][] to keep track
of the last active player:

<figure markdown="block">
<figcaption markdown="span">[~/bin/liskin-media][liskin-media-daemon]</figcaption>
{% raw %}
```bash
playerctl --all-players --follow --format '{{playerName}} {{status}}' status \
| while read -r player status; do
    if [[ $status == @(Paused|Playing) ]]; then
        printf "%s\n" "$player" >"${XDG_RUNTIME_DIR}/liskin-media-last"
    fi
done
```
{% endraw %}
</figure>

[playerctl]: https://github.com/altdesktop/playerctl
[liskin-media-service]: https://github.com/liskin/dotfiles/blob/15c2cd83ce7297c38830053a9fd2be2f3678f4b0/.config/systemd/user/liskin-media-mpris-daemon.service
[liskin-media-daemon]: https://github.com/liskin/dotfiles/blob/15c2cd83ce7297c38830053a9fd2be2f3678f4b0/bin/liskin-media#L46-L54

The [liskin-media][] script then selects the appropriate media player (the first
one that's playing; the one that's paused, if there's only one; or the last one
to play/pause) and uses [playerctl][] to send commands to it:

<figure markdown="block">
<figcaption markdown="span">[~/bin/liskin-media][liskin-media-commands]</figcaption>
```bash
function get-mpris-smart {
    get-mpris-playing || get-mpris-one-playing-or-paused || get-mpris-last
}

function action-play { p=$(get-mpris-smart); playerctl -p "$p" play-pause; }
function action-stop { playerctl -a stop; }
function action-next { p=$(get-mpris-playing); playerctl -p "$p" next; }
function action-prev { p=$(get-mpris-playing); playerctl -p "$p" previous; }
```
</figure>

[liskin-media-commands]: https://github.com/liskin/dotfiles/blob/15c2cd83ce7297c38830053a9fd2be2f3678f4b0/bin/liskin-media#L56-L100

<i>
Note that similar logic is also implemented by [mpris2controller][], which I
unfortunately haven't found until I started writing this post.
</i>

[mpris2controller]: https://github.com/icasdri/mpris2controller

The final component is the media players themselves.

[Chrome][]/[Chromium][] 81 seem to work out of the box, including metadata
(artist, album, track) in websites that use the [Media Session API][].
Somewhat surprisingly, play/pause works for any HTML `<audio>`/`<video>`, so
websites that don't use Media Session (bandcamp, …) can be controlled too. It
seems this wasn't always the case as there are several webextensions that seem
to solve this now non-existent problem: [Media Session Master][], [Web Media
Controller][], …[^webext]

[^webext]:
    And there are more, presumably from back when there was no [MPRIS][]
    support in the browser whatsoever:

    * <https://github.com/otommod/browser-mpris2>
    * <https://github.com/Aaahh/browser-mpris2-firefox>
    * <https://github.com/KDE/plasma-browser-integration>

[Firefox][] 75 works after enabling `media.hardwaremediakeys.enabled` in
`about:config`, but Media Session support is still experimental (enabling it
breaks YouTube entirely) so metadata isn't available. Also, not all websites
can be controlled: YouTube and bandcamp works, soundcloud and plain HTML5
`<audio>` example don't. Haven't investigated it further as I don't use
Firefox for media playback. (Also, it keeps resetting its volume to a weird
level.)

[myNoise][] can't be controlled by media keys in either browser as it uses
plain [Web Audio API][], so I've made [a userscript][mynoise-chrome] as a
workaround (chromium-only) and will try to help get it fixed upstream.

[mpd][] 0.21.22 itself does not support [MPRIS][], but the frontend I use —
[Cantata][] — does.

Likewise, [mpv][] 0.32.0 needs a plugin, [mpv-mpris][] works well.

[vlc][] 3.0 supports [MPRIS][] out of the box. Reportedly, [so does
Spotify](https://wiki.archlinux.org/index.php/Spotify#MPRIS). (I don't use
either.)

[Firefox]: https://firefox.com/
[Chrome]: https://www.google.com/chrome/
[Chromium]: https://www.chromium.org/Home
[Web Media Controller]: https://github.com/f1u77y/web-media-controller
[Media Session Master]: https://github.com/Snazzah/MediaSessionMaster
[Media Session API]: https://www.w3.org/TR/mediasession/
[Cantata]: https://github.com/CDrummond/cantata
[mpv]: https://github.com/mpv-player/mpv
[mpv-mpris]: https://github.com/hoyon/mpv-mpris
[vlc]: https://www.videolan.org/vlc/index.html
[Web Audio API]: https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API
[mynoise-chrome]: https://github.com/liskin/dotfiles/blob/7c89ed287af7f73411ab0dbb36cf948957a17d71/src-webextensions/myNoise-chrome-improvements.user.js

<figure markdown="block" class="video">
<div class="iframe iframe-16x9">
<iframe src="https://www.youtube-nocookie.com/embed/VYQOnKIvGgA" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>
<figcaption markdown="span">demonstration of my setup</figcaption>
</figure>

### State of the art

Out of curiosity, I wanted to know if this is a solved problem if one uses a
less weird operating system or desktop environment. Turns out, not really… :-)

#### [GNOME][] (popular Linux desktop environment)

Media keys (and presumably also headphone buttons, not tested) appear to work
out of the box, including when the desktop is locked. Unfortunately it only
works reliably (predictably) when there's just one media player application.
When there's more than one (e.g. YouTube in Chromium and music in
[Rhythmbox][]), only the one that started first (application launch, not
necessarily start of playing) is controlled, regardless of whether this one
player is stopped/playing/paused, or whether it has any playable media at all.

In practice, this means that a browser that had once visited YouTube blocks
other apps from being controlled by media keys. This is further complicated by
the fact that [gnome-settings-daemon][] has two different APIs for media keys:
[MPRIS][] and [GSD media keys API][] and prefers MPRIS players. Therefore,
Chromium are Rhythmbox are preferred to [Totem][] (GNOME's movie player) even
when launched later, which means that a user needs to understand all these
complicated bits to have any hope of knowing what player will act upon a
play/pause button press. Oh and Totem does support MPRIS in fact, it's a
plugin that may be manually enabled in its preferences.

It is _a bit_ of a mess.

<i>
(I tested this on a [Fedora][] 32 live DVD with [GNOME][] 3.36. Recordings of
some of those experiments: <https://youtu.be/1fN6NMDBFNI>,
<https://youtu.be/FCStseDBwC4>)
</i>

[GNOME]: https://www.gnome.org/gnome-3/
[Rhythmbox]: https://wiki.gnome.org/Apps/Rhythmbox
[Totem]: https://wiki.gnome.org/Apps/Videos
[GSD media keys API]: https://gitlab.gnome.org/GNOME/gnome-settings-daemon/-/blob/28ce4225535329dee6a9aff8c44bd1671ce9d2de/plugins/media-keys/README.media-keys-API
[gnome-settings-daemon]: https://gitlab.gnome.org/GNOME/gnome-settings-daemon/-/tree/28ce4225535329dee6a9aff8c44bd1671ce9d2de/plugins/media-keys
[Fedora]: https://getfedora.org/

<!-- TODO: note about KDE and <https://github.com/KDE/plasma-browser-integration> -->

#### Windows 10

Similarly to GNOME, media keys appear to work fine at first glance, but when
multiple/specific apps are involved, minor problems appear.

Windows 10 have their equivalent of [MPRIS][] called [System Media Transport
Controls][SMTC] and this is supported by [Chromium][chrome-mpris], and
therefore by both [Chrome][] and the [new Chromium-based Edge][Edge-chromium].
It's not supported by the (deprecated) [Windows Media Player][WMP], but that's
probably fine as the modern replacement [Movies/Films & TV][MoviesTV] supports
it very well.

As opposed to GNOME, an application not supporting the [SMTC API][SMTC] does
not mean it doesn't react to media keys. Windows Media Player [does, quite
well actually](https://youtu.be/9DN2tcZGsHU) (even on lock screen), but it
doesn't grab the keys so when there's another app, media keys [control both of
them](https://youtu.be/aPSkMTZcy8w). On the other hand, [vlc][] only [handles
the keys when focused](https://youtu.be/FQAFurnLUVU). Finally and not
surprisingly, [old Edge][Edge-old] and [Internet Explorer][IE] don't [handle
them at all](https://youtu.be/uKRqZ3p76Gw).

Handling of multiple apps that all support SMTC is
[good](https://youtu.be/1-m0kECqt38), but there's a bug that would make this
completely unusable for my podcast use case: it's not possible to continue
playing from the lock screen if it'd been paused for more than a few seconds.
This bug does not affect [Movies/Films & TV][MoviesTV], though.

Were it not for this issue, I'd say it's perfectly usable, as old players and
browsers can easily be avoided and I wouldn't mind not being able to use vlc
for background playback.

[SMTC]: https://docs.microsoft.com/en-us/windows/uwp/audio-video-camera/integrate-with-systemmediatransportcontrols
[IE]: https://en.wikipedia.org/wiki/Internet_Explorer
[Edge-old]: https://en.wikipedia.org/wiki/Microsoft_Edge#Spartan_(2014%E2%80%932019)
[Edge-chromium]: https://blogs.windows.com/windowsexperience/2020/01/15/new-year-new-browser-the-new-microsoft-edge-is-out-of-preview-and-now-available-for-download/
[MoviesTV]: https://en.wikipedia.org/wiki/Microsoft_Movies_%26_TV
[WMP]: https://en.wikipedia.org/wiki/Windows_Media_Player

<i>
(I tested this on a clean Windows 10 Pro version 1909 with no vendor-specific
bloatware. Recordings of the experiments are linked from the preceding
paragraphs, and for completeness also listed here:
<https://youtu.be/9DN2tcZGsHU>, <https://youtu.be/1-m0kECqt38>,
<https://youtu.be/aPSkMTZcy8w>, <https://youtu.be/FQAFurnLUVU>,
<https://youtu.be/uKRqZ3p76Gw>.)
</i>

#### macOS Catalina

I expected this to work almost flawlessly as Apple is known for their focus on
UX, but it seems worse than Windows 10, unfortunately. Worse than Windows 10
with the _optional_ upgrade to the [new Chromium-based Edge][Edge-chromium],
that is.

My experience as a user of macOS is very limited, and as a developer
non-existent, so I won't attempt to go into technical details and I'll only
describe observed behaviour.

Media keys work in every app I tried (but I haven't tried any that don't come
pre-installed, like [vlc][]), and they work on lock screen as well, regardless
of how long it's been paused/locked. Unfortunately they only start controlling
the app after I've interacted with the play/pause button at least once, so
when I open a video and press the play/pause key on the keyboard, instead of
pausing the video, the Music app opens.

When multiple players are open, the last one that I interacted with is
controlled, as it should be. When one of them is closed, however, the control
isn't transferred to the other one, unless the application is terminated
entirely or I manually interact with the other one. Strangely, it works well
when a music-playing tab in Safari is closed.

<i>
(I tested this on a clean [macOS Catalina][catalina] 10.15.4 with no
additional software installed. Recordings of some of those experiments:
<https://youtu.be/VN7-eZsIpOE>, <https://youtu.be/oIo21HRPfhM>)
</i>

[catalina]: https://en.wikipedia.org/wiki/MacOS_Catalina

#### Android

> TODO: Chrome
>
> TODO: browser & media player
>
> TODO: lockscreen
>
> TODO: bluetooth

---

*[UX]: user experience
*[ToS]: Terms of Service
*[meatspace]: Deriving from cyberpunk novels, meatspace is the world outside of the 'net-- that is to say, the real world, where you do things with your body rather than with your keyboard.