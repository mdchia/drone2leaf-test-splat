# Drone2Leaf - greenhouse bay Gaussian splat

Live viewer: **https://mdchia.github.io/drone2leaf-test-splat/**

An interactive 3D Gaussian splat of a greenhouse bay, reconstructed from drone
RGB imagery. Drag to orbit, scroll to zoom, right-drag to pan.

![preview](site/preview.jpg)

## The model

| | |
|---|---|
| source | 353 RGB frames, DJI Mavic 3M, 5280x3956 |
| capture | ten passes, three gimbal pitches (-30/-45/-90 deg), two altitudes (2.4 / 3.4 m), 52 minutes |
| reconstruction | COLMAP 4.1.1, DSP-SIFT + exhaustive matching, 353/353 images in a single model, 443,361 points |
| training | nerfstudio `splatfacto`, 60,000 iterations at 1600 px |
| published | 955,913 gaussians, 29.2 MB |

Held-out evaluation: **cc_psnr 16.18 dB, cc_ssim 0.434** over 45 unseen views.

## What it looks like, and why

Rigid structure - floor, rails, glazing, benches - is sharp. Foliage is soft,
and that is a property of the capture rather than the pipeline: **67.6% of the
reconstructed points were observed by only one of the ten flight passes**, so
most leaf surfaces have no multi-view constraint. Gaussian splatting fits one
set of gaussians to every view at once, so where passes disagree it averages.

Doubling the model from 1.19M to 2.23M gaussians left sharpness unchanged (0.15
to 0.16 of ground-truth high-frequency energy), which confirms capacity was not
the limit. Future captures should prioritise overlap - every surface in three or
more passes - over adding distinct pass types.

## Repository layout

```
site/                   everything published to GitHub Pages
  index.html            the viewer (no build step, no framework)
  gsplat-1.2.3.js       vendored renderer, so the page makes no third-party requests
  all_runs.splat        the model, antimatter15 format, 32 bytes per gaussian
  preview.jpg           link-preview image, cropped from an actual render
.github/workflows/      Pages deployment
```

## Notes for future edits

- **Do not put the `.splat` in Git LFS.** GitHub Pages does not resolve LFS
  objects; it would serve the pointer file as text and the viewer would fail on
  130 bytes of ASCII. The deploy workflow checks for this explicitly.

- **Do not switch back to `SPLAT.Loader.LoadAsync`.** When a response carries a
  `content-length`, gsplat 1.2.3 preallocates `new Uint8Array(content-length)`
  and streams the body into it. Pages serves this file with
  `content-encoding: gzip` to anything that advertises it, so `content-length`
  is the *compressed* size (29,273,441) while the reader yields *decompressed*
  bytes (30,589,216) - the array overflows and `set()` throws
  `RangeError: source array is too long`. The page fetches and assembles the
  buffer itself, then calls `LoadFromArrayBuffer`, which is correct under any
  transfer encoding.

  The same trap invalidates any size or gaussian count derived from
  `content-length`: it would report 914,795 gaussians instead of 955,913.

  Worth knowing when verifying a deployment: **`curl` does not send
  `Accept-Encoding` by default**, so it receives the file uncompressed and every
  check passes while browsers still fail. Test with
  `curl -H 'Accept-Encoding: gzip' -I` to see what a browser actually gets.
- The viewer prefers `all_runs_lod.splat` if one is ever added and offers a
  quality toggle; with only the full model present it loads that directly.
- Cache-busting keys off the file's content length, so replacing the `.splat`
  invalidates automatically without editing any markup.
- `.nojekyll` is present so nothing gets filtered if the Pages source is ever
  switched from Actions to branch deployment.

Colours were verified against nerfstudio's own RGB export to within 1/255 per
channel before publishing.
