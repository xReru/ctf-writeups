---
description: 'Author: 0xreru'
---

# 🧩 (MISC) Mission : Impossible Writeup

Flag: `TSGCTF{Th1S_fl4g_wiLL_s3lf-deSTrucT_in_5_s3c0nds}`

***

### Challenge Overview

We are tasked with infiltrating a state-of-the-art CIA vault room. The room is protected by a suite of pressure, temperature, and audio-sensitive sensors. To interact with the terminal, we must provide a voice command. However, the catch is obvious: a standard voice command would trigger the audio sensors and alert the guards.

***

### Initial Analysis

**Understanding the Source Code**

The challenge provides a Python script utilizing Gradio for the interface. To beat this, we have to look closely at how the "security system" actually processes our sound.

*   Noise Cancellation Gate: The system effectively "mutes" anything too quiet to be a real command. If the magnitude of the frequency space is less than `0.01`, it's zeroed out.

    Python

    ```
    freq_space = librosa.stft(wave, n_fft=N_FFT)
    freq_space[np.abs(freq_space) < 0.01] = 0  # noise cancellation
    ```
*   The "Intruder" Logic: This is where it gets tricky. The function `detect_intruder` calls `cut_high_freqs`, which removes all frequencies below 10,000 Hz. If _anything_ remains in that frequency space afterward, the alarm sounds.

    Python

    ```
    def cut_high_freqs(freq_space, sr, cutoff_freq):
        cutoff_bin = int(cutoff_freq * N_FFT / sr)
        freq_space[cutoff_bin:, :] = 0 # Removes high freqs

    def detect_intruder(freq_space, sr):
        # Monitors everything BELOW 10kHz for speech
        cut_high_freqs(freq_space, sr, 10000) 
        magnitude = np.abs(freq_space).max()
        return magnitude > 0 # Alert if human speech detected
    ```
*   The Payoff: If you manage to bypass that check, the system resamples your audio and hands it over to OpenAI's Whisper. If Whisper hears "give me the flag," you win.

    Python

    ```
    if "give me the flag" in result.lower():
        return "OK, here is the flag: " + FLAG
    ```

**The Contradiction**

Human speech primarily exists in the 300 Hz to 3,000 Hz range. The system detects an "intruder" if it hears _anything_ in the audible spectrum below 10,000 Hz. This means any normal audio command will trigger the alarm before the transcription even starts.

***

### Chain of Thought & Breakthroughs

**The Challenge: Silent Speech**

How do we send a voice command that Whisper can hear, but the intruder detector—which specifically monitors the audible spectrum—ignores?

**The Breakthrough: Ultrasonic Modulation**

Microphones often capture frequencies far higher than the human hearing limit of 20kHz. If we can move our voice command into the ultrasonic range (above 10kHz), we can bypass the `detect_intruder` check.

**The "Folding" Hack: Aliasing and Resampling**

The secret weapon is how `librosa.resample` behaves. If we place a signal at a high frequency and then downsample it without proper anti-aliasing filters, the high-frequency signal "folds" back into the lower frequency range—a phenomenon known as aliasing. By hiding our audio in the high-frequency "safe zone," we bypass the filter, and the system's own downsampling "reveals" the voice to Whisper.

***

### Deep Dive: Implementation

We use Single Sideband (SSB) Modulation. This shifts the entire frequency spectrum of our voice command up by a "carrier" frequency without distorting the underlying linguistic structure.

1. Base Voice Generation: Use `gTTS` to generate a high-quality "give me the flag" recording.
2. SSB Modulation: Use a Hilbert transform to shift the voice up to a 16kHz carrier. This pushes the speech into the 16kHz - 20kHz range—well above the 10kHz alarm limit.
3. Stealth Filtering: Apply a 20th-order High-Pass Filter at 11kHz to ensure zero signal leakage exists in the "audible" 0-10kHz range.
4. Normalization: Boost the volume to `0.95` to ensure it isn't killed by the `0.01` noise gate.

***

#### and with the help of AI we arrive at this solution script

Python

```
import numpy as np
import librosa
import soundfile as sf
from scipy.signal import butter, sosfilt, hilbert
from gtts import gTTS
import io

# Config
TARGET_PHRASE = "give me the flag"
SR = 44100          
CARRIER = 16000     # Carrier freq pushes audio into the ultrasonic range
CUTOFF = 11000      # Higher than the 10kHz detector limit

print("[*] Generating high-quality base voice...")
tts = gTTS(TARGET_PHRASE, lang='en')
fp = io.BytesIO()
tts.write_to_fp(fp)
fp.seek(0)
y, _ = librosa.load(fp, sr=SR)
y, _ = librosa.effects.trim(y)

print("[*] Applying SSB Modulation (Upper Sideband)...")
# Hilbert transform shifts frequency without mirroring the spectrum
t = np.arange(len(y)) / SR
analytic_signal = hilbert(y)
y_shifted = np.real(analytic_signal * np.exp(2j * np.pi * CARRIER * t))

print("[*] Applying 20th-order High-Pass Filter...")
# Steep filter to ensure 0-10kHz is completely empty
sos = butter(20, CUTOFF, fs=SR, btype='highpass', output='sos')
y_stealth = sosfilt(sos, y_shifted)

print("[*] Final Normalization...")
# Set to 0.95 to be loud enough for the 0.01 noise gate
y_final = y_stealth / np.max(np.abs(y_stealth)) * 0.95

sf.write('clean_stealth.wav', y_final, SR)
print("[+] Created clean_stealth.wav")
```

### Result <a href="#result" id="result"></a>

Upon uploading `clean_stealth.wav`:

1. `detect_intruder` scans the 0-10kHz range and finds nothing.
2. `librosa.resample` downsamples the audio to 16kHz. The 16kHz modulation causes the signal to alias perfectly back into the 0-4kHz range.
3. Whisper transcribes the "folded" audio as `"give me the flag"`.

<figure><img src="../../../.gitbook/assets/image (22).png" alt=""><figcaption></figcaption></figure>

System Message: `OK, here is the flag: TSGCTF{Th1S_fl4g_wiLL_s3lf-deSTrucT_in_5_s3c0nds}`

***

#### Key Takeaways

* Aliasing is a Feature: Usually a bug, here it was our "stealth transport" to bypass a frequency filter.
* AI Robustness: Whisper is capable of transcribing aliased audio if the phonemic structure is preserved.
* SSB Modulation: Much more precise than pitch shifting, as it allows for relocation without changing the audio's duration.

#### References

* [SSB Modulation via Hilbert Transform](https://en.wikipedia.org/wiki/Single-sideband_modulation)

Flag: `TSGCTF{Th1S_fl4g_wiLL_s3lf-deSTrucT_in_5_s3c0nds}`

If you have any questions feel free to dm me [xreru](https://app.gitbook.com/u/sV63NjWn0kbva4C066LUjfLD3y92 "mention") or in [linkedin](https://www.linkedin.com/in/reru/)
