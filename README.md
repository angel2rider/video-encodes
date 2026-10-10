# video-encodes

Distributed video encoding on GitHub Actions: a matrix of runners each HTTP-seek
into a source file stored in an [HF Storage Bucket](https://huggingface.co/docs/hub/storage-buckets),
encode one segment, upload the part back to the bucket, and a final job concats
everything and muxes in the original audio track.

No split job, no artifact storage — bucket resolve URLs support HTTP range
requests, so each leg transfers only its own slice of the source.

## Setup

1. Create an HF access token with **write** access to the bucket
   (huggingface.co → Settings → Access Tokens).
2. Add it as a repository secret named `HF_TOKEN`
   (`gh secret set HF_TOKEN` or Settings → Secrets → Actions).
3. Actions → **bucket-encode** → **Run workflow**.

## Inputs

| Input | Default | Meaning |
| --- | --- | --- |
| `bucket` | `Angelrider/video-encodes` | Target HF bucket (`owner/name`) |
| `source_path` | American Psycho MKV | File path inside the bucket |
| `output_name` | `american-psycho-2000-encode.mkv` | Result written to bucket root |
| `segments` | `8` | Parallel encode legs (keep ≤ 18 on the Free plan) |
| `clip_seconds` | `60` | Test mode: encode only N s from each segment start. `0` = full segments |
| `encoder_args` | x265 CRF 24 10-bit | Applied identically to every leg (video only) |

## Pipeline

```
probe   ffprobe over HTTP -> duration -> segment matrix (JSON)
encode  ffmpeg -ss <start> -i <resolve-url> -t <dur> <encoder_args>  (x N legs)
        -> hf buckets cp parts/<run_id>/part_NNN.mkv
merge   hf buckets cp parts back -> ffmpeg -f concat -c copy
        + audio stream-copied from source URL -> upload result -> rm parts
```

## Caveats

- **Subtitles are dropped** (video-only parts keep stream layouts identical so
  concat is safe). Audio is stream-copied once at merge, so Atmos/EAC3 survives.
- **Dolby Vision RPU is lost** on re-encode; HDR10 signaling (PQ/BT.2020) is kept.
- Runners have **~14 GB disk**. Merge needs parts + final output to fit, so keep
  encoded results under ~5–6 GB or merge locally (the workflow prints the exact
  commands when it hits the disk guard).
- 4K x265 on a 4-vCPU runner is slow: size `segments` so one leg fits under the
  6-hour job cap. `clip_seconds: 60` exists to smoke-test the pipeline cheaply.
- `workflow_dispatch` only — never triggers on PRs, so the token can't leak to forks.
