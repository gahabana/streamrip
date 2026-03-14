# Replay Gain: API Data & Implementation Notes

## What Both Services Provide

Both Tidal and Qobuz provide replay gain values in their API responses.
Streamrip currently **ignores all of it** — no extraction, no tagging.

---

## Qobuz

### Track-level (available)

Found in the track metadata response (`track/get` endpoint) under `audio_info`:

```json
{
  "audio_info": {
    "replaygain_track_gain": -0.44,
    "replaygain_track_peak": 0.711731
  }
}
```

- `replaygain_track_gain` — in dB (e.g. `-0.44`)
- `replaygain_track_peak` — linear amplitude (e.g. `0.711731`)

This is present on every track, including tracks within an album response
(`album/get` → `tracks.items[N].audio_info`).

### Album-level (not available)

The album response (`album/get`) does **not** have an `audio_info` field.
Qobuz does not provide album-level replay gain.

### Where in streamrip code

- **API call**: `streamrip/client/qobuz.py` → `get_metadata()` (line ~243)
- **Track metadata extraction**: `streamrip/metadata/track.py` → `TrackMetadata.from_qobuz()` (line ~38)
- **Album metadata extraction**: `streamrip/metadata/album.py` → `AlbumMetadata.from_qobuz()` (line ~91)

Currently `audio_info` is not read in any of these.

---

## Tidal

### Track-level metadata (available)

Found in the track metadata response (`tracks/{id}` endpoint) at the top level:

```json
{
  "replayGain": -4.2,
  "peak": 0.710358
}
```

- `replayGain` — in dB
- `peak` — linear amplitude

### Playback info (available — track + album level)

Found in the playback info response (`tracks/{id}/playbackinfopostpaywall`):

```json
{
  "trackReplayGain": -2.9,
  "trackPeakAmplitude": 0.710358,
  "albumReplayGain": -4.2,
  "albumPeakAmplitude": 0.99063
}
```

This is the **only source of album-level replay gain** from either service.

### Where in streamrip code

- **Track metadata API call**: `streamrip/client/tidal.py` → `get_metadata()` (line ~134)
  - Response is logged at `logger.debug(item)` but `replayGain`/`peak` are not extracted.
- **Track metadata extraction**: `streamrip/metadata/track.py` → `TrackMetadata.from_tidal()` (line ~156)
  - Does not extract `replayGain` or `peak` from the response.
- **Playback info API call**: `streamrip/client/tidal.py` → `get_downloadable()` (line ~218)
  - Response is logged at `logger.debug(resp)` but replay gain fields are not extracted.
  - This method returns a `Downloadable`, not metadata — so passing replay gain
    back to the tagging pipeline requires either:
    - Storing it on the `Downloadable` object, or
    - Making a separate metadata pass, or
    - Extracting from the track-level metadata in `from_tidal()` instead (simpler,
      but loses album-level values).

### Implementation consideration

The track-level `replayGain`/`peak` in the metadata response and the
`trackReplayGain`/`trackPeakAmplitude` in the playback info response
may differ slightly — the playback info values are tied to the specific
quality/encoding being streamed, while the metadata values are generic.

For tagging purposes, the track metadata values (`replayGain`, `peak`) are
simpler to extract since they're already in the `track` dict passed to
`TrackMetadata.from_tidal()`. The album-level values require changes to
`get_downloadable()` to pass them through.

---

## FLAC ReplayGain Tags (standard Vorbis comments)

The standard tags that should be written to FLAC files:

| Tag | Format | Example | Source |
|-----|--------|---------|--------|
| `REPLAYGAIN_TRACK_GAIN` | `{value} dB` | `-0.44 dB` | Both |
| `REPLAYGAIN_TRACK_PEAK` | `{value}` | `0.711731` | Both |
| `REPLAYGAIN_ALBUM_GAIN` | `{value} dB` | `-4.20 dB` | Tidal only |
| `REPLAYGAIN_ALBUM_PEAK` | `{value}` | `0.990630` | Tidal only |

Note: gain values must include the ` dB` suffix. Peak values are linear
amplitude (0.0–1.0), no suffix.

---

## Implementation Steps (for future reference)

1. **Add fields to `TrackMetadata`** (or `TrackInfo`) in `streamrip/metadata/track.py`:
   - `replay_gain: float | None`
   - `peak: float | None`
   - `album_replay_gain: float | None` (Tidal only)
   - `album_peak: float | None` (Tidal only)

2. **Extract in `from_qobuz()`** (`streamrip/metadata/track.py`):
   ```python
   audio_info = safe_get(resp, "audio_info", default={})
   replay_gain = audio_info.get("replaygain_track_gain")
   peak = audio_info.get("replaygain_track_peak")
   ```

3. **Extract in `from_tidal()`** (`streamrip/metadata/track.py`):
   ```python
   replay_gain = track.get("replayGain")
   peak = track.get("peak")
   ```
   For album-level values from playback info, either:
   - Store on `TidalDownloadable` / `TidalDashDownloadable` and pass back, or
   - Accept that album-level values won't be tagged (simpler).

4. **Add to tagger** (`streamrip/metadata/tagger.py`):
   - Add custom handling in the FLAC tagging path (these don't fit the
     existing `METADATA_TYPES` pattern since they need formatting):
   ```python
   if meta.replay_gain is not None:
       audio["REPLAYGAIN_TRACK_GAIN"] = f"{meta.replay_gain:.2f} dB"
       audio["REPLAYGAIN_TRACK_PEAK"] = f"{meta.peak:.6f}"
   if meta.album_replay_gain is not None:
       audio["REPLAYGAIN_ALBUM_GAIN"] = f"{meta.album_replay_gain:.2f} dB"
       audio["REPLAYGAIN_ALBUM_PEAK"] = f"{meta.album_peak:.6f}"
   ```

5. **Verified with real API data** (March 2026):
   - Qobuz track 47683562: `replaygain_track_gain=-0.44, replaygain_track_peak=0.714142`
   - Tidal track 479084078: `replayGain=-4.2, peak=0.710358` (metadata),
     `trackReplayGain=-2.9, trackPeakAmplitude=0.710358, albumReplayGain=-4.2, albumPeakAmplitude=0.99063` (playback info)
