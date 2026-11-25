# Rough-cut flow (simple version)

This is the plan for the `rough-cut` skill, in plain words.
To change the flow: edit this file, then ask Claude to update the skill to match.

How to start it: `/rough-cut <my footage folder>`

---

## Rules (always)

- ✋ = Claude **stops and waits for my OK** before going on.
- Claude **never changes my original videos**.
- Everything Claude makes goes into one folder: `<my footage folder>/_rough-cut/`

---

## The flow

### 1. Check tools
Claude checks that these are on my computer:
- **ffmpeg**: cuts video and pulls out the sound and pictures
- **Whisper**: turns speech into text
- **Premiere Pro**: is it installed? is it open?

✋ If something is missing, Claude asks before installing it.

### 2. Read my videos
Claude lists the videos in my folder and reads their details (length, size, frame rate).

### 3. Pull out the sound and the pictures
- Sound → saved as audio files
- Snapshots → a picture every few seconds, plus one at each scene change

### 4. Understand the videos
- Whisper writes down what people say (with times)
- Claude looks at the snapshots
- Claude finds: the main topic, the best moments, good quotes, bad takes, silences

### 5. Write a summary + suggest an edit
Claude writes `summary.md`:
- what the video is about
- the main points
- the key scenes and quotes (with times)
- **suggested edit**: which parts to keep, in what order, and why

✋ I read it and say OK, or ask for changes.

### 6. Make the timeline CSV
Claude writes `timeline.csv`: one row per clip in the edit (which file, start time, end time, description).

### 7. Make the Premiere timeline
- Claude writes `sequence.xml` from the CSV
- Claude checks that Premiere is open. ✋ If it isn't, Claude asks before opening it
- **I do:** Premiere → File → Import → `sequence.xml`
- I tell Claude whether everything loaded (no missing clips)

### 8. Clean the sound
✋ Claude asks first, then uses one of these:
- **Adobe Podcast Enhance** (best; the sound is uploaded to Adobe)
- **ffmpeg noise filter** (works on my computer, no upload)

Then Claude updates the timeline to use the clean sound.

### 9. Final check, then export
Claude tells me: how many clips, total length, where the files are.

✋ **No export until I say yes.**

I pick how to export:
- I export it myself from Premiere, **or**
- Claude makes `final.mp4` with ffmpeg

### 10. Update the CSV
If I changed the edit in Premiere:
- **I do:** Premiere → File → Export → Final Cut Pro XML
- Claude reads it and updates `timeline.csv` to match my final edit
- Claude writes `changelog.md`: what changed from the suggested edit

---

## What I get at the end (inside `_rough-cut/`)

| File | What it is |
|---|---|
| `summary.md` | what the video is about + the suggested edit |
| `timeline.csv` | list of clips in the edit |
| `sequence.xml` | the timeline to open in Premiere |
| `audio_clean/` | the cleaned sound |
| `export/final.mp4` | the final video (only if Claude exports) |
| `changelog.md` | what changed after my edits |



Step 2 : 

1- Effects : 
    - Subtitle: 
       - take my decision in the style , font , color ,motion with sound effect .
    
   
  - BRoll : 
    - do BRoll if it need it . 
  - Transitions : 
    - do trasnsitions between some clips .
   - Zoom :
     - Zoom in and Zoom out for important moment or important talk  with sound effect .
 - Coloring : 
   - correction . 
   - skin tone smoothness and coloring .

Step 3: 
    - Render : 
          - High Quality . 
          - High bit rate. 
          - good for social media . 