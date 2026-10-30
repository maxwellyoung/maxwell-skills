# Command templates

Starting points, not settings to copy blindly. Check each flag against the installed build (`ffmpeg -h encoder=libx264`, `ffmpeg -h filter=libvmaf`, `ffmpeg -h muxer=mp4`) and read back every output with `ffprobe`. `H`, `CRF` and the colour values come from the per-file config.

## Probe

```sh
# Streams, colour, rotation, container
ffprobe -v error -show_entries \
  stream=index,codec_type,codec_name,codec_tag_string,profile,level,pix_fmt,width,height,sample_aspect_ratio,display_aspect_ratio,r_frame_rate,avg_frame_rate,bit_rate,color_range,color_space,color_transfer,color_primaries,channels,channel_layout,sample_rate:stream_side_data=rotation:format=duration,size,bit_rate \
  -of json in.mp4

# Exact frame count (decodes the stream)
ffprobe -v error -count_frames -select_streams v:0 -show_entries stream=nb_read_frames -of csv=p=0 in.mp4

# Keyframe timestamps
ffprobe -v error -select_streams v:0 -skip_frame nokey -show_entries frame=pts_time -of csv=p=0 in.mp4

# moov before mdat? The first of the two printed is first in the file
ffprobe -v trace in.mp4 2>&1 | grep -oE "type:'(moov|mdat)'" | head -2

# Chapters
ffprobe -v error -show_chapters -of json in.mp4

# Loudness and true peak
ffmpeg -hide_banner -i in.mp4 -af ebur128=peak=true -f null - 2>&1 | grep -E 'I:|LRA:|Peak:'
```

## Encode

```sh
# H.264 baseline, SDR BT.709 limited range, no audio (only for never-audible loops)
ffmpeg -i master.mov -map 0:v:0 \
  -vf "scale=-2:H:flags=lanczos,format=yuv420p,setparams=color_primaries=bt709:color_trc=bt709:colorspace=bt709:range=tv" \
  -c:v libx264 -preset slow -crf CRF -profile:v high \
  -g 60 -keyint_min 60 -sc_threshold 0 \
  -an -map_metadata -1 -fflags +bitexact -flags:v +bitexact \
  -movflags +faststart out.mp4

# Keep audio instead: replace -an with
  -map 0:a:0 -c:a aac -b:a 128k

# AV1 (SVT-AV1); -preset is 0 (slowest) to 13
  -c:v libsvtav1 -preset 4 -crf CRF -pix_fmt yuv420p

# VP9 in WebM, constant quality needs -b:v 0; index at the front
  -c:v libvpx-vp9 -crf CRF -b:v 0 -row-mt 1 -c:a libopus -b:a 96k -cues_to_front 1 out.webm

# VP9 with alpha (Chrome, Firefox)
  -c:v libvpx-vp9 -pix_fmt yuva420p -crf CRF -b:v 0 out-alpha.webm

# HEVC for Safari: tag hvc1, not hev1
  -c:v libx265 -crf CRF -tag:v hvc1

# HEVC with alpha for Safari (macOS VideoToolbox only)
  -c:v hevc_videotoolbox -alpha_quality 0.75 -tag:v hvc1

# Two-pass to a byte budget (x264)
ffmpeg -y -i master.mov -c:v libx264 -b:v RATE -pass 1 -an -f null /dev/null
ffmpeg -i master.mov -c:v libx264 -b:v RATE -pass 2 ... out.mp4

# Poster from the encode's first frame
ffmpeg -i out.mp4 -frames:v 1 poster.png
```

Colour: the generic `-color_primaries`/`-color_trc` output options have been seen not to write tags for untagged input, so tag with `setparams` (as above) or `-x264-params colorprim=bt709:transfer=bt709:colormatrix=bt709`, then confirm with `ffprobe`. If the source is untagged, establish what it really is (camera, export settings, how browsers render it today) before choosing tags; tagging changes how browsers convert it.

## Compare

```sh
# VMAF at the rendered size: first input is the encode, second the original.
# Scale both identically and reset timestamps so frames align.
# Give both the same colour tags (add the same setparams=... to each chain) or ffmpeg
# auto-converts one input and the score collapses. Check with -loglevel verbose for
# "auto-inserting filter 'auto_scale" before trusting a result.
ffmpeg -i out.mp4 -i master.mov -lavfi \
  "[0:v]scale=-2:H:flags=bicubic,setpts=PTS-STARTPTS[d];[1:v]scale=-2:H:flags=bicubic,setpts=PTS-STARTPTS[r];[d][r]libvmaf=model=version=vmaf_v0.6.1:n_threads=8:log_fmt=json:log_path=vmaf.json" \
  -f null -
# vmaf.json has per-frame scores and pooled_metrics.vmaf.{min,mean,harmonic_mean}

# SSIM (same alignment)
ffmpeg -i out.mp4 -i master.mov -lavfi "[0:v]scale=-2:H[d];[1:v]scale=-2:H[r];[d][r]ssim" -f null -

# Frame grab at a fixed time, same command for original and encode
ffmpeg -ss 12.0 -i out.mp4 -frames:v 1 -vf scale=-2:H grab-out-12.png

# Dark scenes worth grabbing
ffmpeg -i master.mov -vf blackdetect=d=0.1:pix_th=0.1 -an -f null - 2>&1 | grep black_start

# Range support (expect 206 and 100 bytes)
curl -sS -r 0-99 -o /dev/null -w '%{http_code} %{size_download} %{content_type}\n' https://example.test/media/out.mp4
```
