# Targon Resource Pack

Server-side resource pack for Targon (per-enchant-level book icons — "Better Enchantments").
Delivered to players via `server.properties` (`resource-pack` + `resource-pack-sha1`), pointing at
`targon-resourcepack.zip` in this repo via its raw.githubusercontent.com URL.

## Rebuilding the zip after a change

`targon-resourcepack.zip` is a real file committed to this repo, not generated on the fly — it has
to be rebuilt and re-pushed by hand every time `assets/` or `pack.mcmeta` changes, and
`server.properties`' `resource-pack-sha1` updated to match (Minecraft refuses/re-downloads based on
that hash, so a stale one means players never see the update).

**Do not use PowerShell's `Compress-Archive`** to build it — it writes entry paths with backslashes
(`assets\minecraft\...`) instead of forward slashes, which real-world reports say Minecraft's
resource pack loader can fail to read correctly. Build it with `System.IO.Compression.ZipArchive`
directly instead, which lets you control the entry separator:

```powershell
Add-Type -AssemblyName System.IO.Compression
Add-Type -AssemblyName System.IO.Compression.FileSystem

$root = "C:\Users\mozar\Desktop\targon\Targon-ResourcePack"
Set-Location $root
$zipPath = Join-Path $root "targon-resourcepack.zip"
if (Test-Path $zipPath) { Remove-Item -LiteralPath $zipPath -Force }

$files = Get-ChildItem -Path "pack.mcmeta", "assets" -Recurse -File
$fs = [System.IO.File]::Open($zipPath, [System.IO.FileMode]::Create)
$archive = New-Object System.IO.Compression.ZipArchive($fs, [System.IO.Compression.ZipArchiveMode]::Create)
foreach ($file in $files) {
    $relative = $file.FullName.Substring($root.Length + 1) -replace [regex]::Escape([System.IO.Path]::DirectorySeparatorChar), "/"
    $entry = $archive.CreateEntry($relative, [System.IO.Compression.CompressionLevel]::Optimal)
    $entryStream = $entry.Open()
    $bytes = [System.IO.File]::ReadAllBytes($file.FullName)
    $entryStream.Write($bytes, 0, $bytes.Length)
    $entryStream.Close()
}
$archive.Dispose()
$fs.Dispose()

(Get-FileHash -Algorithm SHA1 -LiteralPath $zipPath).Hash.ToLower()
```

Then:

```powershell
git add targon-resourcepack.zip
git commit -m "Update resource pack"
git push
```

...and put the printed SHA1 into `server.properties`' `resource-pack-sha1` (same file, no other
changes needed unless the URL itself moved).

## `pack_format` / `supported_formats` in `pack.mcmeta`

The values currently in `pack.mcmeta` are a **best-effort guess** — this server runs Minecraft
**26.2** (see `.paper/version_history.json`), and the real `pack_format` number for that version
wasn't confirmable at the time this was written. If a client shows a "this pack was made for a
different version" style warning, that's the number to correct.
