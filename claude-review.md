# Correctness Audit: WAV Format & Noise Generation

## Bugs Found

### 1. RIFF chunk size is 8 bytes too small (line 190)

The RIFF size field should equal everything after `RIFF` + size (i.e.,
file size minus 8). The content after those 8 bytes is:

| Component | Bytes |
|---|---|
| `WAVE` | 4 |
| `fmt ` chunk ID | 4 |
| fmt chunk size field | 4 |
| fmt chunk data | 16 |
| `data` chunk ID | 4 |
| data chunk size field | 4 |
| sample data | sample_data_len |

**Total = 36 + sample_data_len.** The code computes
`sample_data_len + 3*4 + 16 = sample_data_len + 28`. Should be `5 * 4`
not `3 * 4`. Most players tolerate this, but the file is technically
malformed.

### 2. Phase evolution has a constant negative rotation bias (lines 217-220)

```rust
rng.random::<f64>()
    * (hz as f64 / MAX_FREQUENCY as f64)
    * std::f64::consts::FRAC_PI_2
    - std::f64::consts::FRAC_PI_4,
```

Due to operator precedence this is `(random * hz/MAX_FREQ * PI/2) - PI/4`.
The random term ranges `[0, hz/MAX_FREQ * PI/2]`, then a fixed `PI/4` is
subtracted. At low frequencies where `hz/MAX_FREQ` is tiny, this collapses
to nearly `-PI/4` every interval — a systematic negative phase rotation
rather than a random drift. Over many seconds, low-frequency bins spin
deterministically. Likely intended to be zero-centered, e.g.,
`(random - 0.5) * (hz/MAX_FREQ) * PI/2`.

### 3. Grey noise A-weighting called with wrong frequency (line 157)

The closure receives `pos[1..]` (bins 1 through 22049), so closure-local
`hz = 0` corresponds to actual frequency bin 1. When grey does
`.enumerate().skip(20)`, closure `hz = 20` is actually bin 21 (21 Hz). But
`r_a(hz as f64)` calls `r_a(20.0)` instead of `r_a(21.0)`. The entire
A-weighting curve is shifted down by 1 Hz. Significant near the steep 20 Hz
region, negligible at higher frequencies.

### 4. One-sided imaginary part assertion (line 232)

```rust
assert!(sample.im < 1., "{}", sample.im);
```

Only catches positive imaginary parts. Should be `sample.im.abs() < 1.` to
also catch negative values indicating broken conjugate symmetry.

---

## Things That Are Correct

### WAV format structure (aside from RIFF size)
- `fmt ` chunk: field order, sizes, and values all match WAV PCM spec
  (format tag, channels, sample rate, avg bytes/sec, block align, bits per
  sample)
- `data` chunk: correctly written as chunk ID + size + raw samples
- `sample_data_len` matches actual bytes written
  (duration * 44100 * 2)
- Block align and avg bytes/sec are correctly computed

### Spectral slopes for all noise colors
All five colored-noise spectral slopes are correct, despite the off-by-one
between closure index and actual frequency bin (the `+1` in arithmetic
expressions compensates):

- **White**: flat amplitude — correct
- **Pink**: amplitude ~ `1/sqrt(f)`, power ~ `1/f` — correct
- **Brownian**: amplitude ~ `1/f`, power ~ `1/f^2` — correct
- **Blue**: amplitude ~ `sqrt(f)`, power ~ `f` — correct
- **Violet**: amplitude ~ `f`, power ~ `f^2` — correct

### A-weighting formula (lines 147-152)
Matches the standard IEC 61672-1 formula (as given on Wikipedia). The double
poles at 20.6 Hz and 12194 Hz are correctly reflected.

### Grey noise inversion logic
The approach of computing `target = avg_amplitude / A_weight_ratio` is
correct for producing perceptually flat noise — boosting where the ear is
insensitive, attenuating where it's sensitive.

### Conjugate symmetry (lines 225-228)
Correctly mirrors positive-frequency bins into negative-frequency bins with
conjugation. DC and Nyquist both set to zero.

### Random phase
Uniform random phase in `[0, TAU)` for each bin is the standard approach
for noise synthesis.

### Fade-in / dampen mechanism (lines 207, 231-236)
Linear fade-in over ~10,000 samples (~0.23s) to avoid an initial click.
Works correctly and only applies once at the start.

---

## Fragile but Not Broken

### No IFFT normalization
RustFFT does not normalize its inverse FFT output (result is scaled by
N=44100). The code doesn't divide by N. With `avg_amplitude = 8.0` and
random phases across ~22k bins, the RMS amplitude works out to roughly
`8 * sqrt(22050) ~ 1188`, which fits in i16 range. This works but is
fragile — changing `avg_amplitude` or the number of bins without
understanding this coupling would clip the output.

### Low-frequency cutoff inconsistency
Pink/Brownian/Grey skip bins below ~20 Hz; White/Blue/Violet don't. Not a
bug — the former have singularities at DC, the latter don't — but worth
noting.
