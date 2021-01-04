# MIDI Controller for KSP - Part 1

> Inspired by fasterthenli.me

I recently bought myself a MIDI controller (M-Audio Axiom Air Mini 32). I want
to learn making music, and it seemed like a good beginner device. 'Tho,
honestly, I also bought it thinking about making it into KSP controller.
Kerbal Space Program (KSP) is a game about going to space, sometimes even back.
Game by default allows to use keyboard and mouse/gamepad to control your
rockets/space-planes. But some players, wanting more immersive controls, made
special custom controllers that look more like authentic rocket-launching gear.
My MIDI controller didn't look that much different from them, plus it had
enough knobs and buttons to be functional in game. KSP by itself doesn't detect
MIDI inputs (unsurprisingly, as most games don't), so I decided to implemented
my own input mapper in Rust.

> Note: reddit.com/r/KerbalControllers is main community hub of these custom
controllers

## MIDI

First, I need to get inputs from the MIDI device. Somebody posted recently
on /r/rust about making pong for a MIDI controller. Being somewhat similar to
what I'm looking for, I checked its dependencies and I found about
the `midir` crate. 'Perfect', I thought and started implementing simple demo
to just dump input values to console. The crate works There I found first problem,
controllers pads and piano keys had the same channel number, so program
couldn't distinguish them. But one of the programs that come with
the controller, Ableton, had no problem with that. More precisely, it displays
piano key as channel 1 and pads as channel 16. Something funky was going on.

> Note: There seems to be some kind of hang when I Ctrl+C the program, but
using `ctrlc` crate, a `static` `AtomicBool` and checking condition as part
of the main loop resolves the issue.

Device's user guide nor website had any direct information about how that works
but it mentions something about HyperCtrl. On device, there is mode switching
button with three leds. By default, they're turned off, but when opening
Ableton, one of the leds lights up. So the device had some undocumented way to
switch the way inputs are sent. After snooping with Wireshark, I found two
commands to turn on and off the "HyperCtrl" mode (in hex):
`f0 00 01 05 20 7f 20 3c f7` and `f0 00 01 05 20 7f 20 00 f7`, respectively.

> Note: After updating Wireshark, USB capturing stopped working. In short, to
make it work I had to 1. uninstall wireshark and usbncap, 2. restart
computer, 3. Install usbncap using wireshark installer (not separately, that
was a mistake) 4. restart again for good measure

> Note: These two MIDI messages can be parsed as such:
> - `f0` - System Exclusive (SysEx), special "escape" command for vendor
extensions
> - `00 01 05` - vendor signature, Wireshark decodes it as "M-Audio"
> - `20 7f 20 3c`/`20 7f 20 00` - the undocumented, secret-sauce sequence that
enables "HyperCtrl" mode
> - `f7` - end of sysex marker

We have working midi input, time to implement other side, sending inputs to
the game. At first, I thought about creating virtual xbox 360 controller and
send inputs this way. Even started testing with vigem for that, but this
approach has one problem; xbox 360 gamepad doesn't have enough buttons to be
mapped from midi controller. Thankfully, KSP community had an answer for that.

These custom controllers I mentioned before had similar problems. Players
wanted to send *and receive* precise informations to and from game, and gamepad
emulation wasn't going to cut it. Not only was it limiting input options, it
had not way of outputting information like position, stored resources, etc.
So community developed mods to do just that. Previously mentioned subreddit
[`/r/KerbalControllers`](reddit.com/r/KerbalControllers) has a pinned thread
with list of such mods. "Bingo", I said to myself, "with so many options, it's
gonna be a cakewalk".

## Serial

Having read some comments, I decided to start with `simpit` mod. It provided a
serial interface and a arduino library for easy usage. But I'm not using
an arduino, so I had to communicate with mod using serial ports. Appropriately
name `serialport` seemed like an obvious choice. After implementing helpers
for serializing the protocol, I tried simple handshake and got... missing port.
No, blocked port, as message was "Access is denied".

> Note: Simpit repository has a wiki page explaining how the protocol works.
But it seemed to be very limited in what can do. Checking arduino library's
documentation and code showed much more capabilities. Wiki page just hasn't
been updated in while.

So either Windows doesn't like non-authorized apps using serial ports, or
something else was blocking access to it. After spending time trying to find
possible solution for former, I realised it was the latter, and why. Serial
ports are used by computer to communicate with external devices. Mod was
connecting to it, and my program, not being an external device, couldn't
connect from the other side. Tried googling few variations on "virtual serial
port on windows" I finally found com0com, an null-modem emulator, precisely
what I needed. Installed, configured, setting up mod and program on correct
port, I've got... still nothing. Disappointed, I decided to switch to krpc.

## Protobuf

So I need to generate rust code from `krpc.proto`, either find ready-to-use
protobufs-over-TCP implementation, or simple serde-like generator and do the
network stuff myself. That excludes `tonic`, which works over HTTP/2. I started
with `prost`, but got into major problem, generated structs can't
written or read. Excuse me, they don't work with `std::io::{Write, Read}`.
Instead, they provide way to encode to and decode from `bytes::{Buf, BufMut}`.
They differ from `std::io` traits in how they handle passing communication
errors, they simply don't. This interface would require me to write a lot of
boilerplate to correctly read enough bytes, pass them to encode/decode, and
manually manage buffers, which is too error-prone for my taste, so I dropped
`prost`.

> Note: For a second I thought I found a wrapper that would help here, but I
was wrong. It worked the other way then I wanted. In retrospect, it makes sense
the wrapper that would work in my case didn't exist. It couldn't, because
in this case you can't turn infallible interface into fallible one.

I wanted to avoid installing protobuf compiler or using c bindings, so I
switched to `protobuf-codegen-pure` and `protobu`. There were a few papercuts
with this one:

1. For it to find my `.proto` file I had to add `./` before its name, in
addition to setting `include` to `.`.

2. From documentation, it expects to generate files directly to `src`, which
is *a bad idea*. Setting output folder to `OUT_DIR` and `include!`ing
in `lib.rs` results in lint errors. Not by tripping the lints, but by disabling
them. In config I had to enable `gen_mod_rs` to solve it.

3. This leads to rust-analyzer throwing errors on `include!` and `pub use`,
even though rustc has no problems. So no code completion for me.

4. According to krpc's documentation, messages are sent and received by
prepending them with length of a packet as varint. Generated structs had
a `write_length_delimited_to_writer` so everything's fine here. But the
`protobuf::parse_length_delimited_from_reader` function, the mirror to
the write one, is marked as deprecated, with no substitutes. What gives?
After reading some more about it, it seems that protobuf's most popular
method of packetizing is "un-official". So I just slapped
`#![allow(deprecated)]` on top of `main.rs` and accepted that my code is
already "obsolete".

This will be our break point. I hope this post doesn't come as too whiny/harsh.
In general, API design of rust crates have such a high level of polish, that
even minimal churn feels like a sandpaper. But that might be just me. Have a
good day and thanks for reading.

## Extra: CKAN

Lets go back to mods. There are two main ways to install them, manually or
using a mod manager, in this case CKAN. I tried it once some time ago, but it
errored out when installing mods. I've already manually installed most of the
mods I wanted, so I just skipped it. Recently, I used it again and had the same
issue, this time I reported it to developers. One of them quickly responded and
we started debugging. When using the cli, install worked. Then when I tried
installing from gui again, with logs turned on, it worked correctly this time.
Even with logs off, it still worked. An interesting case of heisenbug.

One issue when installing mods from CKAN is that mods are often not updated to
new versions of base game. They sometime still work with new releases, but
because their metadata isn't corrected, I've had to resort to manually marking
older versions as compatible. Most evident example is krpc, still claiming
compatibility with KSP 1.5 (newest being 1.11).

## Extra: MIDI Reference

NOn - NoteOn, CC - CommandChange; NoteOff isn't used

Midi signals in "HyperCtrl" mode:

- piano: port 1; channel 1; NOn;
    pitch 0-127; velocity 1-127 on press, 0 on de-press

- pads: port 2; channel 16 on bank 1, 15 on bank 2; NOn;
    pitch 81-88; velocity like piano keys

- knobs: port 2; channel 16; CC;
    pitch 33-40; vel. marks rotation, 0 is most left

- buttons:
    - sustain - port 1; channel 1; CC; pitch 64; vel. 127 press, 0 de-press
    - mod - port 1; channel 1; CC; pitch 1;
        vel. 0-127-0, ramps up on hold, down on release
    - dirs and playback - port 2; channel 16; CC; vel. 127 press, 0 de-press;
        pitch:
        - 98 - center
        - 99 - left
        - 100 - right
        - 101 - up
        - 102 - right
        - 116 - stop
        - 117 - play
        - 118 - record
