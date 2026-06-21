# 🎧 spotdl-to-iphone

With all the music in the world instantly available on streaming services, music has started to feel a bit like McDonald's for me. I miss having a library that felt like my own curated collection instead of the work of an algorithm. With the resurgence of older—offline—tech like the iPod, or even wired earphones, it's clear to me that others also want to use tech more intentionally.

Thus, I stiched together this tool that uses [spotDL](https://github.com/spotDL/spotify-downloader)—which handles the heavy lifting of finding tracks, downloading audio, and embedding clean metadata (e.g., album art, artist info; instead of a plain `.mp3` file)—and just automates the final step of getting those files into the Apple Music ecosystem and onto an iPhone.

> 📣 Please remember that if you enjoy an artist's work, **support them directly** where you can—buy their album, go to a show, or get their merch. This tool is for appreciating music you already care about, not replacing the support artists deserve...
 
## ⚙️ Setup
 
### Step 0: Enable Auto-Sync on Your iPhone
 
1. Connect your iPhone to your Mac via USB,
2. Open **Finder** on your Mac,
3. Select your iPhone from the sidebar under **Locations**,
4. Open the **General** tab and, under **Options**, enable **Automatically sync when this iPhone is connected**,
5. Open the **Music** tab and enable **Sync music onto [your iPhone]**,
6. Select **Sync: Entire music library**.

After this, any time your iPhone is connected and the Music app is open, newly added tracks will sync automatically!
 
---
 
### Step 1: Install Python
 
Install Python from the official website:
 
https://www.python.org/downloads/
 
After installing, open **Terminal** and verify:
 
```bash
python3 --version
pip3 --version
```
 
If these commands don't work, Python wasn't added to PATH correctly during installation.
 
---
 
### Step 2: Install Homebrew and FFmpeg

spotDL requires FFmpeg for audio processing. The easiest way to install it is via Homebrew.

Install HomeBrew from the official website:
 
https://brew.sh
 
Then install FFmpeg:
 
```bash
brew install ffmpeg
```
 
---
 
### Step 3: Install spotDL
 
```bash
pip3 install spotdl
```
 
Verify the installation:
 
```bash
spotdl --version
```
 
---
 
### Step 4: Add the Shell Function
 
Open your shell config file:
 
```bash
open -e ~/.zshrc
```
 
Add the following:
 
```bash
# Apple Music's auto-import folder — anything dropped here gets added to your library
MUSIC_AUTO="$HOME/Music/Music/Media.localized/Automatically Add to Music.localized"

spotdl-to-iphone() {
  # Require at least one argument
  if [ $# -eq 0 ]; then
    echo "Usage: spotdl-to-iphone <url-or-search> [url-or-search-2] ..."
    return 1
  fi

  # Create a unique temp folder using the process ID ($$) which prevents conflicts if the function is run in parallel from another window
  local MUSIC_TEMP="$HOME/Music/Music/Media.localized/temp_$$"
  mkdir -p "$MUSIC_TEMP"

  echo "🎵 Downloading to temp folder: $MUSIC_TEMP"

  local failed=0

  # Loop through every argument passed to the function
  for query in "$@"; do
    echo "⬇️  Downloading: $query"
    spotdl "$query" --output "$MUSIC_TEMP"

    # $? is the exit code of the last command — non-zero means something went wrong
    if [ $? -ne 0 ]; then
      echo "⚠️  Failed: $query"
      failed=$((failed + 1))
    fi
  done

  # Check if the temp folder has any files before trying to move them
  # -A lists all files except . and ..
  # 2>/dev/null suppresses any error if the folder doesn't exist or is unreadable
  if [ "$(ls -A "$MUSIC_TEMP" 2>/dev/null)" ]; then
    echo "✅ Moving downloads to Music library..."
    mv "$MUSIC_TEMP"/* "$MUSIC_AUTO/"
    rmdir "$MUSIC_TEMP"  # Remove the now-empty temp folder
    echo "🎧 Done! Check your Music app."
  else
    echo "❌ No files downloaded. Temp folder left at: $MUSIC_TEMP"
  fi

  # Final summary if any individual downloads failed
  if [ $failed -gt 0 ]; then
    echo "⚠️  $failed download(s) failed."
  fi
}
```
 
---
 
### Step 5: Reload Your Shell
 
```bash
source ~/.zshrc
```
 

## ▶️ Usage
```bash
spotdl-to-iphone \
  "https://open.spotify.com/track/..." \
  "https://open.spotify.com/track/..." \
  "artist name - song title"
```

## 💡 How It Works
 
```text
Spotify URL(s) / Search Term(s)
               │
               ▼
    ╔══════════════════════════╗
    ║      Spotify API         ║
    ║    (via SpotipyFree)     ║
    ║  · track name & artists  ║
    ║  · album & artwork       ║
    ║  · duration              ║
    ║  · ISRC                  ║
    ╚══════════╦═══════════════╝
               │
               ▼
    ╔══════════════════════════╗
    ║   YouTube Music API      ║
    ║                          ║
    ║  1st: ISRC lookup        ║
    ║  · exact ID search       ║
    ║  · single result →       ║
    ║    instant match         ║
    ║  · multiple → scored,    ║
    ║    accept if score > 80  ║
    ║                          ║
    ║  fallback: text search   ║
    ║  · "{artist} - {title}"  ║
    ║  · candidates scored on: ║
    ║    artist, name,         ║
    ║    duration (strict),    ║
    ║    album                 ║
    ║  · penalizes remix/      ║
    ║    live/cover versions   ║
    ║  · tie-break: view count ║
    ╚══════════╦═══════════════╝
               │ best match selected
               ▼
         Download audio
           (yt-dlp)
               │
               ▼
        Transcode audio
      (FFmpeg → .mp3/.m4a)
               │
               ▼
    Embed Spotify metadata
    into downloaded audio
         (mutagen)
   ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
       spotDL's job ends here
       this tool takes over
   ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
               │
               ▼
   Apple Music auto-import folder
               │
               ▼
       Sync to iPhone / iPod
```