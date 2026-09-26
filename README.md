# Android USB audio FAST path @44100

A Magisk module that fixes **low-pitch / slow-motion audio through a 44.1 kHz USB
audio device on Android 16**, for apps that use low-latency (FAST) audio.

On affected devices, audio played by a low-latency source (e.g. a piano app built
on Oboe/AAudio) to a 44.1 kHz USB sink comes out **slowed down and pitched down by
a constant `44100/48000 = 0.91875x`**. Native playback and non-low-latency
("None") playback are correct. The same app worked on Android 15.

The module pins the device's **primary output** and **FAST** audio-policy profiles
to 44100, so the device's optimal output rate becomes 44100 and a low-latency
stream can get a **44100 FAST track that matches the sink** - correct pitch with
low latency.

- Tested on: TCL NXTPAPER 14 (`9491G`), MediaTek MT8781V (`mt6789`), Android 16
  (`CVAO`), kernel 6.12
- Tested sink: Roland FP-30X (USB Audio, 44.1 kHz, 24-bit packed)
- Reversible: disable the module in Magisk and reboot

---

## 1. Symptoms

- Audio through the USB device plays too slowly and at a lower pitch.
- The ratio is **constant**: `0.91875 = 44100 / 48000` (a 440 Hz tone sounds at
  ~404 Hz; everything runs ~8.8% long).
- It affects only **low-latency (FAST)** streams. Setting the app to the normal
  path (Oboe/AAudio `PerformanceMode::None`) fixes it.
- Output through the built-in speaker and Bluetooth are fine.
- It appeared with **Android 16**; the same device/app/sink on Android 15 worked.

A quick check while the app is playing:

```sh
adb shell dumpsys media.audio_flinger | grep -A1 -i 'FastMixer Timestamp'
# broken:  rate=0.91875 ... localSR(44100.x)
# fixed:   rate=1 ...       localSR(44100.x)
```

## 2. Root cause (Android 16, MediaTek)

The Android 16 audio policy/HAL opens the **primary/FAST output at 48000 Hz**
while the USB sink only supports **44100**. The FAST mixer does not resample, so
fast-path audio is clocked at `48000` nominal against a `44100` device:

```text
Output thread ... AudioOut_2D, type 2 (DUPLICATING):
  Sample rate: 48000 Hz
  AudioStreamOut: ... flags 0x4 (AUDIO_OUTPUT_FLAG_FAST)
  Timestamp stats: ... rate=0.91875 ... localSR(44100.8)
```

The vendor HAL knows the sink is 44100
(`AudioUSBCenter: setPrimaryOutSampleRate(), 48000 invalid!! use highest rate 44100`),
but AudioFlinger's output thread stays at 48000. Android 15 HIDL audio
HAL had the **identical** `fast`-to-USB route and profiles and worked fine, so this is
a rate-handling regression in the Android 16 AIDL audio HAL/policy, not the route
itself.

The normal MIXER path resamples 48000->44100 correctly, which is why
`PerformanceMode::None` is fine - but that path has higher latency, unusable e.g. for
audio-to-key sync in piano learning apps.

## 3. The fix

Pin the **primary output** and **fast** mix-port profiles to `44100` in
`/vendor/etc/audio_policy_configuration.xml`:

Before:

```xml
<mixPort name="primary output" role="source" flags="AUDIO_OUTPUT_FLAG_PRIMARY">
    <profile format="AUDIO_FORMAT_PCM_32_BIT" samplingRates="44100 48000" .../>
    <profile format="AUDIO_FORMAT_PCM_16_BIT" samplingRates="44100 48000" .../>
<mixPort name="fast" role="source" flags="AUDIO_OUTPUT_FLAG_FAST">
    <profile format="AUDIO_FORMAT_PCM_32_BIT" samplingRates="44100 48000" .../>
    <profile format="AUDIO_FORMAT_PCM_16_BIT" samplingRates="44100 48000" .../>
```

After:

```xml
    <profile format="AUDIO_FORMAT_PCM_32_BIT" samplingRates="44100" .../>
    <profile format="AUDIO_FORMAT_PCM_16_BIT" samplingRates="44100" .../>
```

The USB routes are left unchanged (still include `fast`).

### Why the primary output too?

Apps (and Oboe/AAudio) ask for the **optimal** rate,
`AudioManager.PROPERTY_OUTPUT_SAMPLE_RATE`, which is derived from the primary
output. Pinning only the `fast` profile to 44100 would not help: apps would still
request 48000 and fall back to the mixer. Pinning the **primary output** to 44100
makes the optimal rate 44100, so apps request 44100 and get a 44100 FAST track.

## 4. Install / verify / revert

```sh
adb push usb_fast_44100.zip /sdcard/Download/
adb shell su -c 'magisk --install-module /sdcard/Download/usb_fast_44100.zip'
adb reboot
```

Verify:

```sh
adb shell grep -A3 'mixPort name="primary output"' /vendor/etc/audio_policy_configuration.xml   # samplingRates="44100"
adb shell dumpsys media.audio_flinger | grep -B1 -A2 'name AudioOut_D'                          # Sample rate: 44100 Hz
adb shell dumpsys media.audio_flinger | grep -A1 -i 'FastMixer Timestamp'                        # rate=1, localSR 44100
adb shell dumpsys media.audio_flinger | grep -c 0.91875                                          # 0
```

Revert (then reboot):

```sh
adb shell su -c 'touch /data/adb/modules/usb_fast_44100/disable'
# or: adb shell su -c 'rm -rf /data/adb/modules/usb_fast_44100'
```

## 5. Results

Measured on the NXTPAPER 14 with the FP-30X:

| source | request | result |
| --- | --- | --- |
| OboeTester | 44100 + LowLatency | stays LowLatency; 44100 FAST output; FastMixer `rate=1.0002`, `localSR(44102)`; correct |
| OboeTester | 48000 + LowLatency | falls back to the mixer (44100-only FAST profile); correct, higher latency |
| Playground Sessions | rate 0 (optimal) | opens at **44100**, `perfMode=12`, gets the 44100 FAST output; correct |
| Piano Marvel | rate 0 (optimal) | opens at **44100**, `perfMode=12`; correct |

So apps that request the optimal rate (or 44100) get **low latency and correct
pitch**.

## 6. Adapting to another device

1. Pull the vendor policy:

   ```sh
   adb pull /vendor/etc/audio_policy_configuration.xml ./audio_policy_configuration.xml
   ```

2. In the primary module, set the `primary output` and `fast` mix-port profiles'
   `samplingRates` to the sink's rate (here `44100`). Leave the USB routes as they
   are.
3. Put the file at `system/vendor/etc/audio_policy_configuration.xml` in the
   module (`/system/vendor` is a symlink to `/vendor`) and zip
   `module.prop` + `system/`.
4. Install/reboot, then verify with the commands in section 4.

"Does this apply to me?" - yes if: a low-latency USB stream is pitch-shifted, the
ratio is ~`44100/48000`, `PerformanceMode::None` is correct, and the `fast`
profile can only run at 48000 while the sink is 44100.

## 8. Caveats

- **Device-wide rate change:** the whole primary output becomes 44100 (speaker and
  others too). Fine for most content; it is a global change.
- Apps that **hard-code 48000** with LowLatency will fall back to the mixer
  (correct, higher latency).
- **Firmware updates:** the module ships a full copy of the policy. After a vendor
  OTA, regenerate it from the new file or you may revert unrelated policy changes.
  Disable the module before flashing a full update.
- If a device's HAL forces 48000 regardless of the policy, this will not help.

## 9. FAQ

**Is this module AIDL-specific?**

Not in mechanism. It edits `audio_policy_configuration.xml`, a policy config
parsed by the framework's AudioPolicyManager; the `primary output` / `fast`
mix-port profiles mean the same thing for a HIDL or an AIDL vendor HAL (the 2FA6
HIDL and CVAO AIDL policies are nearly identical here). The *technique* is
HAL-interface-agnostic. What is build-specific is the packaged XML: it was taken
from CVAO/Android 16, so regenerate it from the target build's policy before
using the module elsewhere. The bug it works around happens to be an Android 16
AIDL-HAL regression, but the fix itself is not AIDL-specific.
