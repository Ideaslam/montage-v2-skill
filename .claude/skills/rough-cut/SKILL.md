---
name: rough-cut
description: Turn a folder of raw video footage into a finished, social-ready video with an editable Adobe Premiere Pro timeline. Part 1 analyzes the clips (transcript, snapshots, key scenes), proposes an edit, writes a timeline CSV and a Premiere-importable XML, and cleans the audio. Part 2 adds effects (styled subtitles, B-roll, transitions, punch-in zooms with sound effects, color correction and skin smoothing). Part 3 renders a high-quality export. Every edit and the export need the user's approval. Use when the user asks to analyze footage, build a montage or rough cut, edit a talking-head/social video, or prepare a Premiere timeline from a folder of videos.
argument-hint: <path-to-footage-folder>
---

# rough-cut: footage → edited, social-ready video

The plain-language version of this flow is `docs/m-video/note.md`. If the user changes the note, update this skill to match it.

Footage folder: `$ARGUMENTS`. If it is empty, ask the user for the folder path before doing anything else.

## Ground rules

- **✋ = approval gate:** stop and wait for the user's explicit OK. The user's rule: *always take my approval before any edits.*
- Never modify, overwrite or delete the user's original media. All outputs go in `<footage-folder>/_rough-cut/`.
- **Limits:** you cannot see video or hear audio directly. You "watch" through extracted frames (open them with the Read tool) and "listen" through a transcript. You cannot control Premiere Pro. You produce files that the user imports.
- **What Premiere can and can't import:** a Final Cut Pro 7 XML carries cuts, cross-dissolve transitions, scale/position keyframes (zooms), and extra audio tracks (SFX, clean audio) into Premiere. It does **not** carry Lumetri color or styled/animated captions. Those are built with ffmpeg and burned into the final render (Part 3). Tell the user this once, at the start of Part 2.
- After each step, report in 1–3 lines what you made and where.

---

# PART 1: Rough cut

## 1. Check tools
- `which ffmpeg ffprobe`, plus a transcriber (`whisper`, `whisper-cli`, or `mlx_whisper`).
- Premiere installed? macOS: `ls -d /Applications/Adobe\ Premiere\ Pro*/Adobe\ Premiere\ Pro*.app 2>/dev/null`. Windows: `dir "C:\Program Files\Adobe\Adobe Premiere Pro*"`. Running? `pgrep -fl "Adobe Premiere Pro"` (Windows: `tasklist | findstr /i "Premiere"`).
- If ffmpeg or Whisper is missing, propose `brew install ffmpeg` / `pip install -U openai-whisper`. ✋ Ask before installing.
- If Premiere is missing, say so. Everything still works except opening the XML (DaVinci Resolve can also import it). Ask whether to continue.

## 2. Read the videos
List the media (`.mp4 .mov .mxf .m4v .wav .mp3 .m4a`, any case). For each file:
`ffprobe -v error -show_entries format=duration:stream=codec_type,codec_name,width,height,r_frame_rate,sample_rate,channels -of json "<file>"`
Save the results to `_rough-cut/media_info.json`. The main frame rate and resolution define the sequence.

## 3. Extract sound and snapshots
- Audio for transcription: `ffmpeg -i "<file>" -vn -ac 1 -ar 16000 "_rough-cut/audio/<name>.wav"`
- Full-quality audio: `ffmpeg -i "<file>" -vn -c:a pcm_s24le "_rough-cut/audio/<name>_full.wav"`
- Snapshots (N = 5 s for clips under 5 min, otherwise 15 s, plus scene changes):
  `ffmpeg -i "<file>" -vf "select='gt(scene,0.35)+not(mod(t\,N))',scale=640:-2" -vsync vfr -frame_pts 1 "_rough-cut/frames/<name>_%06d.jpg"`
  Keep each frame's timestamp.

## 4. Understand the videos
- Transcribe with word-level timestamps; the subtitles in Part 2 need them: `whisper "<wav>" --model small --word_timestamps True --output_format all --output_dir _rough-cut/transcripts` (set `--language` if known).
- Read a sample of about 20–40 snapshots per clip.
- For each clip, find: the topic, the speakers, the key moments, strong quotes, bad takes, silences, and B-roll candidates.

## 5. Summary and proposed edit
Write `_rough-cut/summary.md`: an overview, the main points, the key scenes and quotes with timecodes, the metadata, and a **proposed edit** (ordered segments and why).
✋ Show it and wait for approval or changes.

## 6. Timeline CSV
Write `_rough-cut/timeline.csv` in the approved order:
`order,clip_file,source_in,source_out,record_in,record_out,duration,track,type,description,dialogue,notes`
Timecodes are `HH:MM:SS:FF` at the sequence fps. `type` is `a-roll | b-roll | music | title | sfx`.

## 7. Premiere timeline
- Write `_rough-cut/sequence.xml` (FCP7 XML, `xmeml version="4"`) from the CSV: one `<sequence>` with the fps and resolution from step 2 (`<ntsc>TRUE</ntsc>` for 23.976/29.97/59.94), one `<clipitem>` per row with `<in>/<out>` (source frames) and `<start>/<end>` (sequence frames), and linked audio. Each `<file>` has `<pathurl>` = `file://` + the absolute, URL-encoded path.
- Check that it is well-formed: `python3 -c "import xml.dom.minidom,sys;xml.dom.minidom.parse(sys.argv[1])" _rough-cut/sequence.xml`
- If Premiere is not running: ✋ offer to open it (`open -a "<app path>"`), and wait until the user has a project open.
- User: *File → Import → sequence.xml*. Ask them to confirm that nothing is offline (if something is, they use *Link Media*).

## 8. Clean the sound
✋ Ask first, and say which method you will use:
- **Adobe Podcast Enhance Speech (best):** use an Adobe connector speech-enhance tool (e.g. `media_enhance_speech`) if one is available. This uploads audio to Adobe, so confirm first. Otherwise, the user processes the `_full.wav` files at https://podcast.adobe.com/enhance.
- **Local ffmpeg:** `-af "highpass=f=80,afftdn=nf=-25,loudnorm=I=-16:TP=-1.5:LRA=11"`

Save the results to `_rough-cut/audio_clean/<name>_clean.wav`. Update the XML so that it uses the clean audio (keep the original muted on a lower track).

## 9. Approve the cut
Recap: clip count, total length, the audio method, and the file paths.
If the user changed the edit in Premiere, ask them to export *File → Export → Final Cut Pro XML*, then update `timeline.csv` from it. Part 2 must start from their latest cut.
✋ Wait for "the cut is OK" before Part 2.

---

# PART 2: Effects

For every effect: propose the plan first (what, where, timecodes), then ✋ wait for approval. Where it helps, show a preview (a still frame, or a 3–5 s test clip rendered to `_rough-cut/previews/`). Record every effect in `_rough-cut/effects.csv`:
`order,effect,start,end,settings,sfx_file,notes`

**Sound effects:** ask the user for a folder of SFX (whoosh, pop, swoosh…). If they don't have one, offer to find some with a sound-effect connector if available, or skip the SFX. Place each SFX on its own audio track (A3+) in the XML, at about -12 dB under the voice.

## 10. Subtitles
✋ First ask the user to choose (offer 2–3 ready-made styles plus "custom"):
- **Style:** e.g. word-by-word highlight (social style), 2-line classic, or a big single word
- **Font** (it must be installed: `fc-list | grep -i "<font>"`), **size**, **text color**, **highlight color**, **outline/shadow**
- **Position:** bottom, centre, or upper third (keep it clear of the platform UI on 9:16)
- **Motion:** pop-in, fade, slide, or none
- **SFX on subtitles:** none, on every line, or only on key words

Build the subtitles from the word timestamps (step 4), adjusted to the cut's `record_in` times. Write:
- `_rough-cut/subtitles/subs.ass`: the styled and animated version for the render (ASS `\k` karaoke tags for highlighting, `\fad` / `\t(\fscx\fscy)` for pop motion)
- `_rough-cut/subtitles/subs.srt`: a plain version the user can import in Premiere (*File → Import*) to edit the text

Render a 5 s preview. ✋ Wait for approval. Also ask the user to check the spelling, especially names.

## 11. B-roll
Only where it helps (long talking stretches, abstract topics, covering a jump cut). Use B-roll candidates from the footage first. If something is missing, list the shots you suggest and ask the user for files. Don't invent or generate footage without asking.
✋ Show the list (timecode, which B-roll, why). Once approved, add the B-roll on V2 in the XML and the CSV.

## 12. Transitions
Use them only between some clips (scene or topic changes), not on every cut. Use cross dissolve or dip-to-black, 8–15 frames. Put them in the XML as `<transitionitem>`; for the render use ffmpeg `xfade`.
✋ Show the list of transition points.

## 13. Zoom in / zoom out
On important moments or important lines (from the step 4 analysis): a punch-in to 110–120 % scale, either quick (3–6 frames, with a whoosh SFX) or a slow push. Zoom back out at the end of the moment. Put them in the XML as Basic Motion scale keyframes; for the render use ffmpeg `zoompan` / `scale+crop` on that segment.
✋ Show the list (timecode, the line said, zoom type, SFX).

## 14. Color
✋ Show a before/after still for each clip.
- **Correction:** exposure, contrast, white balance, and matching between clips. ffmpeg: `eq=brightness:contrast:saturation:gamma`, `colorbalance`, `colortemperature`, or a `.cube` LUT if the user has one (`lut3d`).
- **Skin:** natural skin tones (don't push orange or red), plus gentle smoothing: `bilateral=sigmaS=4:sigmaR=0.08` or `hqdn3d=2:1:3:3` at low strength. Note: ffmpeg smooths the whole frame. For skin-only smoothing, tell the user to use Premiere Lumetri (HSL Secondary) or a beauty plugin, and offer to write the exact settings.

---

# PART 3: Render

## 15. Final export
✋ **No export until the user says yes.** First ask:
- Which platform? **9:16** (Reels/TikTok/Shorts), **16:9** (YouTube), **1:1 or 4:5** (feed). For a crop, keep the speaker's face centred (check against the snapshots).
- Export it themselves from Premiere, or have Claude render with ffmpeg (which includes the burned-in subtitles and color)?

**ffmpeg render settings (high quality, high bitrate, social-ready):**
- Video: `-c:v libx264 -profile:v high -preset slow -crf 16 -maxrate 50M -bufsize 100M -pix_fmt yuv420p`, at the source fps (no conversion). For 4K, use `-maxrate 80M`.
- Audio: `-c:a aac -b:a 320k -ar 48000`, loudness normalized to **-14 LUFS** (`loudnorm=I=-14:TP=-1:LRA=11`).
- `-movflags +faststart`
- Output: `_rough-cut/export/final_<ratio>.mp4`

If the user exports from Premiere, give them the matching settings: H.264, High profile, VBR 2-pass, target 30–40 Mbps (1080p), AAC 320 kbps, 48 kHz.

After the render, check it with ffprobe (duration, resolution, bitrate, audio present) and report the file path and size.

## 16. Update the CSV
If the user changed anything in Premiere: they export *File → Export → Final Cut Pro XML*, and you update `timeline.csv` and `effects.csv` to match.
Write `_rough-cut/changelog.md`: what changed compared with the proposed edit and effects.

---

## Output files (`_rough-cut/`)
| File | What it is |
|---|---|
| `summary.md` | the video overview + proposed edit |
| `timeline.csv` | the clips in the edit |
| `effects.csv` | every effect with its timecode and settings |
| `sequence.xml` | the timeline for Premiere |
| `audio_clean/` | the cleaned sound |
| `subtitles/` | `subs.ass` (styled) + `subs.srt` (editable) |
| `previews/` | test clips and stills for approvals |
| `export/final_<ratio>.mp4` | the final video |
| `changelog.md` | what changed after the user's edits |
