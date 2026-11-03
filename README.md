# LTX (Legacy Tetrode X) Format

LTX is a family of file formats designed for tetrode recordings in freely moving rodents. It covers general settings, spikes recorded on tetrodes, local field potentials, the animal's position and digital events.

This document attempts to describe the format, which is only loosely defined by convention.

> [!WARNING]
> As of 3 November 2026, this document may still contain inaccuracies. Expect some edits in due course.

## What it's for

A typical experiment has a mouse or rat exploring an arena about a metre across, while tetrodes record from its
brain. A tether runs from the headstage to a commutator above the arena, and an overhead camera tracks two LEDs, one
big and one small, on the animal's head.

During the recording:

- An acquisition board samples the headstage channels, typically at 30 kHz. Channels are grouped into tetrodes of four.
- Spikes are detected online. Only a short waveform on all four channels of the tetrode is kept for each spike, not
  the broadband signal.
- A few channels are kept as continuous data at 5 kHz, to capture the LFP, including the theta rhythm.
- The camera gives the position of each LED, usually at around 50 Hz. The two LEDs also give the head's direction.
- Digital inputs record sync pulses and stimuli.

Afterwards, spikes are clustered into putative cells, automatically and then by hand. Each cell's firing can then be
related to the animal's behaviour, with spatial rate maps, spatial autocorrelograms, directional and speed tuning,
and temporal autocorrelograms.

A recording session is a **trial**, and a day's work is usually several trials, for example several sessions in the
same arena or in different ones. Each trial is its own set of files sharing a name.

## Tools

| Tool | Role | Files |
| --- | --- | --- |
| [Waveform](https://github.com/ChenNeuroscience/waveform) | Browser-based viewer and manual cluster editor | Reads `.set`, `.N`, `.pos`, `.cut`, `.clu.N`. Writes `.cut`, `.clu.N`. Never modifies the originals. |
| [ltx-klusta](https://github.com/ChenNeuroscience/ltx-klusta) | Automatic spike clustering, derived from [KlustaKwik](https://github.com/klusta-team/klustakwik) | Reads `.N`. Writes `.clu.N`. |
| [ephys-rs](https://github.com/ChenNeuroscience/ephys-rs) | Recording software, partly derived from [Open Ephys](https://github.com/open-ephys) | Supports writing `.set`, `.N`, `.egf*`, `.pos`, `.ttl`. |

Waveform can simulate somewhat realistic data and display it in one click: [try it here](https://chenneuroscience.github.io/waveform/).

ephys-rs can simulate somewhat realistic data in real time, so you can run through a whole recording session without any hardware. Its synthetic streams include an acquisition board, with spiking units over a theta rhythm, and a fake video feed of a mouse with two head LEDs running around a box, trailing its cable.

## The files in a trial

Every file in a trial shares a base `name`, e.g. `2026-11-03_a`.

| File | Contents | Format |
| --- | --- | --- |
| `name.set` | Trial settings | Header only |
| `name.1`, `name.2`, ... | Spikes, one file per tetrode, numbered from 1 | Header and binary |
| `name.egf`, `name.egf2`, ... | Continuous data at 5 kHz, one file per channel | Header and binary |
| `name.pos` | LED positions | Header and binary |
| `name.ttl` | Digital input changes | Header and text |
| `name.clu.N` | Cluster labels for tetrode N | Text, no LTX header |
| `name_N.cut` or `name.N.cut` | Cluster labels for tetrode N | Text, no LTX header |

Writers may also leave these non-LTX files beside the data:

| File | Written by | Contents |
| --- | --- | --- |
| `name.ephys-rs.json` | ephys-rs | The node graph and settings used for the recording |

## Common file layout

All the header-and-data files look like this, with `\r\n` between lines:

```text
trial_date Tuesday, 03 Nov 2026
trial_time 14:30
created_by ephys-rs 0.37.0
duration 1200
key value
...
data_start<data>
data_end
```

- Each header line is a key, one space, then a value. The rest of the line is the value, which can contain spaces.
- The data begins with the byte straight after `data_start`, with no line break, and is followed by `\r\ndata_end`.
- Binary multi-byte values are **big-endian**.
- Values that carry a unit include it, such as `96000 hz`.

### Reading the header safely

- **Trim values.** Values filled in when recording stops (`duration` and the counts) sit in a space-padded placeholder, so
  they have trailing spaces.
- **Use the count headers to find the end of the data.** Don't search for `data_end`, because those bytes can occur in
  binary data. Find `data_start` at the start of a line, and read the header as Latin-1.
- **Expect no markers if there is no data.** `data_start` and `data_end` are only written along with data. A `.set`
  file never has them, and neither does a tetrode with no spikes, or a `.pos` or `.ttl` with no samples. Those files
  end in the padding left in the header.
- **Treat blank counts as an interrupted recording.** If a recording doesn't finish, `duration` and the count stay
  blank, and `data_end` is missing.

### Headers in every file

| Key | Example | Meaning |
| --- | --- | --- |
| `trial_date` | `Tuesday, 03 Nov 2026` | Local date that recording started |
| `trial_time` | `14:30` | Local time that recording started, `HH:MM` |
| `created_by` | `ephys-rs 0.37.0` | Writer and version |
| `duration` | `1200` | Length of the recording, in whole seconds of wall-clock time |

## Data files

### `.set`: trial settings

Header only. Besides the common headers:

| Key | Example | Meaning | Writer |
| --- | --- | --- | --- |
| `lasttrialdatetime` | `1793739079` | Start of the recording, in Unix seconds | Both |
| `lightBearing_1`...`4` | `0`, `180`, `0`, `0` | Fixed values, for compatibility | Both |
| `colactive_1`...`4` | `1`, `1`, `0`, `0` | Fixed values, for compatibility | Both |
| `ADC_fullscale_mv` | `1500` | Full scale at a gain of 1, in mV | ephys-rs |
| `gain_ch_N` | `6000` | Gain of spike channel N, numbered from **0** across all tetrodes | ephys-rs |
| `EEG_ch_N` | `5` | Which upstream channel (from 1) `.egf` number N holds | ephys-rs |
| `EEG_gain_N` | `7500` | Gain of `.egf` number N, numbered from **1** | ephys-rs |

**Converting to µV.** For a sample `s` from an ephys-rs recording:

```text
µV = s / 128 × (ADC_fullscale_mv × 1000) / gain
```

So `gain_ch_0 6000` covers ±250 µV. Channel `c` (0–3) of tetrode `t` uses `gain_ch_{4(t-1)+c}`.

Open Ephys `.set` files have no gains, so the file doesn't record the voltage scale. See the notes on scaling under
[`.1`, `.2`...](#1-2-tetrode-spikes) and [`.egf`](#egf-egf2-continuous-data).

### `.1`, `.2`...: tetrode spikes

| Key | Value | Meaning |
| --- | --- | --- |
| `num_chans` | `4` | Channels per spike |
| `bytes_per_timestamp` | `4` | |
| `samples_per_spike` | `50` | Samples stored per channel |
| `bytes_per_sample` | `1` | |
| `spike_format` | `t,ch1,t,ch2,t,ch3,t,ch4` | Layout of a spike |
| `sample_rate` | `30000.000000 hz` | The rate of the waveform samples |
| `timebase` | `96000 hz` | Ticks per second of the timestamps |
| `num_spikes` | `125000` | Number of spikes |

Each spike is a 216-byte record: four 54-byte blocks, one per channel.

| Offset in block | Type | Contents |
| --- | --- | --- |
| 0 | int32, big-endian | Timestamp in `timebase` ticks, the same in all four blocks |
| 4 | 50 × int8 | Waveform samples |

- **Length of the waveform.** Only the first 40 samples are normally real data, 8 before the spike and 32 after it.
  The Open Ephys plugin requires exactly 40. Both writers fill the rest with zeros, to keep the 50-sample layout
  legacy readers expect.
- **Scaling.** In ephys-rs, a Spike Gain node scales the spikes into the int8 range, and the gains are stored in
  `.set`.
- **Limits of the readers.** Waveform rejects anything other than this layout, and accepts up to 20 million spikes.
  ltx-klusta needs `num_spikes` to be at least 2.

### `.egf`, `.egf2`...: continuous data

One channel per file: `.egf` for the first, then `.egf2`, `.egf3` and so on.

| Key | Example | Meaning | Writer |
| --- | --- | --- | --- |
| `num_chans` | `1` | | Both |
| `sample_rate` | `5000 hz` | Always 5 kHz | Both |
| `stream_name` | `eeg` | Name of the source stream | Both |
| `chan_name` | `CH5` | Name of the channel | Both |
| `num_EEG_samples` | `6000000` | Number of samples | Both |
| `gain` | `7500` | Same as `EEG_gain_N` in `.set` | ephys-rs |
| `range_uv` | `200.0` | µV either side of zero that the int8 range covers | ephys-rs |

The data is one int8 per sample, with no timestamps. Sample `i` is at `i / 5000` seconds from the start of this file.
There is no `bytes_per_sample` header.

- **Downsampling.** The writers keep every Nth sample (every 6th at 30 kHz), with no filtering of their own, so any
  anti-aliasing has to happen upstream. The standard ephys-rs graph band-passes this branch at 1–300 Hz. ephys-rs
  accepts any input rate that is a multiple of 5 kHz. The Open Ephys plugin requires exactly 30 kHz.
- **Scaling.** With ephys-rs, `µV = sample / 128 × range_uv`. The Open Ephys plugin halves the incoming value, truncates
  it and clamps it, so the input is expected within ±250. It does not record the scale.

### `.pos`: positions

| Key | Example | Meaning |
| --- | --- | --- |
| `timestamp_timebase` | `96000 hz` | Ticks per second of the timestamps |
| `timebase` | `50.000000 hz` | **The sample rate**, not the rate of the timestamps, unlike in tetrode files |
| `sample_rate` | `50.000000 hz` | Position samples per second |
| `bytes_per_timestamp` | `4` | |
| `bytes_per_coord` | `2` | |
| `pos_format` | `t,x1,y1,x2,y2,numpix1,numpix2` | Layout of a sample, before its padding |
| `num_colours` | `4` | Fixed value, for compatibility |
| `bearing_colour_1`...`4` | `330`, `150`, `0`, `0` | Fixed values. Waveform does use `bearing_colour_1`, as the LEDs' angle in degrees when working out direction from two LEDs. |
| `pixels_per_metre` | `567.78` | Calibration of the **stored** coordinates. `NaN` or `nan` if uncalibrated. |
| `window_min_x`, `window_max_x`, `window_min_y`, `window_max_y` | `0`, `1022`, `0`, `767` | Arena bounds in stored coordinates |
| `min_x`, `max_x`, `min_y`, `max_y` | | Same values as the `window_*` keys |
| `num_pos_samples` | `60000` | Number of samples |

Each sample is a 20-byte record, all big-endian:

| Offset | Type | Contents |
| --- | --- | --- |
| 0 | int32 | Timestamp in `timestamp_timebase` ticks |
| 4, 6 | uint16 | `x1`, `y1`: first LED, normally the big one |
| 8, 10 | uint16 | `x2`, `y2`: second LED |
| 12, 14 | uint16 | `numpix1`, `numpix2`: LED areas in camera pixels, saturating at 65535 |
| 16, 18 | uint16 | Padding, always zero and not mentioned in `pos_format` |

- **Missing positions.** A coordinate of **1023** means there is no usable position for that LED, and an LED that wasn't found
  also has an area of 0. Waveform treats an LED as missing if x or y is 1023, or its area is 0.
- **Uncalibrated files.** Waveform requires `pixels_per_metre` to be positive, so it rejects a `.pos` with `NaN`.

**Coordinates.** Real camera coordinates can reach 1023 and beyond, so coordinates are stored relative to the crop's
bounding box and scaled down if needed to fit 0–1022:

```text
scale    = min(1, 1022 / max(window_width, window_height))
stored_x = round((camera_x - window_min_x) * scale)
```

`pixels_per_metre` and the `window_*` headers are in stored coordinates, so readers can use them without knowing about
this transform. These headers record the original camera geometry:

| Key | Example | Meaning |
| --- | --- | --- |
| `position_metadata_source` | `camera_tracker` | `camera_tracker`, or `legacy_viewer` for Open Ephys streams without camera metadata, which have none of the keys below |
| `data_source` | `ephys-rs 0.37.0` | The source of the positions. `open-ephys-plugin-camera v0.1` is a placeholder rather than a real version. |
| `camera_tracker.pixels_per_metre` | `800` | Calibration in camera pixels |
| `camera_tracker.window` | `100 50 1540 1130` | Crop's bounding box in camera pixels: min x, min y, max x, max y |
| `camera_tracker.crop` | `rectangle(cx,cy,w,h,deg) 820 590 1440 1080 17.19` | Crop shape. The text comes from the source, so its form varies by writer. |
| `camera_tracker.frame_size` | `1920 1200` | Width and height of the camera frame |
| `position_coordinate_transform` | `crop_relative_uniform_scale` | Or `identity` if stored coordinates are camera coordinates |
| `position_coordinate_origin_x`, `position_coordinate_origin_y` | `100`, `50` | The origin subtracted from camera coordinates |
| `position_coordinate_scale` | `0.7097` | The scale applied |
| `position_missing_coordinate` | `1023` | The value meaning "not found" |
| `position_area_units` | `camera_pixels_saturated_uint16` | Units of `numpix1` and `numpix2` |

To get back to camera pixels: `camera_x = stored_x / scale + origin_x`.

### `.ttl`: digital events

There are no headers beyond the common ones (and the timing headers in Open Ephys files). The data is text, with no
count header. One line per change of a digital input:

```text
ttl_<line> <seconds> <state>
```

For example, `ttl_2 0.01 1` means that line 2, numbered from 1, went high 0.01 s into the recording.

Unlike other times in LTX, these are seconds, not ticks. Because data starts straight after the marker, the first event
shares a line with it, and the last line's `\r\n` is followed by the usual `\r\ndata_end`:

```text
data_startttl_2 0.01 1
ttl_2 0.02 0

data_end
```

## Spike sorting files

These hold one cluster label per spike, in the same order as the tetrode file. Neither has an LTX header.

**`.clu.N`** (from KlustaKwik's format, written by ltx-klusta and Waveform). The first line is the number of clusters,
then there is one label per line. Labels start at 1. ltx-klusta uses 1 for noise, and 2 upwards for clusters. Waveform
shows label `L` as cell `L - 1`, so noise is its cell 0.

**`.cut`** (written by Waveform). Labels are whitespace-separated, and start at 0. After a preamble, a line gives the
trial and spike count, and the labels follow it:

```text
n_clusters: 3
n_channels: 4
n_params: 0
times_used_in_Vt: 0 0 0 0
cluster: 0 center: 0 0 0 0 0 0 0 0
min: 0 0 0 0 0 0 0 0
max: 0 0 0 0 0 0 0 99
...

Exact_cut_for: 2026-11-03_a spikes: 125000
0 2 1 1 0 ...
```

Waveform needs only the `Exact_cut_for:` line, and checks that its spike count matches the tetrode. It accepts labels
up to 1023 in `.cut`, and up to 1024 in `.clu.N`.

## Timing and alignment

- Spike and position timestamps count ticks of a 96 kHz clock from the start of recording.
- Ticks are stored in a signed 32-bit integer, so they run out after about 6 h 12 min. Past that point, ephys-rs
  holds timestamps at the maximum and the Open Ephys plugin stops recording. The plugin also refuses position
  timestamps over 4 hours, because they reach it as 32-bit floats.
- Position timestamps count from the first position sample, not from the start of the spikes' clock.
- Waveform ignores position timestamps, and treats sample `i` as being at `i / sample_rate`.
- Each file's timeline starts from zero. Neither writer pads or shifts data to correct any offset between the streams.

## Reading LTX in Python

This reads each binary file type with NumPy. It expects a complete file with data in it.

```python
import re
import numpy as np

SPIKE = np.dtype([("t", ">i4"), ("samples", "i1", 50)])  # one channel of a spike
POSITION = np.dtype([("t", ">i4"), ("xy", ">u2", 4), ("numpix", ">u2", 2), ("padding", ">u2", 2)])


def read_ltx(path):
    """A file's header as a dict of strings, and the bytes following data_start."""
    raw = open(path, "rb").read()
    marker = re.search(rb"(?:^|\n)data_start", raw)
    text = raw[: marker.start()] if marker else raw
    header = {}
    for line in text.decode("latin-1").splitlines():
        key, _, value = line.strip().partition(" ")
        if key:
            header[key] = value.strip()
    return header, raw[marker.end() :] if marker else b""


def rate(value):
    """A value such as "96000 hz", as a number."""
    return float(value.split()[0])


def read_tetrode(path):
    """Spike times in seconds, and waveforms as spikes x 4 channels x 50 samples."""
    header, data = read_ltx(path)
    n = int(header["num_spikes"])
    spikes = np.frombuffer(data, SPIKE, count=4 * n).reshape(n, 4)
    return spikes["t"][:, 0] / rate(header["timebase"]), spikes["samples"]


def read_egf(path):
    """Samples as int8, and their sample rate."""
    header, data = read_ltx(path)
    samples = np.frombuffer(data, "i1", count=int(header["num_EEG_samples"]))
    return samples, rate(header["sample_rate"])


def read_pos(path):
    """Positions as samples x (x1, y1, x2, y2), with NaN where an LED is missing, and LED areas."""
    header, data = read_ltx(path)
    samples = np.frombuffer(data, POSITION, count=int(header["num_pos_samples"]))
    xy = samples["xy"].astype(float)
    xy[xy == 1023] = np.nan
    return xy, samples["numpix"]
```
