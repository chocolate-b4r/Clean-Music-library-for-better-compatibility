# AUDIO LIBRARY MANAGER

PowerShell scripts for standardizing, repairing, organizing, and transferring audio libraries with **zero audio quality loss**.

---

## Requirements

* **Operating System:** Windows
* **Shell:** PowerShell
* **Permissions:** Run PowerShell as **Administrator**
* **Dependencies:** FFmpeg and FFprobe

---

## 1. Install FFmpeg

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
