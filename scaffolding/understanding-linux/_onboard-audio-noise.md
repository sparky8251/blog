+++
title = "Onboard Audio Noise: The Hardware Problem"
description = "Why your motherboard's audio has noise Linux can't fix, and what you can actually do about it"
date = 2026-03-01
weight = 3
+++

# Onboard Audio Noise: The Hardware Problem

## The Problem

Your onboard audio has noise, crackle, hiss, or distortion. Not in Wine — everywhere. Native apps, music playback, maybe even a faint whine when your system is idle. Or maybe it's more subtle: audio that's mostly fine but develops crackle under load, or pops when other hardware is active, or hiss that gets louder when you scroll a webpage or move your mouse.

You've tried every PulseAudio and PipeWire tweak you can find. Some helped temporarily. Some helped for someone else but not you. Some made it worse. Nothing sticks.

---

## The Explanation

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

---

## The Proof

**Linux kernel source:** The quirk table in `patch_realtek.c` is primary evidence that boards objectively need individual tuning. Every entry exists because generic driver behavior was insufficient for that hardware. The [kernel HD-Audio documentation](https://docs.kernel.org/sound/hd-audio/notes.html) explicitly discusses model-specific options and the need for per-board configuration.

**Realtek's own statement:** They tell you their generic driver lacks manufacturer customizations. If generic was sufficient, this disclaimer wouldn't exist.

**Windows user experience:** Forum threads are full of people reporting that their motherboard manufacturer's audio driver sounds different from the generic Realtek or Windows drivers. People report [buzzing that only occurs with the manufacturer's Realtek driver and disappears with the generic Windows driver](https://www.tenforums.com/sound-audio/216697-realtek-driver-audio-buzzing-when-connecting-ethernet-cable-wifi-bt.html) — the manufacturer's driver is actively doing signal processing. People report BIOS updates fixing audio problems — compensation happens at the firmware level too.

**Manufacturer behavior:** ASUS ships Sonic Studio. MSI bundles Nahimic. ASRock's [support page](https://www.asrock.com/support/faq.asp?id=487) has instructions for enabling noise suppression in their Realtek audio console. Every manufacturer builds audio compensation software because the hardware needs it.

---

## Why This Is Harder Than the Other Two

The buffer problem has a clean fix: calculate your latency, set it per-app. The USB clock problem has a clean fix: get UAC 2.0. Both are deterministic.

Onboard audio noise is different. Your noise profile is unique. It depends on your specific motherboard, installed hardware, workload, even your power supply. A fix that works for someone with the same motherboard but a different GPU might not work for you, because the GPU changes the EMI environment.

The real, durable fixes are:

1. **Get your board into the kernel quirk table.** Figure out what fixups your board needs (by studying what the Windows driver does or experimenting with HDA verb commands) and submit a patch upstream. The kernel's [HD-Audio documentation](https://www.kernel.org/doc/html/latest/sound/hd-audio/notes.html) explains how to experiment with model options and codec verbs. Tools like `hda-analyzer` and `hda-verb` exist for this.

2. **Bypass onboard audio entirely with a UAC 2.0 USB DAC.** Move the analog conversion off the noisy motherboard onto an external device with its own clean power and no EMI from surrounding components. The digital signal over USB is immune to board noise. A $20-50 USB DAC eliminates the entire problem category permanently, regardless of kernel updates, driver changes, or hardware swaps.

I get that "buy a DAC" or "reverse-engineer your board's audio codec initialization and submit a kernel patch" are not what you want to hear. So here's the diagnostic toolkit.

---

## Why Forum Advice Is Contradictory

Before getting into tools, let's explain why the advice you've found online seems so random.

Person A has crackle on their ASUS board with an AMD GPU. They set `tsched=0` and it fixed it. Person B has crackle on the same ASUS board with an Nvidia GPU. They try `tsched=0` and nothing changes. Person C on an MSI board changes their sample rate from 48000 to 44100 and it clears up. Person D tries the same and it gets worse.

None of them are wrong. None are lying. They all have different noise environments, and each tweak interacts with those environments differently:

- **`tsched=0`** changes PulseAudio's scheduling from timer-based to interrupt-based. This changes the *timing* of when audio data moves through the pipeline. If your noise correlates with specific timing patterns (USB polling, PCIe transactions), changing the scheduling can accidentally avoid the problem. On a different board with a different noise correlation, it does nothing.

- **Changing sample rate** changes how often the DAC converts samples. Some noise artifacts alias differently at different rates. A rate change might push a noise artifact out of audible range on one board and into it on another.

- **Changing buffer sizes** changes how much audio is processed per chunk. Larger buffers absorb intermittent noise bursts. If your noise is continuous, buffer size changes nothing.

- **Changing bit depth** changes the noise floor characteristics. On some boards, 24-bit mode uses a different analog path or gain stage than 16-bit, with different noise susceptibility.

Every one of these tweaks adjusts a software parameter that interacts indirectly with a hardware noise problem. Whether it helps depends entirely on the specific noise on your specific board with your specific hardware under your specific workload. There is no universal correct setting because the underlying problem isn't software misconfiguration — it's hardware noise that software can sometimes incidentally mask.

---

## The Diagnostic Toolkit

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

---

## Things to Try (With Monitoring Running)

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

---

## What Results Tell You

- **Buffer/quantum changes help:** Your noise or timing issue is intermittent. Larger buffers absorb it. Least durable fix — any system timing change can break it.

- **Sample rate changes help:** Frequency-dependent interference. Semi-durable but could change with hardware changes.

- **Model option helps:** Your codec needs specific initialization it isn't getting. Most durable software fix — submit it upstream and it becomes permanent.

- **Power save off helps:** Power transitions causing glitches. Legitimate, durable fix for that symptom.

- **Position fix helps:** Controller reporting issue. Legitimate driver-level fix.

- **Nothing helps:** Continuous broadband noise, or interference that no software parameter usefully affects. You need a quirk-table-level fix or a DAC.

---

## Contributing Back

If you find a model option or configuration that fixes your board, consider contributing it. The process:

1. Identify your codec and subsystem ID (commands above)
2. Document what fix works (model option, HDA verbs, pin config)
3. Submit to the [ALSA development mailing list](https://alsa-project.org/wiki/Mailing-lists) or the kernel patch process

Even if you can't write a kernel patch, reporting your board model, codec, and what worked on the ALSA bug tracker gives maintainers information they can use. Every entry in that quirk table exists because someone took the time.

---

*Everything in this section is symptomatic treatment, not a cure. You're tuning around a hardware problem with software. It might work. It might work for a while and stop. It might break when you update your kernel, change your PipeWire config, or install new hardware. The underlying cause — your board's unique noise profile lacking proper driver compensation — hasn't changed.*

*But understanding what's happening turns you from someone blindly trying forum advice into someone who can observe the problem and make informed decisions. That's what this section is for.*

---

## Sources

**Linux Kernel Source:**
- [patch_realtek.c (torvalds/linux)](https://github.com/torvalds/linux/blob/master/sound/pci/hda/patch_realtek.c) — Hundreds of board-specific audio fixups proving hardware needs per-board tuning.
- [HD-Audio Driver Notes (kernel.org)](https://docs.kernel.org/sound/hd-audio/notes.html) — Official kernel documentation on HD-Audio quirks, model options, and board-specific workarounds.

**Manufacturer Statements:**
- [Realtek Downloads Page](https://www.realtek.com/Download/ToDownload?type=) — Realtek's own statement: *"Audio drivers available for download from the Realtek website are general drivers for our audio ICs, and may not offer the customizations made by your system/motherboard manufacturer."*

**Community Evidence:**
- [TenForums: Realtek Driver Audio Buzzing](https://www.tenforums.com/sound-audio/216697-realtek-driver-audio-buzzing-when-connecting-ethernet-cable-wifi-bt.html) — Demonstrates manufacturer's driver doing active signal processing that generic driver lacks.
- [ASRock Support: Audio Noise Suppression](https://www.asrock.com/support/faq.asp?id=487) — Manufacturer documentation on enabling noise compensation features.
