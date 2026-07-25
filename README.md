# Maqam Analysis — Turkish Makam Pitches in the Frequency Domain

**Why the piano cannot play Turkish classical music, demonstrated with Fourier analysis.**

Western music divides the octave into 12 equal semitones. Turkish classical music divides it into
**53 commas** — and the pitches that give a maqam its character fall in the gaps between piano
keys. This project synthesises nine maqam scales at mathematically exact 53-tone frequencies, then
uses FFT to show precisely where those pitches sit relative to the Western grid.

The result is a set of spectra where the perfect fourths and fifths line up almost exactly with
the Western reference lines — and the expressive notes visibly do not.

---

## The Core Idea

### 53-tone equal temperament

The octave is split into 53 equal steps. One step is the **Holdrian comma**:

```
1200 / 53 = 22.6415 cents
```

Any pitch is generated from a reference by its comma index:

```python
def get_freq(comma_index):
    return REF_FREQ * (2 ** (comma_index / 53.0))
```

With `REF_FREQ = 440 Hz` = **Dügah (A)** as comma 0. Traditional Turkish interval names map onto
comma counts:

| Interval | Turkish name | Commas | Cents |
|---|---|---|---|
| T | Tanini (whole tone) | 9 | 203.8 |
| K | Küçük mücennep | 8 | 181.1 |
| S | Bakiye-plus | 5 | 113.2 |
| B | Bakiye | 4 | 90.6 |
| A | Artık ikili (augmented) | 12–13 | 271.7–294.3 |

### Why 53, of all numbers?

53-TET is not an arbitrary choice — it approximates pure harmonic intervals better than 12-TET
does, by a wide margin:

| Interval | Just (pure) | 53-TET | error | 12-TET | error |
|---|---|---|---|---|---|
| Perfect fifth | 701.955¢ | 701.887¢ (31 commas) | **−0.07¢** | 700¢ | −1.96¢ |
| Perfect fourth | 498.045¢ | 498.113¢ (22 commas) | **+0.07¢** | 500¢ | +1.96¢ |
| Major third | 386.314¢ | 384.906¢ (17 commas) | −1.41¢ | 400¢ | **+13.69¢** |

The 53-TET fifth is accurate to seven hundredths of a cent — effectively pure. And note the third:
the maqam **Hicaz** pitch at 17 commas sits 1.4 cents from a *just* major third, while the piano's
major third is off by 13.7 cents. The "exotic" pitch is the one closer to natural harmony.

## The Money Table: Maqam Pitches vs the Piano

Every pitch used across the nine maqams, with its deviation from the nearest Western
equal-tempered note:

| Commas | Pitch name | Cents | Frequency | Nearest 12-TET | Deviation |
|---:|---|---:|---:|---:|---:|
| 0 | Dügah (A) | 0.0 | 440.00 Hz | 0 | 0.0¢ |
| 4 | Kürdi (B♭) | 90.6 | 463.63 Hz | 100 | −9.4¢ |
| 5 | Dik Kürdi (B♭) | 113.2 | 469.73 Hz | 100 | **+13.2¢** |
| 8 | Segâh (B−1) | 181.1 | 488.53 Hz | 200 | **−18.9¢** |
| 9 | Buselik (B) | 203.8 | 494.96 Hz | 200 | +3.8¢ |
| 13 | Çargâh (C) | 294.3 | 521.54 Hz | 300 | −5.7¢ |
| 17 | Hicaz (C♯) | 384.9 | 549.55 Hz | 400 | **−15.1¢** |
| 22 | Nevâ (D) | 498.1 | 586.69 Hz | 500 | −1.9¢ |
| 31 | Hüseynî (E) | 701.9 | 659.97 Hz | 700 | +1.9¢ |
| 35 | Acem (F) | 792.5 | 695.42 Hz | 800 | −7.5¢ |
| 39 | Evîç (F♯) | 883.0 | 732.77 Hz | 900 | **−17.0¢** |
| 44 | Gerdâniye (G) | 996.2 | 782.28 Hz | 1000 | −3.8¢ |
| 53 | Muhayyer (A) | 1200.0 | 880.00 Hz | 1200 | 0.0¢ |

**The pattern is the finding.** The structural skeleton — octave, fourth (Nevâ), fifth (Hüseynî) —
lands within 2 cents of the Western grid. The notes that carry a maqam's identity —
**Segâh (−18.9¢), Evîç (−17.0¢), Hicaz (−15.1¢), Dik Kürdi (+13.2¢)** — sit 13 to 19 cents away.

For scale: a semitone is 100 cents, and a trained listener can hear pitch differences of roughly
5–10 cents in melodic context. These deviations are **two to four times** that threshold. They are
not rounding error — they are the sound of the music, and a 12-tone instrument cannot produce them.

## Pipeline

### 1. Synthesis — `maqam_generator.ipynb`

Each note is built by **additive synthesis** rather than a bare sine, so the FFT has real harmonic
structure to show:

| Partial | Amplitude |
|---|---|
| f (fundamental) | 0.50 |
| 2f | 0.30 |
| 3f | 0.15 |
| 4f | 0.05 |

An attack/release envelope (50 ms / 100 ms linear ramps) prevents the click transients that would
otherwise smear broadband noise across the spectrum — important, because a click's energy would
sit right on top of the pitch peaks the analysis is trying to isolate.

Each maqam is rendered as an **ascending then descending** scale, 0.6 s per note with 0.1 s gaps,
peak-normalised to 0.9, written as 16-bit 44.1 kHz WAV (10.50 s per file). A companion CSV logs
every note's maqam, name, comma index and exact frequency.

### 2. Analysis — `maqam_fourier_analysis.ipynb`

- `scipy.fft.rfft` over the **entire** 10.5 s file — a 463k-sample window giving ~0.1 Hz frequency
  resolution, far finer than the 6.4 Hz gap between the closest two pitches in the set
  (Segâh 488.53 Hz vs Buselik 494.96 Hz).
- `scipy.signal.find_peaks` with a 10%-of-max height floor and minimum peak separation, isolating
  the true partials from spectral leakage.
- Detected peaks are matched against a maqam pitch table and auto-labelled with their Turkish
  names.
- Each plot overlays **green dashed lines at Western equal-tempered frequencies**, so the offset
  between a maqam peak and the nearest piano key is visible directly on the axis.

Output: nine annotated spectra, one per maqam, over the 350–900 Hz range.

## The Nine Maqams

| Maqam | Tonic | Comma offsets | Character |
|---|---|---|---|
| **Rast** | Rast (G) | −9, 0, 8, 13, 22, 31, 39, 44 | The foundational maqam; T−K−S−T−T−K−S |
| **Buselik** | Dügah (A) | 0, 9, 13, 22, 31, 35, 44, 53 | Closest to the Western natural minor |
| **Nihavend** | Dügah (A) | 0, 9, 13, 22, 31, 35, 44, 53 | Same scale as Buselik, different *seyir* |
| **Kürdi** | Dügah (A) | 0, 4, 13, 22, 31, 35, 44, 53 | Kürdi tetrachord: B−T−T |
| **Uşşak** | Dügah (A) | 0, 8, 13, 22, 31, 35, 44, 53 | Segâh second — the archetypal neutral interval |
| **Hüseynî** | Dügah (A) | 0, 8, 13, 22, 31, 39, 44, 53 | Uşşak, but with Evîç instead of Acem |
| **Hicaz** | Dügah (A) | 0, 5, 17, 22, 31, 35, 44, 53 | Hicaz tetrachord S−A−S = 5+12+5 = 22 commas (a pure fourth) |
| **Segâh** | Segâh (B) | 8, 13, 22, 31, 39, 44, 53, 61 | Tonic is *not* A — the whole spectrum shifts |
| **Sabâ** | Dügah (A) | 0, 8, 13, 17, 31, 35, 44, 53 | The most distinctive interval set in the tradition |

Three of these are deliberately chosen as analytical test cases:

- **Uşşak vs Hüseynî** differ at exactly one degree (Acem 35 vs Evîç 39). Their spectra are
  identical except for a single peak moving 37 Hz — a clean visual proof that one comma-level
  choice defines a maqam's identity.
- **Segâh** is transposed to a non-A tonic, shifting every peak and confirming the analysis is not
  quietly hard-coded around 440 Hz.
- **Hicaz** contains the augmented second (12 commas), the widest step in the set.

## A Genuine Result: What Fourier *Cannot* See

**Buselik and Nihavend produce byte-identical audio files** (verified: same MD5 checksum). This is
not a bug — it is the honest conclusion of the method, and it is the most interesting thing the
project demonstrates.

The two maqams share the same scale. What separates them is *seyir* — the melodic progression, the
order in which pitches are approached, which degrees are emphasised, where phrases come to rest.
An FFT over a whole scale discards all temporal information by construction, so it is
mathematically incapable of telling them apart.

The takeaway generalises: **pitch content identifies some maqams and not others.** Maqams
distinguished by interval structure (Uşşak vs Hüseynî) separate cleanly in the frequency domain.
Maqams distinguished by melodic behaviour (Buselik vs Nihavend) require time-frequency analysis —
a spectrogram or STFT — or sequence modelling. Knowing which problem you have is the prerequisite
for choosing the right tool.

## Repository Contents

| Path | Description |
|---|---|
| `maqam_generator.ipynb` | 53-TET synthesis: pitch math, additive synthesis, envelope, WAV + CSV export |
| `maqam_fourier_analysis.ipynb` | FFT, peak detection, pitch labelling, Western-overlay plots |
| `maqam_53tet/*.wav` | Nine synthesised maqam scales (44.1 kHz, 16-bit, 10.5 s each) |
| `maqam_53tet/maqam_analysis_data.csv` | 72 rows: maqam, note name, comma index, exact frequency |

## Running It

```bash
pip install numpy scipy matplotlib
```

Run `maqam_generator.ipynb` first — it creates the `maqam_53tet/` folder and populates it. Then run
`maqam_fourier_analysis.ipynb`, which reads every `.wav` in that folder and produces one annotated
spectrum per maqam. The generated audio is already committed, so the analysis notebook can be run
standalone. Developed on Python 3.13.

## Notes & Limitations

- **Synthetic, not performed.** These are mathematically exact tones. Real performers apply
  vibrato, glide between pitches, and bend intervals expressively — a live *ney* or *tanbur*
  recording would show far messier spectra. The synthetic approach is the right starting point
  precisely because it isolates the pitch question from performance variation, but it is a
  controlled baseline, not a claim about live music.
- **Comma values vary by theorist.** Turkish music theory is not fully standardised; Arel-Ezgi-Uzdilek
  comma assignments differ from Yekta's and from practice. The Kürdi degree here is taken as 4
  commas (bakiye) where some sources use 5. The comments in the generator flag these choices where
  they are contested.
- **Ascending forms only.** Several maqams — Hüseynî most notably — use different pitches ascending
  and descending (Evîç going up, Acem coming down). Only the ascending form is synthesised, chosen
  because it maximises the spectral contrast with Uşşak.
- The reference pitch table in the analysis notebook uses hand-entered approximate frequencies with
  a ±10 Hz matching tolerance. It labels the current set correctly, but deriving it from
  `get_freq()` instead of literals would make it exact and self-maintaining.

## Roadmap

- **STFT / spectrogram** — recover the time axis and make *seyir* visible, closing the Buselik /
  Nihavend gap.
- **Real recordings** — run the same pipeline on performed maqam audio and measure how far live
  intonation drifts from theoretical 53-TET.
- **Classification** — with spectra as features, train a model to identify maqam from audio; the
  results here predict which pairs it will confuse and why.
- **Cent-deviation plots** — chart each maqam's degrees against the 12-TET grid directly in cents.

## Tech Stack

Python · NumPy · SciPy (`fft`, `signal`, `io.wavfile`) · Matplotlib

---

*Turkish classical music theory meets digital signal processing. The commas are not out of tune —
the piano is.*
