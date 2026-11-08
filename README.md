# JBL Quantum USB headsets on Linux: the silent-device / XRUN fix

A JBL Quantum **USB** headset that shows up, is selectable, responds to the volume
controls — and produces no sound, while everything that plays to it stalls. The
journal repeats `snd_pcm_avail after recover: Broken pipe` every couple of
seconds. Restarting PipeWire helps for a moment, then it comes back.

The cause is a per-device ALSA period that PipeWire cannot feed. Raising the
period and the headroom fixes it permanently:

- [`99-jbl-headsets.conf`](99-jbl-headsets.conf) — the WirePlumber drop-in
- Works on any distro using **WirePlumber 0.5+** (the current config format)
- Tested on a JBL Quantum **360X Wireless** and a **Quantum 350**, both over USB;
  see [Scope](#scope)

---

## Symptoms

| What you see | Notes |
|---|---|
| No sound from the headset | It is listed, selected, unmuted |
| Video stuck on "buffering", games "freeze" their audio | Any program playing to it stalls, because the device never consumes the buffer |
| `journalctl --user -u pipewire` repeats `spa.alsa: ... snd_pcm_avail after recover: Broken pipe` | Roughly every 2 s, forever |
| `systemctl --user restart pipewire` fixes it for seconds | Then the same state returns |
| It worked at first, or works on Windows | The failure is a timing race, not a broken device |

Two traps that cost real time:

- **It looks like a network problem.** A video sits on the buffering spinner, so the
  obvious suspect is wifi or the router. It is not: measured during the "buffering",
  receive throughput was 102 KB/s with 2 ms gateway RTT and 0% loss — the data
  arrives fine and the player blocks on a device that never consumes its buffer.
- **PipeWire says the node is `RUNNING`.** That state belongs to the *PipeWire node*,
  not to the hardware. The ALSA device can be stuck in `XRUN` at the same moment.
  Always check the hardware:

  ```bash
  head -1 /proc/asound/card*/pcm*p/sub0/status   # expect: state: RUNNING
  ```

## Root cause

Measured on the 360X (and the numbers are the whole story):

```
$ cat /proc/asound/card*/pcm*p/sub*/hw_params   # while a stream is open
period_size: 48        # frames -- 1 ms at 48000 Hz
buffer_size: 32768
```

- The headset opens at 48 kHz with a **period of 48 frames (1 ms)** on a 32768-frame
  buffer.
- PipeWire's graph runs in cycles of 1024 frames by default. It cannot deliver a
  1 ms period; the very first cycle underruns.
- ALSA then tries `snd_pcm_recover`, which fails with `Broken pipe`, and nothing
  restarts the stream. The PCM stays in XRUN → silence, and every write to it
  blocks → the stalling application.

This is why the usual advice does not stick: it is not a graph-size problem, so
changing the *PipeWire quantum* alone does not help. The **device** period has to
be raised, which is what `api.alsa.period-size` does.

## The fix

Two node properties, applied to any JBL USB playback and capture node:

```conf
api.alsa.period-size = 1024
api.alsa.headroom    = 8192
```

With those, the device comes up with `period_size 512`, which PipeWire feeds
comfortably. No more XRUN, no more log spam, audio plays.

### Why you will see both 1024 and 512 (and neither is wrong)

You configured 1024, so a first look at the node can look like the rule was ignored:

| Node state | `api.alsa.period-size` | `api.alsa.headroom` | `hw_params` `period_size` |
|---|---|---|---|
| **closed** (reports the request) | `1024` | `8192` | n/a |
| **streaming** (negotiated) | `512` | `8704` | `512` |

A closed node reports what the rule asked for. A streaming node reports what
PipeWire negotiated with the hardware: the period is halved and the difference is
moved into headroom (8192 → 8704), so total buffering is unchanged. `hw_params`
always reports the negotiated value, which is the number to trust while audio is
playing. A second PCM (the 360X chat channel) negotiates `period_size 256` the same
way.

### Install

Copy [`99-jbl-headsets.conf`](99-jbl-headsets.conf) to one of:

```bash
# system-wide
sudo install -Dm644 99-jbl-headsets.conf \
  /etc/wireplumber/wireplumber.conf.d/99-jbl-headsets.conf

# or per user
install -Dm644 99-jbl-headsets.conf \
  ~/.config/wireplumber/wireplumber.conf.d/99-jbl-headsets.conf
```

Then reload WirePlumber:

```bash
systemctl --user restart wireplumber
```

(Older WirePlumber 0.4 used `/etc/wireplumber/main.lua.d/` and a different syntax;
this file targets the 0.5+ format. Check with `wireplumber --version`.)

### Matching

The node rules match a regex, so they cover every JBL USB headset without
enumerating models:

```conf
{ node.name = "~alsa_output.usb-JBL.*" }
{ node.name = "~alsa_input.usb-JBL.*" }
```

## The 360X bonus: game and chat as two separate outputs

This part is **model-specific** and is the reason the shipped file also contains a
card rule. Everything below applies to the 360X; on other models it is inert.

The 360X exposes **two playback PCMs**:

| PCM | Format | Typical use |
|---|---|---|
| `pro-output-0` | 2 ch / 48000 Hz | game, music, everything normal |
| `pro-output-1` | **1 ch / 16000 Hz** | the chat channel |

They only become **separate PipeWire nodes** when the card uses the `pro-audio`
profile. Under the default profile the card presents a single output, so the
split is simply not addressable — no amount of per-application routing will give
you a chat-only output.

Selecting `pro-audio` therefore buys you a real game/chat split: keep the game on
`pro-output-0` and move Discord (or whatever voice app) to `pro-output-1`, and the
two are mixed independently on the headset.

Two things make that usable:

- **`pro-output-1` is mono 16 kHz.** You do not want your music or game audio
  landing there. The drop-in gives `pro-output-0` a higher `priority.session`
  (2500) so ordinary audio keeps choosing the 2-channel node as the default sink.
- **The profile is selected by exact card name**, deliberately:

  ```conf
  { device.name = "alsa_card.usb-JBL_JBL_Quantum_360X_Wireless-00" }
  actions = { update-props = { device.profile = "pro-audio" } }
  ```

  The 350, for example, has a single playback PCM, so there is no split to gain
  and its normal profile should be kept. Note its node name is
  `usb-JBL_JBL_Quantum350_Wireless-00` — no underscore before `350` — so a
  hand-written per-model rule copied from the 360X will silently miss it. To enable the split for another model,
  find your card name and add (or edit) that rule — see
  [Adapting to your model](#adapting-to-your-model).

### Routing per application

- GUI: `pavucontrol` → **Playback** → move the stream to `pro-output-1`.
- CLI: `pactl move-sink-input <stream-index> <sink-name>`
  (`pactl list sink-inputs short`, `pactl list sinks short`).

### Two headsets on one machine

Edge case worth knowing: those priorities do not add up across models. A 350 left
on its normal `analog-stereo` profile carries profile priority **6500**, which
outranks the 360X `pro-output-0` at **2500** — so the 350 wins the default sink.
Harmless with one headset per machine; raise rule 3 above 6500 if the 360X should
always win on a shared one.

## The gotcha that wastes the most time: WirePlumber remembers

WirePlumber stores the last profile **per card** in
`~/.local/state/wireplumber/default-profile`, and the default sink in
`default-nodes`. **A remembered choice beats the drop-in.** If the card was ever
set by hand — including to `off` — the `pro-audio` rule will never apply, and the
configuration looks like it is being ignored.

Check and clear:

```bash
# look for your card
grep -i jbl ~/.local/state/wireplumber/default-profile

# stop the daemon so it does not rewrite the file, drop the line, restart
systemctl --user stop wireplumber
sed -i '/alsa_card.usb-JBL/d' ~/.local/state/wireplumber/default-profile
systemctl --user start wireplumber
```

The same applies to a remembered default sink in `default-nodes`.

## Verify it worked

```bash
# 1. the hardware is running, not XRUN (while something is playing)
head -1 /proc/asound/card*/pcm*p/sub0/status      # expect: state: RUNNING
awk '/period_size|buffer_size/' /proc/asound/card*/pcm*p/sub0/hw_params
                                                  # expect period_size 512 (see above)

# 2. the log stops producing the line; this must stay at 0
journalctl --user --since '-2min' | grep -c 'snd_pcm_avail after recover'

# 3. play a test tone straight at the node (bypasses the desktop mixer)
pw-play --target=alsa_output.usb-JBL_JBL_Quantum_360X_Wireless-00.analog-stereo \
  /path/to/tone.wav

# 4. the card is on pro-audio (360X only)
pactl list cards short

# 5. both PCMs are addressable (360X only)
wpctl status | grep -E 'pro-output'

# 6. no errors accumulating on the node
pw-top
```

## Adapting to your model

```bash
# card name (for the device.profile rule)
pactl list cards short

# profiles available on the card, and the active one
pactl list cards | grep -E 'Name:|Active Profile:|pro-output'

# sink/source node names (for the period rule, if the JBL regex misses yours)
pactl list sinks short
pactl list sources short
```

## If it still does not work

| Check | Command / action |
|---|---|
| Is the file actually loaded? | `systemctl --user status wireplumber`, look for config errors; confirm the path is one WirePlumber scans |
| Does the node name match the regex? | `pactl list sinks short` — adapt the `node.name` regex |
| Still seeing XRUN, but rarer | Try `headroom = 16384` and/or `period-size = 2048`; USB controllers differ |
| Profile keeps reverting | See [the gotcha](#the-gotcha-that-wastes-the-most-time-wireplumber-remembers) |
| Nothing at all changed | WirePlumber 0.4? The 0.5 config format is ignored there |

## What the dongle does on plug (not a fault, but it confuses debugging)

Every plug produces **two enumeration events about 9 seconds apart**. The dongle
comes up in its other personality first and then re-enumerates into the simpler
(Xbox-compatible) mode:

```
20:44:15  new device 24, 0ecb:20a9  "JBL Quantum 360 Wireless", Mfr Harman International
20:44:23  USB disconnect, device 24
20:44:24  new device 25, 0ecb:20a5  ...        <- the card you actually use
```

So the card vanishing and reappearing while you are inspecting it is the dongle
switching modes, not a driver crash. Wait for the second event before drawing any
conclusion. (A 350 enumerates only once, `0ecb:206b`, so it does not do this.)

## Scope

- **Measured on:** JBL Quantum 360X Wireless (USB), PipeWire 1.6.x, WirePlumber
  0.5.x.
- **Confirmed on:** JBL Quantum 350 Wireless (USB) on 2026-11-06. Same failure,
  same fix, same `period_size 512`. The 350 exposes a single playback PCM, so the
  pro-audio game/chat split below does not apply to it and it should keep its
  normal profile.
- **The period fix** should apply to other JBL Quantum **USB** headsets, since it
  targets the device's own period rather than a model name. It is drafted that
  way on purpose.
- **The pro-audio section** is 360X-specific (it needs two playback PCMs).
- **Bluetooth is out of scope** — this only affects the USB audio path.
- Only the WirePlumber configuration is distro-specific; everything above is
  PipeWire/WirePlumber behaviour.

## Contributing

Issues with "model + distro + did it work" are welcome, so the list of known-good
models can grow. If you have the `hw_params` numbers from a model that still
misbehaves, include them — that is what identifies the period.

## License

[CC0 1.0 Universal](LICENSE) — public domain dedication. No conditions, no
attribution required. Take it, change it, ship it.
