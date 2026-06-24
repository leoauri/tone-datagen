# tone-datagen

Generates synthetic audio datasets of mixed sine tones for training instrument-like audio models.

## What it does

Each generated sample is a short audio clip containing a random number of sine tones layered together. Each tone has independently randomised frequency, amplitude, and envelope (attack, sustain, release), with a random onset delay to stagger entries. The mix is passed through a reverb effect and normalised, then exported as a fixed-length WAV file.

## Modules

### `tone_element` — single tone synthesis

`tone(freq, db, delay, attack, sustain, release)` produces a sine wave at the given frequency and amplitude (in dB), shaped by a delay–attack–sustain–release envelope.

```python
from tone_datagen.tone_element import tone

# 440 Hz at -6 dB, 0.1s delay, 0.05s attack, 0.3s sustain, 0.2s release
t = tone(440, -6, 0.1, 0.05, 0.3, 0.2)
```

### `tonegen` — randomised tone generation

Provides a composable probability distribution system and `RandomToneGen`, which draws each tone parameter from a supplied distribution. Distributions (normal, gamma, uniform) support arithmetic operators so complex shapes can be built inline.

```python
from tone_datagen.tonegen import RandomToneGen, gamma, uniform, clip

tonegen = RandomToneGen(
    freq_dist    = gamma(3) * 100 + 20,    # Hz, roughly 20–500 Hz
    db_dist      = gamma(2) * -2 - 2,      # dB, always negative
    delay_dist   = uniform(0, 1.5) + clip(gamma(0.9) * 0.2, high=4),
    attack_dist  = uniform(0.05, 0.6),
    sustain_dist = uniform() * 1.5,
    release_dist = uniform(0.05, 1),
)

stream = tonegen()  # returns a new random tone each call
```

Distributions expose a `.pdf()` method for quick visual inspection in a notebook.

### `generate` — mixing and export

`mixed_tones()` layers several randomly generated tones into a `Streamix`, applies a reverb tail and a global fade-out envelope, and returns a normalised NumPy array. `generate()` calls this in a loop and writes the results as numbered WAV files.

```python
from tone_datagen.generate import mixed_tones

mix = mixed_tones(rate=44100, longerness=2)  # ~2× longer notes and clips
```

## CLI usage

```sh
uv run -m tone_datagen.generate --outdir ./output --n_generate 100
```

Key options:
- `--outdir` — directory to write WAV files into (required)
- `--n_generate` — number of files to generate (default 30)
- `--longerness` — scale factor for clip and note durations (default 1)
- `--sample_len` — output length in samples, should be a power of two (default 2^18)
- `--rate` — sample rate in Hz (default 44100)

Files are named sequentially (`000000.wav`, `000001.wav`, …) and existing files in the output directory are continued from rather than overwritten.
