# Clean-Music-library-for-better-compatibility


All the commands run on powershell using admin privilege.
1. Install FFmpeg 
winget install ffmpeg

2. This command cleans files without touching sound quality. 
> Images are converted to a 1000x1000 pixel size
2.a. Command (Targeting 1 folder):
# Edit the path below to your specific folder
$TargetFolder = "C:\Users\example\Music\x_x"

$Files = Get-ChildItem -LiteralPath $TargetFolder -Include *.flac, *.mp3 -File
foreach ($File in $Files) {
    Write-Host "Standardizing: $($File.Name)" -ForegroundColor Cyan
    $HasVideo = & ffprobe -v error -select_streams v:0 -show_entries stream=codec_name -of csv=p=0 "$($File.FullName)"
    $TempFile = $File.FullName + ".tmp.flac"
    if (![string]::IsNullOrWhiteSpace($HasVideo)) {
        & ffmpeg -i "$($File.FullName)" -map 0:a -map 0:v:0 -c:a copy -c:v mjpeg -pix_fmt yuvj420p -vf "scale='min(1000,iw)':-1" -disposition:v:0 attached_pic "$TempFile" -y -loglevel error
    } else {
        & ffmpeg -i "$($File.FullName)" -c:a copy "$TempFile" -y -loglevel error
    }
    if ($LASTEXITCODE -eq 0) {
        Remove-Item -LiteralPath $File.FullName -Force
        Rename-Item -Path $TempFile -NewName $File.Name
    }
}

2.b. Library-Wide Standardizer (Automated) Targeting whole library and subfolders within a path.
If you want to process everything inside your main Music folder at once without specifying sub-folders.
Scans recursively through all subfolders.
Skips audio re-encoding entirely.
Ensures no gray pixel formats crash the player.
Command (Whole Library):
$ParentFolder = "C:\Users\example\Music"

$Files = Get-ChildItem -Path $ParentFolder -Recurse -Include *.flac, *.mp3 -File
foreach ($File in $Files) {
    $HasVideo = & ffprobe -v error -select_streams v:0 -show_entries stream=codec_name -of csv=p=0 "$($File.FullName)"
    $TempFile = $File.FullName + ".tmp.flac"
    if (![string]::IsNullOrWhiteSpace($HasVideo)) {
        & ffmpeg -i "$($File.FullName)" -map 0:a -map 0:v:0 -c:a copy -c:v mjpeg -pix_fmt yuvj420p -vf "scale='min(1000,iw)':-1" -disposition:v:0 attached_pic "$TempFile" -y -loglevel error
        if ($LASTEXITCODE -eq 0) {
            Remove-Item -LiteralPath $File.FullName -Force
            Rename-Item -Path $TempFile -NewName $File.Name
            Write-Host "Success: $($File.Name)" -ForegroundColor Green
        }
    }
}

individual file error during re-encoding:
Use this only when a file (like an .m4a with a broken image) refuses to play or convert
$f = "C:\Users\example\Music\x_x\Juice WRLD - moonlight.m4a"; ffmpeg -i $f -vn -c:a flac -compression_level 5 ([System.IO.Path]::ChangeExtension($f, ".flac")) -y

Bonus: Timed Lyrics (.lrc) Downloader
$ParentFolder = "C:\Users\example\Music"

$Files = Get-ChildItem -Path $ParentFolder -Recurse -Include *.flac, *.mp3, *.m4a -File
foreach ($File in $Files) {
    $LrcPath = Join-Path $File.DirectoryName ($File.BaseName + ".lrc")
    if (Test-Path -LiteralPath $LrcPath) { continue }
    $SearchTerm = [uri]::EscapeDataString(($File.BaseName -replace "\(.*?\)", "").Trim())
    try {
        $Match = (Invoke-RestMethod "https://lrclib.net/api/search?q=$SearchTerm") | Where-Object { ![string]::IsNullOrEmpty($_.syncedLyrics) } | Select-Object -First 1
        if ($Match) { 
            $Match.syncedLyrics | Out-File -LiteralPath $LrcPath -Encoding utf8
            Write-Host "LRC Found: $($File.BaseName)" -ForegroundColor Green 
        }
    } catch {}
    Start-Sleep -Milliseconds 200
}

Bonus: Genre Normalization
Mapping Table:
Converts same gerne across different languegues into 1 cohisive one for example: 
Alternatif et indé
alternativ und indie
alternative & indie
alternative en indie
alternativo & indie 
= into 1
Paste the tags that are in different langueges or duplicate and ask AI to insert those changes in your command. The command bello wis just a example:

$GenreMap = @{"ロック"="Rock"; "ワールド"="World"; "Alternatif et indé"="Alternative"}
$Files = Get-ChildItem -Path "C:\Users\example\Music" -Recurse -Include *.flac, *.mp3
foreach ($File in $Files) {
    $Current = & ffprobe -v error -show_entries format_tags=genre -of default=noprint_wrappers=1:nokey=1 "$($File.FullName)"
    if ($GenreMap.ContainsKey($Current.Trim())) {
        $New = $GenreMap[$Current.Trim()]
        $Temp = $File.FullName + ".tag.tmp"
        & ffmpeg -i "$($File.FullName)" -c copy -metadata genre="$New" "$Temp" -y -loglevel error
        Move-Item -LiteralPath $Temp -Destination $File.FullName -Force
    }
}
Lastly. Transfer (Robocopy)
The fastest way to move your library to your external storage.
robocopy "C:\Users\example\path\from" "F:\Music\example\path\to" /E /MT:32 /Z /R:3 /W:5 /FFT
