---
name: m-video
description: Turn a folder of raw video footage into an editable Adobe Premiere Pro rough cut. Analyzes the clips (transcript, snapshots, key scenes), writes a timeline CSV and a structured summary, generates a Premiere-importable XML sequence, cleans the audio (Adobe Podcast Enhance Speech or an ffmpeg fallback), and exports only after the user approves. Use when the user asks to analyze footage, build a montage or rough cut, or prepare a Premiere timeline from a folder of videos.
argument-hint: <path-to-footage-folder>
---

# m-video: footage → Premiere rough cut

Footage folder: `$ARGUMENTS`. If it is empty, ask the user for the folder path before doing anything else.

## Ground rules

- **Approval gates:** stop and wait for the user's explicit "yes" at every step marked **[APPROVAL]**. Never modify, overwrite or delete the user's original media. All outputs go in `<footage-folder>/_m-video/`.
- **Limits:** you cannot see video or hear audio directly. You "watch" through extracted frames (open them with the Read tool) and "listen" through a transcript. You cannot open or control Premiere Pro. You produce files the user imports.
- **Progress:** after each step, report in 1–3 lines what you produced and where.

## Step 0: Check tools

Run `which ffmpeg ffprobe` and check for a transcriber (`whisper`, `whisper-cli` / whisper.cpp, or `mlx_whisper`).
If any are missing, tell the user what is missing and propose the install command (macOS: `brew install ffmpeg`, `pip install -U openai-whisper`). **[APPROVAL]** Ask before installing anything.

**Check that Premiere Pro is ready:**
- Installed? macOS: `ls -d /Applications/Adobe\ Premiere\ Pro*/Adobe\ Premiere\ Pro*.app 2>/dev/null`. Windows: `dir "C:\Program Files\Adobe\Adobe Premiere Pro*"`. Report the version found (taken from the folder name, e.g. `Adobe Premiere Pro 2025`).
- Running? macOS: `pgrep -fl "Adobe Premiere Pro"`. Windows: `tasklist | findstr /i "Adobe Premiere Pro.exe"`.
- If it is not installed, tell the user. The analysis, CSV, summary and XML steps still work, but they will need Premiere (or another editor that imports FCP7 XML, e.g. DaVinci Resolve) to open the timeline. Ask whether to continue.
- If it is installed but not running, say so; it does not need to be open until Step 6.

## Step 1: Read the resources

List the media in the folder (`.mp4 .mov .mxf .m4v .wav .mp3 .m4a`, case-insensitive). For each file, run:

```
ffprobe -v error -show_entries format=duration:stream=codec_type,codec_name,width,height,r_frame_rate,sample_rate,channels -of json "<file>"
```

Save the results to `_m-video/media_info.json`. Note the dominant frame rate and resolution: the sequence will use them.

## Step 2: Extract sound and snapshots

For each video:
- Audio: `ffmpeg -i "<file>" -vn -ac 1 -ar 16000 "_m-video/audio/<name>.wav"` (16 kHz mono for transcription). Keep a full-quality copy too: `-vn -c:a pcm_s24le "_m-video/audio/<name>_full.wav"`.
- Snapshots: one frame every N seconds (N = 5 for clips under 5 min, otherwise 15) plus scene changes:
  `ffmpeg -i "<file>" -vf "select='gt(scene,0.35)+not(mod(t\,N))',scale=640:-2" -vsync vfr -frame_pts 1 "_m-video/frames/<name>_%06d.jpg"`
  Record each frame's timestamp, which you need for the CSV.

## Step 3: Transcribe and analyze

- Transcribe each audio file with timestamps (e.g. `whisper "<wav>" --model small --output_format srt --output_dir _m-video/transcripts`). Use the language the user speaks if it is known.
- Open a representative set of snapshots with Read (not every frame: sample about 20–40 per clip).
- From the frames and transcripts, identify for each clip: what it shows, the speakers, the key moments, strong quotes, bad takes or silences, and the B-roll candidates.

## Step 4: Summarize

Write `_m-video/summary.md` with:
- A one-paragraph overview of what the footage is about
- The main points and insights
- Key scenes (clip, in/out timecode, description)
- Key dialogue/quotes with timecodes
- Metadata (files, durations, fps, resolution, audio format)
- **A proposed edit**: the ordered list of segments for the rough cut and why

Show the user the summary and the proposed edit. **[APPROVAL]** Wait for approval or changes before building the timeline.

## Step 5: Build the timeline CSV

Write `_m-video/timeline.csv` with one row per segment in the approved order:

```
order,clip_file,source_in,source_out,record_in,record_out,duration,track,type,description,dialogue,notes
```

- Timecodes are `HH:MM:SS:FF` at the sequence frame rate. `record_in`/`record_out` are positions on the sequence.
- `type` is one of `a-roll`, `b-roll`, `music`, `title`.

## Step 6: Generate the Premiere timeline

Write `_m-video/sequence.xml` in **Final Cut Pro 7 XML (xmeml v4)** format, which Premiere imports natively, built from `timeline.csv`:
- One `<sequence>` with the frame rate and resolution from Step 1 (`<timebase>`, `<ntsc>TRUE</ntsc>` for 23.976/29.97/59.94).
- One `<clipitem>` per row on V1 (and the linked A1/A2 audio), with `<in>`/`<out>` (source frames) and `<start>`/`<end>` (sequence frames).
- Each `<file>` has a `<pathurl>` of `file://` plus the **absolute, URL-encoded** path to the original media, and declares its own duration/rate so Premiere can relink.
- Validate that the XML is well-formed (`python3 -c "import xml.dom.minidom,sys;xml.dom.minidom.parse(sys.argv[1])" _m-video/sequence.xml`).

Before handing off, check again that Premiere is running (`pgrep -fl "Adobe Premiere Pro"`). If it is not, **[APPROVAL]** offer to launch it (macOS: `open -a "<app path found in Step 0>"`) and wait for the user to confirm that it is open and a project is loaded (Premiere imports into the open project).

Tell the user: *Premiere Pro → File → Import → `sequence.xml`*. The sequence appears in the Project panel, ready to edit. Ask them to confirm that the import worked and that no clips show as offline (if any do, use *Link Media* and point to the original folder).

## Step 7: Clean the audio

**[APPROVAL]** Ask before processing audio, and say which method you will use:
1. **Adobe Podcast Enhance Speech (preferred):** if an Adobe connector tool for speech enhancement (e.g. `media_enhance_speech`) is available, use it on the `_full.wav` files. This uploads audio to Adobe, so confirm with the user first. Otherwise, tell the user to run the files through https://podcast.adobe.com/enhance manually and put the results in `_m-video/audio_clean/`.
2. **Local fallback (ffmpeg):** `ffmpeg -i in.wav -af "highpass=f=80,afftdn=nf=-25,loudnorm=I=-16:TP=-1.5:LRA=11" out.wav`

Save the cleaned files as `_m-video/audio_clean/<name>_clean.wav`. Regenerate `sequence.xml` so the audio tracks point to the cleaned files (keep the original audio muted on a lower track for reference).

## Step 8: Review before export

Give a short recap: segment count, total duration, the audio method used, and the file locations. **[APPROVAL]** Do not export until the user explicitly approves.

Export options (ask which one the user wants):
- **Premiere (recommended if the user edited there):** the user exports from Premiere themselves.
- **ffmpeg render** of the approved CSV: trim each segment, concatenate, and mux in the cleaned audio, to `_m-video/export/final.mp4` (H.264 at the source resolution/fps, AAC 320k).

## Step 9: Update the CSV after the final export

If the user edited in Premiere, ask them to export the final sequence as XML (*File → Export → Final Cut Pro XML*) and give you the path. Parse it and update `timeline.csv` to match the final cut (changed in/out points, order, removed or added clips). Write a short `_m-video/changelog.md` listing what changed from the proposed edit.
