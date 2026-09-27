# AUDIO LIBRARY MANAGER

PowerShell scripts for standardizing, repairing, organizing, and transferring audio libraries with **zero audio quality loss**.

---

## Requirements

* **Operating System:** Windows
* **Shell:** PowerShell
* **Permissions:** Run PowerShell as **Administrator**
* **Dependencies:** FFmpeg and FFprobe

---

## 1. Install FFmpeg & [Flac](https://github.com/xiph/flac/releases/tag/1.5.0)

Place the folder containing "flac.exe" & metaflac.exe in to your "C:\" within a folder, such as "C:\flac"

FFmpeg is the core engine used for audio processing, metadata editing, and artwork standardization.

```powershell
winget install ffmpeg
```

Verify the installation:

```powershell
ffmpeg -version
ffprobe -version
```

---

## 2. File Standardizer (No Quality Loss)

Standardizes album artwork and container metadata without re-encoding audio.

### Features

* Preserves original audio quality (**bit-perfect copy**)
* Maintains sample rate and bit depth (for example, **192 kHz / 24-bit**)
* Resizes embedded artwork to a maximum of **1000 × 1000 pixels**
* Converts artwork to a compatible format
* Prevents grayscale artwork crashes
* Reduces PNG memory overhead

---

### 2.1 Standardize a Single Folder

Edit the path below to match your music folder.

```powershell
$TargetFolder = "C:\Users\example\Music\x_x"

$Files = Get-ChildItem -LiteralPath $TargetFolder -Include *.flac, *.mp3 -File

foreach ($File in $Files) {
    Write-Host "Standardizing: $($File.Name)" -ForegroundColor Cyan

    $HasVideo = & ffprobe -v error -select_streams v:0 `
        -show_entries stream=codec_name `
        -of csv=p=0 "$($File.FullName)"

    $TempFile = $File.FullName + ".tmp.flac"

    if (![string]::IsNullOrWhiteSpace($HasVideo)) {
        & ffmpeg -i "$($File.FullName)" `
            -map 0:a `
            -map 0:v:0 `
            -c:a copy `
            -c:v mjpeg `
            -pix_fmt yuvj420p `
            -vf "scale='min(1000,iw)':-1" `
            -disposition:v:0 attached_pic `
            "$TempFile" -y -loglevel error
    }
    else {
        & ffmpeg -i "$($File.FullName)" `
            -c:a copy `
            "$TempFile" -y -loglevel error
    }

    if ($LASTEXITCODE -eq 0) {
        Remove-Item -LiteralPath $File.FullName -Force
        Rename-Item -Path $TempFile -NewName $File.Name
    }
}
```

---

### 2.2 Standardize an Entire Library

Processes all supported files recursively.

### Features

* Scans all subfolders automatically
* Skips audio re-encoding entirely
* Standardizes embedded artwork
* Fixes incompatible pixel formats

```powershell
$ParentFolder = "C:\Users\example\Music"

$Files = Get-ChildItem -Path $ParentFolder -Recurse -Include *.flac, *.mp3 -File

foreach ($File in $Files) {
    $HasVideo = & ffprobe -v error -select_streams v:0 `
        -show_entries stream=codec_name `
        -of csv=p=0 "$($File.FullName)"

    $TempFile = $File.FullName + ".tmp.flac"

    if (![string]::IsNullOrWhiteSpace($HasVideo)) {
        & ffmpeg -i "$($File.FullName)" `
            -map 0:a `
            -map 0:v:0 `
            -c:a copy `
            -c:v mjpeg `
            -pix_fmt yuvj420p `
            -vf "scale='min(1000,iw)':-1" `
            -disposition:v:0 attached_pic `
            "$TempFile" -y -loglevel error

        if ($LASTEXITCODE -eq 0) {
            Remove-Item -LiteralPath $File.FullName -Force
            Rename-Item -Path $TempFile -NewName $File.Name
            Write-Host "Success: $($File.Name)" -ForegroundColor Green
        }
    }
}
```

---

## 3. Repair a Corrupted File

Use this command when a file refuses to play or convert due to damaged embedded artwork or metadata.

> **Warning:** This process removes embedded artwork and re-encodes the audio to FLAC.

```powershell
$f = "C:\Users\example\Music\x_x\Juice WRLD - moonlight.m4a"

ffmpeg -i $f -vn -c:a flac -compression_level 5 `
    ([System.IO.Path]::ChangeExtension($f, ".flac")) -y
```

---

## 4k Block Size Standardizer (Hardware Compatibility)

Why this is needed
High-res FLAC files often use large internal block sizes. Hardware players can fail to play these if a block exceeds their internal buffer. Forcing a 4096 (4k) block size ensures the hardware can digest the data stream without losing quality.

```powershell
$ParentFolder = "C:\Users\example\Music"

$FlacExe     = "C:\flac\flac.exe"
$MetaFlacExe = "C:\flac\metaflac.exe"

# ------------------------------------------------------------
# Check required programs
# ------------------------------------------------------------

if (!(Test-Path -LiteralPath $FlacExe)) {
    Write-Host "ERROR: flac.exe not found at $FlacExe" -ForegroundColor Red
    exit 1
}

if (!(Test-Path -LiteralPath $MetaFlacExe)) {
    Write-Host "ERROR: metaflac.exe not found at $MetaFlacExe" -ForegroundColor Red
    exit 1
}

if (!(Test-Path -LiteralPath $ParentFolder)) {
    Write-Host "ERROR: Music folder not found: $ParentFolder" -ForegroundColor Red
    exit 1
}

# ------------------------------------------------------------
# Find all FLAC files recursively
# ------------------------------------------------------------

$Files = Get-ChildItem `
    -Path $ParentFolder `
    -Recurse `
    -Filter *.flac `
    -File

$Total   = $Files.Count
$Current = 0
$Success = 0
$Skipped = 0
$Failed  = 0

Write-Host ""
Write-Host "============================================" -ForegroundColor Cyan
Write-Host "FiiO ECHO MINI FLAC Converter" -ForegroundColor Cyan
Write-Host "============================================" -ForegroundColor Cyan
Write-Host "Library: $ParentFolder"
Write-Host "Files found: $Total"
Write-Host "Target maximum block size: 4096"
Write-Host ""

foreach ($File in $Files) {

    $Current++

    Write-Host "[$Current/$Total] $($File.Name)" -ForegroundColor White

    # Unique temporary file in the same directory
    $TempFile = Join-Path `
        $File.DirectoryName `
        (".__ECHO_MINI_REBLOCK_" + [guid]::NewGuid().ToString() + ".flac")

    try {

        # --------------------------------------------------------
        # 1. Read current block size
        # --------------------------------------------------------

        $BlockOutput = & $MetaFlacExe `
            --show-min-blocksize `
            --show-max-blocksize `
            "$($File.FullName)" 2>&1

        if ($LASTEXITCODE -ne 0) {
            throw "Could not read FLAC metadata."
        }

        $BlockValues = @(
            $BlockOutput |
            Where-Object { $_ -match '^\d+$' } |
            ForEach-Object { [int]$_ }
        )

        if ($BlockValues.Count -lt 2) {
            throw "Could not determine block size."
        }

        $MinBlock = $BlockValues[0]
        $MaxBlock = $BlockValues[1]

        Write-Host "    Current block size: $MinBlock / $MaxBlock"

        # --------------------------------------------------------
        # 2. Skip files already compatible
        # --------------------------------------------------------

        if ($MaxBlock -le 4096) {
            Write-Host "    Already compatible - SKIPPED" -ForegroundColor Yellow
            $Skipped++
            Write-Host ""
            continue
        }

        # --------------------------------------------------------
        # 3. Lossless FLAC re-encode
        # --------------------------------------------------------

        Write-Host "    Re-encoding losslessly..." -ForegroundColor Cyan

        & $FlacExe `
            --force `
            --blocksize=4096 `
            --output-name="$TempFile" `
            "$($File.FullName)" 2>&1 | Out-Null

        if ($LASTEXITCODE -ne 0) {
            throw "FLAC conversion failed."
        }

        if (!(Test-Path -LiteralPath $TempFile)) {
            throw "FLAC encoder did not create an output file."
        }

        # --------------------------------------------------------
        # 4. Verify FLAC integrity using the FLAC decoder itself
        # --------------------------------------------------------

        Write-Host "    Testing FLAC integrity..." -ForegroundColor Cyan

        & $FlacExe `
            --test `
            "$TempFile" 2>&1 | Out-Null

        if ($LASTEXITCODE -ne 0) {
            throw "FLAC integrity test failed."
        }

        # --------------------------------------------------------
        # 5. Verify block size
        # --------------------------------------------------------

        $NewBlockOutput = & $MetaFlacExe `
            --show-min-blocksize `
            --show-max-blocksize `
            "$TempFile" 2>&1

        if ($LASTEXITCODE -ne 0) {
            throw "Could not read converted FLAC metadata."
        }

        $NewBlockValues = @(
            $NewBlockOutput |
            Where-Object { $_ -match '^\d+$' } |
            ForEach-Object { [int]$_ }
        )

        if ($NewBlockValues.Count -lt 2) {
            throw "Could not determine converted block size."
        }

        $NewMinBlock = $NewBlockValues[0]
        $NewMaxBlock = $NewBlockValues[1]

        Write-Host "    New block size:     $NewMinBlock / $NewMaxBlock"

        if ($NewMaxBlock -gt 4096) {
            throw "Converted file still exceeds 4096 block size."
        }

        # --------------------------------------------------------
        # 6. Replace original
        # --------------------------------------------------------

        Write-Host "    Replacing original..." -ForegroundColor Cyan

        Remove-Item `
            -LiteralPath $File.FullName `
            -Force

        Rename-Item `
            -LiteralPath $TempFile `
            -NewName $File.Name

        Write-Host "    SUCCESS - original filename preserved" -ForegroundColor Green

        $Success++

    }
    catch {

        Write-Host "    FAILED: $($_.Exception.Message)" -ForegroundColor Red

        # Delete failed temporary file
        if (Test-Path -LiteralPath $TempFile) {
            Remove-Item `
                -LiteralPath $TempFile `
                -Force `
                -ErrorAction SilentlyContinue
        }

        $Failed++
    }

    Write-Host ""
}

# ------------------------------------------------------------
# Summary
# ------------------------------------------------------------

Write-Host "============================================" -ForegroundColor Cyan
Write-Host "Finished!" -ForegroundColor Cyan
Write-Host "============================================" -ForegroundColor Cyan

Write-Host "Total files: $Total"
Write-Host "Converted:   $Success" -ForegroundColor Green
Write-Host "Skipped:     $Skipped" -ForegroundColor Yellow
Write-Host "Failed:      $Failed" -ForegroundColor Red

Write-Host "============================================" -ForegroundColor Cyan

```

---
---

## Bonus: Timed Lyrics Downloader (.LRC)

Downloads synchronized lyrics from LRCLib for files that do not already have `.lrc` files.

### Features

* Recursively scans your library
* Skips tracks with existing lyrics
* Saves synchronized lyrics alongside each audio file

```powershell
$ParentFolder = "C:\Users\example\Music"

$Files = Get-ChildItem -Path $ParentFolder -Recurse -Include *.flac, *.mp3, *.m4a -File

foreach ($File in $Files) {
    $LrcPath = Join-Path $File.DirectoryName ($File.BaseName + ".lrc")

    if (Test-Path -LiteralPath $LrcPath) {
        continue
    }

    $SearchTerm = [uri]::EscapeDataString(
        ($File.BaseName -replace "\(.*?\)", "").Trim()
    )

    try {
        $Match = (
            Invoke-RestMethod "https://lrclib.net/api/search?q=$SearchTerm"
        ) | Where-Object {
            ![string]::IsNullOrEmpty($_.syncedLyrics)
        } | Select-Object -First 1

        if ($Match) {
            $Match.syncedLyrics | Out-File -LiteralPath $LrcPath -Encoding utf8
            Write-Host "LRC Found: $($File.BaseName)" -ForegroundColor Green
        }
    }
    catch {}

    Start-Sleep -Milliseconds 200
}
```

---

## Bonus: Genre Normalization

Standardizes genre tags across multiple languages.

### Example Mapping

| Found Genre            | Normalized To |
| ---------------------- | ------------- |
| `Alternatif et indé`   | `Alternative` |
| `alternativ und indie` | `Alternative` |
| `ロック`                  | `Rock`        |

Update the mapping table as needed.

```powershell
# 1. FORCE POWERSHELL TO USE UTF-8 FOR EVERYTHING
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8

$MusicPath = "C:\Users\ProudCatOwner\Music"

# 2. Define the Mapping
$GenreMap = @{
    "ロック" = "Rock"
    "ヒップホップ" = "Hip Hop"
    "ワールド" = "World"
    "サウンドトラック" = "Soundtrack"
    "エレクトロニック" = "Electronic"
    "Électronique" = "Electronic"
    "オルタナティヴ" = "Alternative & Indie"
    "Alternatif" = "Alternative & Indie"
}

Write-Host "--- UTF-8 Forced Translation Started ---" -ForegroundColor Cyan

# Find all music files
$Files = Get-ChildItem -Path $MusicPath -Recurse -Include *.flac, *.mp3 -File

foreach ($File in $Files) {
    # Get genre using UTF-8 encoding
    $RawGenre = & ffprobe -v error -show_entries format_tags=genre -of default=noprint_wrappers=1:nokey=1 "$($File.FullName)"
    
    if (![string]::IsNullOrWhiteSpace($RawGenre)) {
        $RawGenre = $RawGenre.Trim()
        $NewGenre = $null

        # Check for matches
        foreach ($Key in $GenreMap.Keys) {
            if ($RawGenre -like "*$Key*") {
                $NewGenre = $GenreMap[$Key]
                break
            }
        }

        if ($null -ne $NewGenre) {
            Write-Host "Match Found: $($File.Name)" -ForegroundColor Yellow
            Write-Host "   Tag: $RawGenre -> $NewGenre" -ForegroundColor Gray
            
            $Temp = $File.FullName + ".gtmp.flac"
            
            # -c copy ensures 192kHz audio is BIT-PERFECT
            & ffmpeg -i "$($File.FullName)" -c copy -metadata genre="$NewGenre" -map_metadata 0 "$Temp" -y -loglevel error
            
            if ($LASTEXITCODE -eq 0) {
                Move-Item -LiteralPath $Temp -Destination $File.FullName -Force
                Write-Host "   SUCCESS" -ForegroundColor Green
            } else {
                if (Test-Path $Temp) { Remove-Item $Temp }
            }
        }
    }
}
Write-Host "--- Translation Complete ---" -ForegroundColor Cyan
```

## Gerne tagging for unknown gerne

using Deezer , iTunes and MusicBrainz


```powershell
# 1. Configuration
$MusicPath = "C:\Users\ProudCatOwner\Music"
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8

Write-Host "--- DEFINITIVE MULTI-DATABASE GENRE FETCHING ---" -ForegroundColor Cyan
Write-Host "Audio: Bit-Perfect Copy | Databases: Deezer, iTunes, MusicBrainz`n"

$Files = Get-ChildItem -Path $MusicPath -Recurse -Include *.flac, *.mp3 -File

foreach ($File in $Files) {
    # Check if Genre is missing
    $CurrentGenre = & ffprobe -v error -show_entries format_tags=genre -of default=noprint_wrappers=1:nokey=1 "$($File.FullName)"
    
    if (![string]::IsNullOrWhiteSpace($CurrentGenre) -and $CurrentGenre -notmatch "Unknown|None|Genre|Other") {
        continue # Skip if already tagged
    }

    # Get Artist and Title
    $Artist = & ffprobe -v error -show_entries format_tags=artist -of default=noprint_wrappers=1:nokey=1 "$($File.FullName)"
    $Title = & ffprobe -v error -show_entries format_tags=title -of default=noprint_wrappers=1:nokey=1 "$($File.FullName)"

    if ([string]::IsNullOrWhiteSpace($Artist) -or [string]::IsNullOrWhiteSpace($Title)) {
        Write-Host "SKIP: Missing Artist/Title tags for $($File.Name)" -ForegroundColor Gray
        continue
    }

    # Clean Title for better searching (Remove (Lyrics), [Official], etc)
    $CleanTitle = $Title -replace "\(.*?\)", "" -replace "\[.*?\]", "" -replace "feat\..*", ""
    $SearchTerm = "$Artist $CleanTitle".Trim()
    $NewGenre = $null

    Write-Host "Searching: $SearchTerm... " -NoNewline

    # --- DATABASE 1: DEEZER (Best for General/Remixes) ---
    try {
        $Url = "https://api.deezer.com/search?q=" + [uri]::EscapeDataString($SearchTerm)
        $Resp = Invoke-RestMethod -Uri $Url -TimeoutSec 5
        if ($Resp.total -gt 0) {
            $AlbumId = $Resp.data[0].album.id
            $AlbumResp = Invoke-RestMethod -Uri "https://api.deezer.com/album/$AlbumId"
            $NewGenre = $AlbumResp.genres.data[0].name
        }
    } catch { }

    # --- DATABASE 2: ITUNES (Fallback - Best for Japanese/Pop) ---
    if ([string]::IsNullOrWhiteSpace($NewGenre)) {
        try {
            $Url = "https://itunes.apple.com/search?term=" + [uri]::EscapeDataString($SearchTerm) + "&media=music&limit=1"
            $Resp = Invoke-RestMethod -Uri $Url -TimeoutSec 5
            if ($Resp.resultCount -gt 0) { $NewGenre = $Resp.results[0].primaryGenreName }
        } catch { }
    }

    # --- DATABASE 3: MUSICBRAINZ (Last Resort - Best for Underground) ---
    if ([string]::IsNullOrWhiteSpace($NewGenre)) {
        try {
            $Query = [uri]::EscapeDataString("artist:""$Artist"" AND recording:""$CleanTitle""")
            $Url = "https://musicbrainz.org/ws/2/recording/?query=$Query&fmt=json"
            $Resp = Invoke-RestMethod -Uri $Url -UserAgent "SnowskyManager/1.1" -TimeoutSec 5
            $NewGenre = $Resp.recordings[0].tags | Sort-Object count -Descending | Select-Object -ExpandProperty name -First 1
        } catch { }
    }

    # --- APPLY GENRE IF FOUND ---
    if (![string]::IsNullOrWhiteSpace($NewGenre)) {
        Write-Host "FOUND: $NewGenre" -ForegroundColor Green
        $Temp = $File.FullName + ".tagtmp.flac"
        
        # -c copy is MANDATORY: It preserves the 192kHz/24-bit audio exactly.
        & ffmpeg -i "$($File.FullName)" -c copy -metadata genre="$NewGenre" -map_metadata 0 "$Temp" -y -loglevel error
        
        if ($LASTEXITCODE -eq 0) {
            Move-Item -LiteralPath $Temp -Destination $File.FullName -Force
        } else {
            if (Test-Path $Temp) { Remove-Item $Temp }
        }
    } else {
        Write-Host "NOT FOUND" -ForegroundColor Yellow
    }

    # API Politeness Delay
    Start-Sleep -Milliseconds 400
}
Write-Host "`nProcess Complete!" -ForegroundColor Cyan
```


---

## Transfer Your Library with Robocopy

The fastest and most reliable way to move your library to external storage.

```batch
robocopy "C:\Users\example\path\from" "F:\" /E /MT:32 /Z /R:3 /W:5 /FFT
```

### Robocopy Flags

| Flag     | Function                                               |
| -------- | ------------------------------------------------------ |
| `/E`     | Copies all subdirectories, including empty folders     |
| `/MT:32` | Uses 32 simultaneous threads for faster transfers      |
| `/Z`     | Enables restartable mode for interrupted transfers     |
| `/R:3`   | Retries failed copies three times                      |
| `/W:5`   | Waits five seconds between retries                     |
| `/FFT`   | Improves compatibility with FAT and exFAT file systems |

---

## Supported Audio Formats

* FLAC
* MP3
* M4A (repair and lyrics download only)

---

## Disclaimer

Always create a backup before batch-processing your music library. Although these scripts are designed to preserve audio quality, metadata operations modify files in place.
