+++
title = "The Root Causes of (Almost) All Your Linux Audio Issues"
description = "Wine buffer problems, USB mic clock drift, and onboard audio noise — explained with primary sources and actual fixes"
date = 2026-03-01
weight = 1
+++

# The Root Causes of (Almost) All Your Linux Audio Issues

Linux audio has real problems. The most common ones are rooted in hardware realities that every OS deals with — audio is the last analog component on the board, and that comes with physics constraints. Windows handles them with vendor-specific or OS-level compensations. Linux mostly lacks that coverage because vendors don't properly test on or support Linux, and Linux tends to not do OS-level compensations as a rule. That said, here are three specific, provable causes behind most everyday audio complaints, how to diagnose which one is yours, and what to do about it.

This doesn't cover Bluetooth, professional DAW latency tuning, or JACK routing — just the everyday "why does my audio crackle" issues that drive people to forums and away from Linux. Each section covers what the problem is, proof it exists from primary sources, how to check if it's yours, and what to do about it. Everything is verifiable — don't take my word for it.

If you're gaming through Proton and have crackle, start with Section 1. If your USB mic has issues, jump to Section 2. If your onboard audio has noise or crackle on native Linux apps, read Sections 3 and 4 (in a follow-up post).

---

## Section 1: Wine/Proton Audio Crackle — The Buffer Problem

### What You Experience

You're playing a game through Proton or Wine. Audio crackles, pops, stutters. Maybe it starts fine and degrades over time. Maybe a lag spike causes permanent audio corruption until you restart. Native Linux applications — your browser, your music player, system sounds — work perfectly. The problem is exclusively in Wine/Proton.

You google it. Every forum thread says the same thing:

```
PULSE_LATENCY_MSEC=60 %command%
```

Maybe it helps. Maybe it doesn't. Nobody explains why, and nobody mentions that 60ms of audio latency is *terrible* for gaming.

### What's Actually Happening

Windows games request audio buffers from the operating system. Many games request very small buffers — sometimes as low as 3ms — because on Windows, it doesn't matter what you ask for.

Here's why: Microsoft's own documentation on the Windows audio engine states that prior to Windows 10, **the audio engine buffer was always ~10ms in shared mode**. Applications could request whatever they wanted. Windows gave them 10ms regardless. From Microsoft's [Low Latency Audio documentation](https://learn.microsoft.com/en-us/windows-hardware/drivers/audio/low-latency-audio):

> *"Before Windows 10, the latency of the audio engine was equal to ~12ms for applications that use floating point data and ~6ms for applications that use integer data"*

> *"Data transfers don't have to always use 10ms buffers, **as they did in previous Windows versions**"*

That second quote is the key. Before Windows 10, the 10ms buffer was *mandatory*. The game asks for 3ms? It gets 10ms. The game asks for 1ms? It gets 10ms. The game never knows the difference because Windows silently substituted a sane value.

Wine and Proton don't do that. They're compatibility layers designed to faithfully implement the Windows API. When a game says "give me a 3ms buffer," Wine actually tries to deliver a 3ms buffer through PulseAudio or PipeWire. And your audio hardware probably can't actually deliver clean audio at 3ms — so you get buffer underruns, which you hear as crackle and pops.

This is Wine being *too accurate*, not Linux being broken.

### Proving It

This isn't speculation. You can watch it happen in real time.

**Step 1: See the underruns live**

Open a terminal and run:

```bash
pw-top
```

This shows you every audio stream on your system, its buffer size, and — critically — the `ERR` column, which counts buffer underruns (xruns). Now launch a game through Proton. Watch the ERR column. If it's climbing, you're watching your crackle happen.

Note the `QUANT` column — that's the buffer quantum (size) the stream is using. If it's very small (64, 128 samples), that's your game requesting a tiny buffer and PipeWire trying to honor it.

**Step 2: Confirm it's the buffer**

Set a higher latency for just that game:

```bash
PULSE_LATENCY_MSEC=40 %command%
```

Relaunch the game. Watch `pw-top` again. The ERR column should stay at zero or climb much more slowly. The crackle should be gone or dramatically reduced.

You've just proven the cause: the game requested a buffer too small for your hardware to service reliably, and Wine faithfully passed that request through instead of silently overriding it like Windows does.

**Step 3: Corroborating evidence**

This pattern is confirmed across the ecosystem:

- A [Fedora discussion on Wine/Proton audio stutter](https://discussion.fedoraproject.org/t/wine-proton-with-pipewire-sound-stutters-pipewire/126047) explicitly identifies the cause as "very low quantum value" — the application requesting tiny buffer sizes.
- An [Ubuntu bug report on PipeWire + Wine](https://bugs.launchpad.net/bugs/2086206) documents the problem reproducing specifically with Wine/Proton and not with native applications on the same hardware.
- A developer working on Wine compatibility for the game engine wrapper DxWnd [discovered](https://sourceforge.net/p/dxwnd/discussion/general/thread/c518bfd4cd/?page=1) that Windows "seems to swallow and process sound headers making less controls" — accepting technically incorrect audio parameters that Wine rejects because Wine does stricter validation. Making buffers larger fixed the crackling under Wine.
- Hundreds of [Arch Linux](https://bbs.archlinux.org/viewtopic.php?id=276168), [Linux Mint](https://forums.linuxmint.com/viewtopic.php?t=451300), and [GitHub](https://github.com/ValveSoftware/Proton/issues/1147) threads all converge on the same pattern: native audio fine, Wine/Proton crackle, buffer increase fixes it.

### The Fix (And Why the Common Advice is Harmful)

**Do not blindly set `PULSE_LATENCY_MSEC=60`.**

60 milliseconds of audio latency is *huge*. That's perceptible delay between a visual event and its sound. In a game where audio cues matter — footsteps, gunshots, impact feedback — you're adding almost a tenth of a second of desync. You've traded one problem (crackle) for another (lag).

The correct approach is to find the *minimum stable latency for your hardware*, not cargo-cult a value from a forum post.

**Step 1: Find your hardware's reported period**

```bash
pw-top
```

Look at the `QUANT` and `RATE` columns for your output device (not the game stream — the actual sink). This tells you the quantum your hardware is running at. You can also check:

```bash
pactl list sinks | grep -A 5 "Latency"
```

This shows the configured and actual latency of your audio sink.

**Step 2: Calculate your minimum**

If your hardware quantum is 1024 samples at 48000 Hz:

```
1024 / 48000 * 1000 = ~21.3ms
```

That's your hardware's actual buffer period. You need your Wine latency to be at or slightly above this — not three times higher.

**Step 3: Set it per-application, never system-wide**

In Steam, set the launch option for the specific game:

```
PULSE_LATENCY_MSEC=25 %command%
```

Start with your calculated value, test with `pw-top` for underruns, and adjust up or down. Every game might need a slightly different value depending on how aggressively it requests small buffers.

**Never set this system-wide.** It affects *all* PulseAudio/PipeWire clients. People have reported that setting it globally actually *causes* underruns in other applications that were working fine, because you've forced a latency that's wrong for their buffer expectations.

If you're using PipeWire, you can also tune the quantum directly in your PipeWire config rather than using the PulseAudio environment variable:

```bash
# In ~/.config/pipewire/pipewire.conf.d/gaming.conf (or similar)
context.properties = {
    default.clock.min-quantum = 512
}
```

Again — calculate the right value for your hardware, don't guess.

**Step 4: Verify the fix**

Launch the game with your tuned latency. Open `pw-top`. Confirm:
1. The ERR column stays at zero (no underruns)
2. The audio doesn't crackle
3. Audio-visual sync feels correct (no perceptible delay on in-game sounds)

If you're still getting underruns, bump the value up slightly. If the latency feels too high, try lowering it. You're finding *your* system's sweet spot, not applying someone else's.

### The Same Problem in Discord, Zoom, and Electron Apps

This isn't just a Wine/Proton issue. Native Linux applications can trigger the same buffer underrun pattern — Discord is the most common offender.

Discord uses WebRTC for voice chat, and WebRTC has aggressive timing requirements. From its [audio architecture](https://webrtchacks.com/how-webrtcs-neteq-jitter-buffer-provides-smooth-audio/):

> *"GetAudio returns exactly 10ms of audio samples. It MUST be called by the playout thread exactly 100 times per second."*

That's a 10ms callback interval — tighter than most games request. PulseAudio's own documentation notes it "usually doesn't handle very low latencies well" and recommends at least 20ms buffers.

Discord also uses Opus, which only operates at 48kHz. If your system runs at 44.1kHz or 192kHz, resampling happens on every audio callback. Resampling under tight timing constraints compounds the underrun risk.

Users on [Arch Linux Forums](https://bbs.archlinux.org/viewtopic.php?id=307744) have observed this directly in `pw-top`:

> *"In pw-top, ALSA audio_output.pci shows ERR while crackling is present. Other applications such as VLC or game audio do not display such errors."*

The [Arch Wiki Discord page](https://wiki.archlinux.org/title/Discord) explicitly acknowledges the pattern:

> *"If you experience crackling sounds when in voice chat, try the steps outlined in PulseAudio/Troubleshooting#Troubleshooting buffer underruns"*

And there's a Chromium bug ([#169075](https://issues.chromium.org/40959351)) titled "Total audio corruption with WebRTC and PulseAudio" documenting this at the browser engine level.

**The fix is the same principle:** increase buffer headroom so the audio stack can meet the timing demands. For Discord specifically:

```bash
# Launch Discord with increased latency
PULSE_LATENCY_MSEC=60 discord
```

Or try switching Discord's audio subsystem to "Legacy" in Voice & Video settings, which uses less aggressive buffering.

If your system isn't running at 48kHz, consider matching it:

```bash
# In ~/.config/pipewire/pipewire.conf.d/discord.conf
context.properties = {
    default.clock.rate = 48000
}
```

This eliminates resampling overhead during Discord's tight callback loop.

### PipeWire's Quantum Clamping — A Proper Fix

If you're on PipeWire (most modern distros), you have a better option than environment variables: **quantum clamping**. This lets you set a floor on buffer sizes so apps can't request dangerously small values.

The problem, explained by a user on [Linux Mint Forums](https://forums.linuxmint.com/viewtopic.php?t=449235):

> *"The default time buffers for pipewire-pulse is 2.7 ms, which means that an application needs to be able to fill sound content every 2.7 ms, otherwise pipewire will send to the audio device whatever was previously filled and potentially some random data, hence the crackling."*

2.7ms is aggressive. When a Discord notification fires while you're watching YouTube, a new audio stream starts. If the app can't fill that tiny buffer in time, you get garbage data — crackling.

**The fix: raise the quantum floor.**

For all PulseAudio-compatible apps:

```ini
# ~/.config/pipewire/pipewire-pulse.conf.d/min-quantum.conf
pulse.properties = {
    pulse.min.quantum = 512/48000
}
```

Or globally for all PipeWire clients:

```ini
# ~/.config/pipewire/pipewire.conf.d/min-quantum.conf
context.properties = {
    default.clock.min-quantum = 1024
}
```

For Discord specifically (without affecting other apps):

```ini
# ~/.config/pipewire/pipewire-pulse.conf.d/discord.conf
pulse.rules = [
    {
        matches = [ { application.process.binary = "Discord" } ]
        actions = {
            update-props = {
                pulse.min.quantum = 1024/48000
            }
        }
    }
]
```

Then restart PipeWire:

```bash
systemctl --user restart pipewire pipewire-pulse
```

**This works.** From [EasyEffects Issue #1567](https://github.com/wwmm/easyeffects/issues/1567), where users debugged Discord notification crackling:

> *"After using it for some time, it would seem that indeed setting a fixed quantum rate reduces a lot the number of times I get a crackle."*

The maintainer's blunt assessment of Discord:

> *"Discord's app is an endless source of problems"*

Common values that work:
- `512/48000` (~10.7ms) — often sufficient
- `1024/48000` (~21.3ms) — common sweet spot
- `2048/48000` (~42.7ms) — for stubborn cases

Start low, increase if you still get crackling. Higher values add latency, so find your minimum stable setting.

---

## Section 2: USB Microphone/Input Crackle — The Clock Problem

### What You Experience

Your USB microphone or audio interface has crackle, intermittent distortion, or periodic pops on Linux. It might work fine for a while and then develop issues. It might be worse under system load. Meanwhile, the same device on Windows works perfectly.

Unlike the Wine issue above, this affects native Linux applications too — recording in Audacity, voice in Discord, video calls, everything that uses the USB audio device for input.

### What's Actually Happening

This is a physics problem. Your USB audio device has its own clock crystal — a piece of hardware that generates a precise timing signal to control the rate at which audio samples are captured. Your computer has a separate clock. These two clocks need to stay synchronized, because if the device captures audio at a slightly different rate than the computer expects to receive it, samples pile up or go missing. You hear this as crackle.

USB audio devices communicate using *isochronous transfers* — a USB transfer mode designed for real-time data like audio and video. Within isochronous mode, there are different synchronization strategies:

**USB Audio Class 1.0 (UAC 1.0)** primarily uses *adaptive* isochronous mode. The device tries to sync to the host's clock, but it's doing this over USB — a bus with its own timing characteristics and jitter. The two clocks drift relative to each other because they're independent oscillators. Physics guarantees they'll never be perfectly synchronized.

On Windows, vendor drivers add compensation for this drift — extra buffering, clock smoothing, error correction. You never see it because it's hidden inside the proprietary driver. The device "just works" because the driver is doing constant behind-the-scenes correction.

On Linux, the ALSA USB audio driver is more standards-compliant. It implements the UAC specification faithfully. When the clocks drift, you hear it, because the driver isn't applying proprietary compensation tricks.

**USB Audio Class 2.0 (UAC 2.0)** introduced *asynchronous* isochronous mode. This is the actual fix at the protocol level. In async mode, the device is the clock master. It runs its own clock and tells the host exactly how fast to feed it data via a feedback endpoint. The host adjusts its data rate to match the device. Drift is eliminated by design — it's not two clocks trying to sync, it's one clock in charge with the other following.

**UAC 2.0 was published in 2006.** The protocol-level fix has existed for nearly two decades. It works cleanly everywhere — Windows, Linux, macOS — because the *protocol itself* solved the synchronization problem. No driver hacks needed.

### Proving It

**The Linux kernel knows about this**

The Linux USB audio driver handles UAC 1.0 and UAC 2.0 devices through different code paths precisely because of the synchronization difference. The kernel source in `sound/usb/endpoint.c` implements the different sync models, and `sound/usb/clock.c` handles the clock management that UAC 2.0's async mode relies on.

If you check the kernel's [USB Audio documentation](https://docs.kernel.org/sound/), you'll find the driver architecture reflects these two fundamentally different approaches to clock synchronization.

**Windows vendor drivers exist for a reason**

Why does every USB audio manufacturer ship a custom Windows driver when Windows has a built-in USB audio class driver (`usbaudio.sys`)? Because the generic driver doesn't compensate well enough for their specific hardware's clock behavior. The vendor driver adds device-specific clock drift correction, buffering adjustments, and error handling tuned to that particular device's oscillator characteristics.

On Linux, there's one driver for all USB audio devices. It handles the standard correctly but doesn't have per-device clock compensation. UAC 1.0 devices with poor oscillators or marginal clock behavior will have issues that their Windows vendor driver would mask.

**The WASAPI evidence**

Microsoft's own developer forums contain reports of [WASAPI exclusive mode audio becoming distorted at minimum buffer sizes](https://learn.microsoft.com/en-us/answers/questions/832677/wasapi-exclusive-mode-audio-becomes-distorted-at-l) specifically with USB audio using the standard Windows USB audio driver (`usbaudio.sys`). The reporter notes it doesn't happen with the Realtek driver (onboard audio) — only USB. The symptoms are consistent with clock drift when the standard driver's compensation isn't sufficient. Even Windows can't fully hide this problem in all cases.

### The Market Reality: Why This Still Happens in 2026

If UAC 2.0 solved this problem in 2006, why are you still buying UAC 1.0 microphones? Because Windows hides the problem well enough that manufacturers have no incentive to change.

Here's the feedback loop:

1. **Windows compensates** for clock drift with driver tricks → device "works"
2. **Device "works" on Windows** → vendor ships it, no engineering pressure to use UAC 2.0
3. **UAC 1.0 chips are cheaper** → keep designing them into new products
4. **Linux user complains** → gets told "Linux audio is broken"
5. **Vendor never hears about it** → cycle continues for another product generation

The result: most consumer USB microphones sold today — including current 2024-2026 models from major brands — still use UAC 1.0.

| Microphone | UAC Version | Evidence |
|------------|-------------|----------|
| **HyperX SoloCast** | 1.0 | Operates at USB 1.1 Full Speed (12 Mb/s) per [hardware database](https://linux-hardware.org) |
| **Fifine K669B** | 1.0 | Plug-and-play, uses generic `snd-usb-audio`, no drivers on any OS |
| **Samson Go Mic** | 1.0 | Detected as 16-bit Full Speed USB in kernel logs |
| **Audio-Technica AT2020USB+** | 1.0 | 16-bit/48kHz max, kernel support since 2.6.0, plug-and-play |
| **Rode NT-USB Mini** | 1.0 | "Class compliant" 24-bit/48kHz, explicitly driverless on all platforms |
| **Blue Yeti** | 1.0 | Class-compliant, no drivers needed |
| **Blue Yeti Pro** | **2.0** | Explicitly UAC 2.0 — and notably **requires driver installation on Windows** |

The Blue Yeti vs. Yeti Pro comparison is telling. The standard Yeti is UAC 1.0 and "just works" everywhere. The Yeti Pro is UAC 2.0 — and because UAC 2.0 wasn't natively supported on older Windows versions, it ships with a driver installer. The need for a Windows driver is actually a sign of the *better* protocol.

The pattern: if a USB microphone advertises "plug and play, no drivers needed on Windows/Mac/Linux," it's almost certainly UAC 1.0. The driverless convenience is a symptom of using the older, problematic protocol.

Meanwhile, USB DACs (output devices) in the $20-50 range have been UAC 2.0 for years. Why? Because audiophiles demanded sample rates above 96kHz, which UAC 1.0 physically cannot support due to USB 1.1 Full Speed bandwidth limits. Market pressure forced the upgrade for output. No equivalent pressure exists for microphone input, where 48kHz is sufficient for nearly everyone.

### How to Check If This Is Your Issue

**Step 1: Identify your device's UAC version**

Plug in your USB audio device and run:

```bash
lsusb -v 2>/dev/null | grep -A 5 -i "audio"
```

Look for `bInterfaceProtocol`. For a more readable view:

```bash
lsusb -v 2>/dev/null | grep -B 3 -A 10 "Audio Control"
```

What you're looking for:
- **`bcdADC` of `0100` or `bInterfaceProtocol` of `00`** → UAC 1.0 (potentially vulnerable to clock drift issues)
- **`bcdADC` of `0200` or `bInterfaceProtocol` of `20`** → UAC 2.0 (has async mode, likely fine)

You can also check `dmesg` when you plug the device in:

```bash
dmesg | tail -20
```

The kernel will log which driver path it's taking and often indicates the device class version.

**Step 2: Check for clock-related errors**

While using the device, watch the kernel log:

```bash
dmesg -w | grep -i "usb\|audio\|xrun\|clock"
```

Clock drift issues often generate periodic warnings in the kernel log about clock adjustments, sync errors, or buffer overflows/underflows.

**Step 3: Test under load**

Clock drift issues often worsen under CPU or USB bus load because the host has less time to service USB transfers precisely. If your mic sounds fine when idle but crackles during gaming or heavy work, this is a strong indicator.

### The Voice Call Problem: You Sound Fine to Yourself

Here's what makes USB mic clock drift particularly insidious for voice calls: **you can't hear your own problem.**

With microphone input issues, the distorted audio goes out to others. You hear yourself through monitoring or sidetone perfectly fine — but everyone on the Discord call, Zoom meeting, or Teams standup hears crackling, robotic distortion, or periodic breakup. They tell you "your mic sounds terrible" and you have no idea why.

This is documented across Linux forums:

A user on the [PulseAudio mailing list](https://lists.freedesktop.org/archives/pulseaudio-discuss/2017-November/029122.html) reported severe crackling specifically when joining Discord voice channels. Their debugging showed buffer underruns coinciding with the crackling, and they noted that `tsched=0` (disabling timer-based scheduling) *"eliminated initial crackling but introduced microphone static"* — a partial fix that shows the problem is timing-related.

An [Arch Linux thread about Skype](https://bbs.archlinux.org/viewtopic.php?id=164959) documented the same pattern: voice call crackling fixed with `PULSE_LATENCY_MSEC=30 skype`. Larger buffers absorb the timing jitter from clock drift — the same reason buffer increases help with any timing-sensitive audio path.

**Why voice apps expose this more than recording apps:**

When you record in Audacity, you're capturing audio at a steady rate to disk. Small timing variations get absorbed — you might not notice slight drift in a recording.

Voice call applications are different. They're doing real-time encoding (usually Opus), transmission over network, and playback on the other end. Clock drift that causes periodic buffer underruns becomes audible as crackling or robotic artifacts in the compressed stream. The codec is trying to encode audio that's arriving at an inconsistent rate, and the artifacts get baked into what others hear.

**The stack of workarounds:**

All these fixes address the same underlying clock drift at different layers:

| Layer | Fix | What it does |
|-------|-----|--------------|
| Kernel | `snd_usb_audio.implicit_fb=1` | Software PLL to track device clock |
| Kernel | `snd_usb_audio.lowlatency=0` | Disable aggressive low-latency mode |
| PulseAudio | `tsched=0` | Interrupt-driven instead of timer-based scheduling |
| Application | `PULSE_LATENCY_MSEC=30` | Larger buffer absorbs timing jitter |

Larger buffers help because they absorb timing jitter — whether from clock drift (USB mics) or aggressive buffer requests (games). The symptom is the same (underruns → crackling), even when the root cause differs.

### The Fix

This one is straightforward, and there's really only one reliable answer:

**Use a UAC 2.0 device.**

Check your device's UAC version using the steps above. If it's UAC 1.0 and you're having issues, the device relies on clock synchronization that Windows compensates for with vendor drivers but Linux handles generically. No amount of PipeWire or PulseAudio tuning will reliably fix a hardware clock synchronization problem.

UAC 2.0 devices with async isochronous mode solve the problem at the protocol level. The device controls the clock, the host follows. It works correctly on every operating system without special driver support.

When shopping for a USB audio device for Linux:

- Look for **explicit UAC 2.0 support** in the specs (sometimes listed as "USB Audio Class 2.0" or "USB 2.0 Audio Compliant")
- **USB DACs** in the $20-50 range (for output) almost universally support UAC 2.0 at this point
- For **USB microphones**, check specs more carefully — cheaper mics sometimes still use UAC 1.0
- **USB audio interfaces** (Focusrite Scarlett, etc.) generally support UAC 2.0 and work excellently on Linux

### If You Can't Replace the Device: Kernel Workarounds

If buying a new microphone isn't an option, there are kernel parameters that can help. These are workarounds, not fixes — they're compensating for hardware limitations in software. But they work for many people.

**The Evidence This Works**

These aren't theoretical. Linux users have discovered them through debugging:

An [Arch Linux user](https://bbs.archlinux.org/viewtopic.php?id=294015) with a cheap Generalplus USB microphone (vendor `0x1b3f`) reported severe "cracking, noise and distortion" with a "robotic" quality — despite the same mic working fine on Windows. After extensive debugging, they found the fix:

```bash
# /etc/modprobe.d/usb-audio.conf
options snd_usb_audio lowlatency=0 implicit_fb=1
```

Their verdict: *"sounds a thousand times better."*

A [LinuxMusicians thread](https://linuxmusicians.com/viewtopic.php?t=26699) documents users hitting "clock source 41 is not valid" errors with USB audio interfaces, resolved with the same `implicit_fb=1` parameter.

**What These Parameters Do**

`snd_usb_audio.implicit_fb=1`

This enables *implicit feedback* mode. Normally, UAC 2.0 async devices have an explicit feedback endpoint that tells the host the exact clock rate. UAC 1.0 devices don't have this. Implicit feedback makes the kernel *derive* the clock rate by counting samples on the input endpoint — essentially implementing a software PLL (phase-locked loop) to track the device's clock.

This is what Windows vendor drivers do behind the scenes. The Linux kernel can do it too, but it's not enabled by default because it's computationally expensive and not all devices need it.

`snd_usb_audio.lowlatency=0`

Disables low-latency mode for USB audio. Low-latency mode uses smaller buffers and tighter timing, which works great when clocks are synchronized but causes glitches when they're not. Disabling it gives the driver more slack to absorb clock drift.

`snd_usb_audio.quirk_flags=0x20`

A bitmask that modifies driver behavior. Bit 5 (value `0x20` in hex, or `32` in decimal) skips clock selector manipulation entirely. Useful when the device's clock reporting is broken or confuses the driver.

**How to Apply Them**

Temporary (test first):

```bash
sudo modprobe -r snd_usb_audio
sudo modprobe snd_usb_audio implicit_fb=1 lowlatency=0
```

Unplug and replug your USB audio device, then test.

Permanent (if it works):

```bash
# Create /etc/modprobe.d/usb-audio.conf
options snd_usb_audio implicit_fb=1 lowlatency=0
```

Then reboot, or reload the module.

**Which Parameter to Try First**

1. Start with `implicit_fb=1` alone — this is the most common fix for clock drift
2. If still crackling, add `lowlatency=0`
3. If you're getting "clock source is not valid" errors in `dmesg`, try `quirk_flags=0x20`

**These Are Workarounds, Not Fixes**

These parameters may break with kernel updates, may not work for all devices, and add CPU overhead. They're the Linux equivalent of what Windows vendor drivers do automatically — but done manually and generically rather than tuned per-device.

If the workarounds help, your mic works, and you're happy — great. But understand you're papering over a hardware protocol limitation. A UAC 2.0 device would just work.

---

## Section 3: Onboard Audio Noise and Crackle — The Hardware Problem

### What You Experience

Your onboard audio has noise, crackle, hiss, or distortion. Not in Wine — everywhere. Native apps, music playback, maybe even a faint whine when your system is idle. Or maybe it's more subtle: audio that's mostly fine but develops crackle under load, or pops when other hardware is active, or hiss that gets louder when you scroll a webpage or move your mouse.

You've tried every PulseAudio and PipeWire tweak you can find. Some helped temporarily. Some helped for someone else but not you. Some made it worse. Nothing sticks.

### What's Actually Happening

Your motherboard has a DAC — typically a Realtek codec chip — sitting on a PCB surrounded by some of the noisiest components in computing. VRMs switching at high frequency to power your CPU. A GPU dumping EMI across PCIe lanes. NVMe drives, USB controllers, RAM, ethernet — all generating electromagnetic interference inches from the one component on your board that handles analog signals.

The analog portion of your audio path picks up this interference. This is physics. You can't fully shield a DAC from its surroundings on a dense consumer motherboard at consumer price points. Every board has a unique noise profile determined by its PCB layout, component placement, shielding design, trace routing, and what you've plugged into it. Even two identical motherboards can have different noise characteristics depending on what GPU, NVMe, or USB devices are installed, because those change the EMI environment.

On Windows, every motherboard manufacturer ships their own customized version of the Realtek audio driver, even though they all use the same Realtek chip. Each board needs board-specific noise compensation — noise gates, filters, gain adjustments, pin configurations — tuned to that specific PCB's characteristics. Some compensation happens in BIOS/UEFI firmware. Some in the driver. Some in the manufacturer's audio software (Realtek Audio Console, Nahimic, Sonic Studio, etc.).

Realtek themselves say this directly. Their [download page](https://www.realtek.com/Download/ToDownload?type=) states:

> *"Audio drivers available for download from the Realtek website are general drivers for our audio ICs, and may not offer the customizations made by your system/motherboard manufacturer."*

Realtek's own generic driver isn't good enough. The manufacturer's version is necessary. That tells you everything.

On Linux, the kernel's `snd-hda-intel` driver has a quirk table — a list of specific vendor and board IDs mapped to specific fixups. You can see it in [`sound/pci/hda/patch_realtek.c`](https://github.com/torvalds/linux/blob/master/sound/pci/hda/patch_realtek.c). It's enormous. Hundreds of entries:

```c
SND_PCI_QUIRK(0x1028, 0x0798, "Dell Inspiron 17 7000 Gaming", ALC256_FIXUP_DELL_INSPIRON_7559_SUBWOOFER),
SND_PCI_QUIRK(0x17aa, 0x3813, "Legion 7i 15IMHG05", ALC287_FIXUP_LEGION_15IMHG05_SPEAKERS),
SND_PCI_QUIRK(0x1462, 0xcc34, "MSI Godlike X570", ALC1220_FIXUP_GB_DUAL_CODECS),
```

Each one is a board-specific fixup — pin remapping, gain adjustment, amplifier initialization sequences, codec verb overrides — submitted by someone who figured out what that board needed. Some are one-liners. Some are elaborate multi-step initialization sequences writing specific values to specific codec registers.

This quirk table is the Linux equivalent of what Windows motherboard manufacturers do in their custom drivers. The difference is coverage. The Windows driver covers every board because the manufacturer wrote it. The Linux quirk table covers boards that someone in the community reverse-engineered and submitted. If your board isn't in there, you get generic behavior on hardware that demonstrably needs per-board tuning.

### The Proof Chain

**Linux kernel source:** The quirk table in `patch_realtek.c` is primary evidence that boards objectively need individual tuning. Every entry exists because generic driver behavior was insufficient for that hardware. The [kernel HD-Audio documentation](https://docs.kernel.org/sound/hd-audio/notes.html) explicitly discusses model-specific options and the need for per-board configuration.

**Realtek's own statement:** They tell you their generic driver lacks manufacturer customizations. If generic was sufficient, this disclaimer wouldn't exist.

**Windows user experience:** Forum threads are full of people reporting that their motherboard manufacturer's audio driver sounds different from the generic Realtek or Windows drivers. People report [buzzing that only occurs with the manufacturer's Realtek driver and disappears with the generic Windows driver](https://www.tenforums.com/sound-audio/216697-realtek-driver-audio-buzzing-when-connecting-ethernet-cable-wifi-bt.html) — the manufacturer's driver is actively doing signal processing. People report BIOS updates fixing audio problems — compensation happens at the firmware level too.

**Manufacturer behavior:** ASUS ships Sonic Studio. MSI bundles Nahimic. ASRock's [support page](https://www.asrock.com/support/faq.asp?id=487) has instructions for enabling noise suppression in their Realtek audio console. Every manufacturer builds audio compensation software because the hardware needs it.

### Why This Is Harder Than the Other Two

The Wine buffer problem has a clean fix: calculate your latency, set it per-app. The USB clock problem has a clean fix: get UAC 2.0. Both are deterministic.

Onboard audio noise is different. Your noise profile is unique. It depends on your specific motherboard, installed hardware, workload, even your power supply. A fix that works for someone with the same motherboard but a different GPU might not work for you, because the GPU changes the EMI environment.

The real, durable fixes are:

1. **Get your board into the kernel quirk table.** Figure out what fixups your board needs (by studying what the Windows driver does or experimenting with HDA verb commands) and submit a patch upstream. The kernel's [HD-Audio documentation](https://www.kernel.org/doc/html/latest/sound/hd-audio/notes.html) explains how to experiment with model options and codec verbs. Tools like `hda-analyzer` and `hda-verb` exist for this.

2. **Bypass onboard audio entirely with a UAC 2.0 USB DAC.** Move the analog conversion off the noisy motherboard onto an external device with its own clean power and no EMI from surrounding components. The digital signal over USB is immune to board noise. A $20-50 USB DAC eliminates the entire problem category permanently, regardless of kernel updates, driver changes, or hardware swaps.

I get that "buy a DAC" or "reverse-engineer your board's audio codec initialization and submit a kernel patch" are not what you want to hear. So here's the next section.

---

## Section 4: Living With Onboard Audio — A Diagnostic Toolkit

Not everyone can buy a DAC today. Not everyone wants to reverse-engineer HDA codec verbs. If you're stuck with onboard audio that has issues, this section is for you.

**Everything in this section is symptomatic treatment, not a cure.** You're tuning around a hardware problem with software. It might work. It might work for a while and stop. It might break when you update your kernel, change your PipeWire config, or install new hardware. The underlying cause — your board's unique noise profile lacking proper driver compensation — hasn't changed.

But understanding what's happening turns you from someone blindly trying forum advice into someone who can observe the problem and make informed decisions. That's what this section is for.

### Why Forum Advice Is Contradictory

Before getting into tools, let's explain why the advice you've found online seems so random.

Person A has crackle on their ASUS board with an AMD GPU. They set `tsched=0` and it fixed it. Person B has crackle on the same ASUS board with an Nvidia GPU. They try `tsched=0` and nothing changes. Person C on an MSI board changes their sample rate from 48000 to 44100 and it clears up. Person D tries the same and it gets worse.

None of them are wrong. None are lying. They all have different noise environments, and each tweak interacts with those environments differently:

- **`tsched=0`** changes PulseAudio's scheduling from timer-based to interrupt-based. This changes the *timing* of when audio data moves through the pipeline. If your noise correlates with specific timing patterns (USB polling, PCIe transactions), changing the scheduling can accidentally avoid the problem. On a different board with a different noise correlation, it does nothing.

- **Changing sample rate** changes how often the DAC converts samples. Some noise artifacts alias differently at different rates. A rate change might push a noise artifact out of audible range on one board and into it on another.

- **Changing buffer sizes** changes how much audio is processed per chunk. Larger buffers absorb intermittent noise bursts. If your noise is continuous, buffer size changes nothing.

- **Changing bit depth** changes the noise floor characteristics. On some boards, 24-bit mode uses a different analog path or gain stage than 16-bit, with different noise susceptibility.

Every one of these tweaks adjusts a software parameter that interacts indirectly with a hardware noise problem. Whether it helps depends entirely on the specific noise on your specific board with your specific hardware under your specific workload. There is no universal correct setting because the underlying problem isn't software misconfiguration — it's hardware noise that software can sometimes incidentally mask.

### The Diagnostic Toolkit

If you're going to tweak, do it with your eyes open. Here's how to observe what's happening.

**PipeWire monitoring with `pw-top`:**

```bash
pw-top
```

Watch these columns for your audio sink:
- `QUANT`: Current buffer quantum (samples per cycle)
- `RATE`: Sample rate
- `WAIT`: Time spent waiting for hardware — if erratic, timing is unstable
- `BUSY`: Time spent processing — if this spikes, CPU isn't servicing audio promptly
- `ERR`: Buffer underruns/overruns — your crackle counter

ERR climbing means underruns. Erratic WAIT means timing instability. BUSY spikes mean scheduling issues. Each points to a different kind of problem.

**ALSA codec information:**

```bash
cat /proc/asound/card*/codec#*
```

Shows your codec configuration — widget types, pin configs, current settings. Compare to what the quirk table does for supported boards.

```bash
cat /proc/asound/card*/pcm*/sub*/hw_params
```

Shows current hardware parameters for active streams — format, rate, buffer size. Useful for confirming what's actually running versus what you think you configured.

**Kernel log:**

```bash
dmesg -w | grep -i "hda\|snd\|audio\|codec"
```

Watch for errors or warnings during playback. Codec timeouts, response errors, or power management transitions can correlate with glitches.

**Identify your codec and check quirk coverage:**

```bash
cat /proc/asound/card*/codec#* | head -5
```

Gives your codec vendor, product ID, and name. Then check your subsystem ID:

```bash
lspci -nn | grep -i audio
```

The `[XXXX:YYYY]` at the end is your subsystem ID. Search for it in [`patch_realtek.c`](https://github.com/torvalds/linux/blob/master/sound/pci/hda/patch_realtek.c) (or whichever `patch_*.c` matches your codec vendor). If it's there, your board has specific fixups. If not, you're running generic initialization.

### Things to Try (With Monitoring Running)

With `pw-top` or `dmesg -w` running alongside, you can observe the effect of each change rather than guessing.

**Adjust PipeWire quantum:**

```ini
# ~/.config/pipewire/pipewire.conf.d/tuning.conf
context.properties = {
    default.clock.rate = 48000
    default.clock.quantum = 1024
    default.clock.min-quantum = 512
}
```

If increasing the quantum reduces ERR counts and crackle, your problem involves bursty noise or timing that larger buffers absorb. If it doesn't help, your noise is continuous and buffer size isn't relevant.

**Try different sample rates:**

```ini
context.properties = {
    default.clock.rate = 44100
}
```

If switching between 44100 and 48000 changes the noise character, your problem involves frequency-dependent interference. Use whichever rate sounds better, but understand this could change with hardware modifications.

**HDA model options:**

```ini
# /etc/modprobe.d/snd.conf
options snd-hda-intel model=auto
```

Try model values listed in the [HD-Audio Models documentation](https://docs.kernel.org/sound/hd-audio/models.html) for your codec. Different models apply different pin configurations and initialization sequences. If a model option fixes your audio, you've found the quirk your board needs — consider submitting it upstream.

**Power saving:**

```bash
cat /sys/module/snd_hda_intel/parameters/power_save
```

If non-zero, the codec powers down between uses. Power transitions can cause clicks, pops, and change noise characteristics. The [kernel docs](https://docs.kernel.org/sound/hd-audio/notes.html) specifically note this. Try:

```ini
# /etc/modprobe.d/snd.conf
options snd-hda-intel power_save=0
```

If this eliminates clicks, the problem was power state transitions — a legitimate fix for that specific symptom.

**Stream position reporting:**

```ini
# /etc/modprobe.d/snd.conf
options snd-hda-intel position_fix=1
```

Values: 0 (auto), 1 (POSBUF), 2 (LPIB), 3 (VIACOMBO), 4 (COMBO). Some HDA controllers report incorrect stream positions, causing timing glitches. If changing this fixes things, the problem was position reporting, not noise — a real driver-level fix.

### What Results Tell You

- **Buffer/quantum changes help:** Your noise or timing issue is intermittent. Larger buffers absorb it. Least durable fix — any system timing change can break it.

- **Sample rate changes help:** Frequency-dependent interference. Semi-durable but could change with hardware changes.

- **Model option helps:** Your codec needs specific initialization it isn't getting. Most durable software fix — submit it upstream and it becomes permanent.

- **Power save off helps:** Power transitions causing glitches. Legitimate, durable fix for that symptom.

- **Position fix helps:** Controller reporting issue. Legitimate driver-level fix.

- **Nothing helps:** Continuous broadband noise, or interference that no software parameter usefully affects. You need a quirk-table-level fix or a DAC.

### Contributing Back

If you find a model option or configuration that fixes your board, consider contributing it. The process:

1. Identify your codec and subsystem ID (commands above)
2. Document what fix works (model option, HDA verbs, pin config)
3. Submit to the [ALSA development mailing list](https://alsa-project.org/wiki/Mailing-lists) or the kernel patch process

Even if you can't write a kernel patch, reporting your board model, codec, and what worked on the ALSA bug tracker gives maintainers information they can use. Every entry in that quirk table exists because someone took the time.

---

*The takeaway from all of this: Linux audio problems usually aren't mysterious. They're specific, diagnosable, and have identifiable causes. The frustration comes from not knowing which cause is yours, which means you don't know which fix applies. Hopefully this helps.*

---

## A Note on Sources

Everything in this post is backed by primary sources: kernel source code, Microsoft's own documentation, manufacturer statements, and developer testimony. I've linked them throughout, but here's a consolidated list for reference:

**Microsoft Documentation:**
- [Low Latency Audio (Windows Driver Documentation)](https://learn.microsoft.com/en-us/windows-hardware/drivers/audio/low-latency-audio) — Documents the pre-Windows 10 mandatory ~10ms buffer, the audio engine latency figures, and the Windows 10 changes to allow smaller buffers.
- [WASAPI Exclusive Mode Distortion at Low Buffer Sizes](https://learn.microsoft.com/en-us/answers/questions/832677/wasapi-exclusive-mode-audio-becomes-distorted-at-l) — Microsoft Q&A thread demonstrating USB audio clock issues even on Windows with the standard driver.

**Linux Kernel Source:**
- [patch_realtek.c (torvalds/linux)](https://github.com/torvalds/linux/blob/master/sound/pci/hda/patch_realtek.c) — Hundreds of board-specific audio fixups proving hardware needs per-board tuning. (Referenced in Sections 3/4.)
- [HD-Audio Driver Notes (kernel.org)](https://docs.kernel.org/sound/hd-audio/notes.html) — Official kernel documentation on HD-Audio quirks, model options, and board-specific workarounds.

**Manufacturer Statements:**
- [Realtek Downloads Page](https://www.realtek.com/Download/ToDownload?type=) — Realtek's own statement: *"Audio drivers available for download from the Realtek website are general drivers for our audio ICs, and may not offer the customizations made by your system/motherboard manufacturer."* (Referenced in Sections 3/4.)

**Linux Kernel Source (USB Audio):**
- [implicit.c (torvalds/linux)](https://github.com/torvalds/linux/blob/master/sound/usb/implicit.c) — Kernel implementation of implicit feedback for USB audio clock synchronization.
- [Linux Hardware Database: HyperX SoloCast](https://linux-hardware.org) — Documents USB 1.1 Full Speed operation (12 Mb/s), characteristic of UAC 1.0.

**Community Evidence / Bug Reports (Section 1 - Wine/Proton):**
- [Fedora Discussion: Wine/Proton PipeWire Stutters](https://discussion.fedoraproject.org/t/wine-proton-with-pipewire-sound-stutters-pipewire/126047) — Identifies low quantum value as the cause.
- [Ubuntu Bug #2086206: Wine/Proton Audio Stutter](https://bugs.launchpad.net/bugs/2086206) — Documents the problem as Wine/Proton-specific, not present with native apps.
- [Valve/Proton Issue #1147](https://github.com/ValveSoftware/Proton/issues/1147) — Long-running community thread on Proton audio crackling.
- [DxWnd Developer Discussion](https://sourceforge.net/p/dxwnd/discussion/general/thread/c518bfd4cd/?page=1) — Developer discovering Windows accepts incorrect audio parameters that Wine rejects.
- [Arch Linux Forums: Wine Audio Stuttering (Solved)](https://bbs.archlinux.org/viewtopic.php?id=276168) — Demonstrates the PULSE_LATENCY_MSEC workaround and its limitations.

**Community Evidence / Bug Reports (Section 1 - Discord/WebRTC/Electron):**
- [webrtcHacks: NetEQ Jitter Buffer](https://webrtchacks.com/how-webrtcs-neteq-jitter-buffer-provides-smooth-audio/) — Documents WebRTC's aggressive 10ms callback requirement: *"GetAudio returns exactly 10ms of audio samples. It MUST be called by the playout thread exactly 100 times per second."*
- [Chromium Issue #169075](https://issues.chromium.org/40959351) — "Total audio corruption with WebRTC and PulseAudio" affecting Linux.
- [Arch Wiki: Discord](https://wiki.archlinux.org/title/Discord) — Acknowledges crackling in voice chat and links to buffer underrun troubleshooting.
- [Arch Linux Forums: Audio Popping/Crackling](https://bbs.archlinux.org/viewtopic.php?id=307744) — Users observe `pw-top` showing ERR for Discord while other apps are clean.
- [Discord Support: Sample Rate Limitations](https://support.discord.com/hc/en-us/community/posts/360042925852-Support-for-higher-audio-sample-rates) — Confirms Opus codec requires 48kHz; higher rates cause complete audio failure.
- [Mozilla Bug #1430827](https://bugzilla.mozilla.org/show_bug.cgi?id=1430827) — WebRTC + PulseAudio latency analysis: *"caused by missing real-time audio callback deadlines"*

**Community Evidence / Bug Reports (Section 1 - PipeWire Quantum Clamping):**
- [Linux Mint Forums: PipeWire Crackling with Multiple Sources](https://forums.linuxmint.com/viewtopic.php?t=449235) — Explains the 2.7ms default buffer problem: *"an application needs to be able to fill sound content every 2.7 ms, otherwise pipewire will send... potentially some random data, hence the crackling."*
- [EasyEffects Issue #1567: Sporadic Audio Crackling](https://github.com/wwmm/easyeffects/issues/1567) — Discord notification crackling debugged; maintainer notes *"Discord's app is an endless source of problems"*; confirms fixed quantum *"reduces a lot the number of times I get a crackle."*
- [EndeavourOS Forums: PipeWire Crackling/Popping Fix](https://forum.endeavouros.com/t/pipewire-tweak-crackling-popping-fix/32860) — Users report various quantum values fixing output crackling.
- [PipeWire Docs: Protocol Pulse Module](https://docs.pipewire.org/page_module_protocol_pulse.html) — Official documentation on `pulse.min.quantum` and how PipeWire translates PulseAudio latency requests.
- [LinuxMusicians: PipeWire Dynamic Quantum](https://linuxmusicians.com/viewtopic.php?t=26617) — Explains quantum negotiation: *"applications will themselves request a quantum and PipeWire will use the last lowest requested value."*

**Community Evidence / Bug Reports (Section 2 - USB Microphone Clock Drift):**
- [Arch Linux Forums: USB Microphone Cracking/Noise (Solved)](https://bbs.archlinux.org/viewtopic.php?id=294015) — User discovers `snd_usb_audio.implicit_fb=1` and `lowlatency=0` fix for cheap Generalplus USB mic. Quote: *"sounds a thousand times better."*
- [LinuxMusicians: Mackie ProFX10v3 Problems (Solved)](https://linuxmusicians.com/viewtopic.php?t=26699) — Documents "clock source is not valid" errors fixed with `implicit_fb=1`.
- [Arch Linux Forums: ALSA Clock Source Invalid](https://bbs.archlinux.org/viewtopic.php?id=275707) — Technical discussion of clock validity errors and quirk_flags workarounds.
- [Samson Technologies: Ubuntu Compatibility](https://samsontech.zendesk.com/hc/en-us/articles/360045639093-Can-I-use-a-Samson-USB-microphone-with-Ubuntu) — Manufacturer confirms USB class-compliant operation (UAC 1.0).

**Community Evidence / Bug Reports (Section 2 - Voice Call Applications):**
- [PulseAudio Mailing List: Discord Voice Channel Crackling](https://lists.freedesktop.org/archives/pulseaudio-discuss/2017-November/029122.html) — User documents buffer underruns causing crackling in Discord voice chat. `tsched=0` *"eliminated initial crackling but introduced microphone static"* — partial fix showing timing-related root cause.
- [Arch Linux Forums: Skype Crackling Sound (Solved)](https://bbs.archlinux.org/viewtopic.php?id=164959) — Voice call crackling fixed with `PULSE_LATENCY_MSEC=30` — larger buffers absorb clock drift timing jitter.
- [Fedora Wiki: Glitch-Free Audio](https://fedoraproject.org/wiki/Features/GlitchFreeAudio) — Documents PulseAudio's timer-based scheduling and why `tsched=0` helps with certain hardware.
- [Focus Fidelity: Clock Drift Correction](https://www.focusfidelity.com/clockdriftcorrection) — Explains clock drift between USB mic ADC and system DAC: *"The most typical case is when using a USB microphone. The microphone has a built-in ADC and clock... independent of the clock in the DAC."*
