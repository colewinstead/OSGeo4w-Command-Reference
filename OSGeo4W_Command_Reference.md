# OSGeo4W / GDAL Command Reference for Civil Raster Workflows

Practical Windows command-line recipes for inspecting, converting, georeferencing, mosaicking, clipping, reprojecting, and optimizing raster data used in QGIS, OpenRoads, and MicroStation V8i.

> **Safety rule:** Keep the source imagery unchanged. Write to a new output file unless you intentionally use an in-place metadata editor. After every important operation, inspect the output with `gdalinfo` before attaching it in CAD.

## My Most Used OSGeo4W Commands

These are the everyday copy/paste commands. Replace the obvious placeholders before running them:

```text
"C:\Path\Input.tif"
"C:\Path\Output.tif"
"C:\Path\Boundary.gpkg"
EPSG:SOURCE
EPSG:TARGET
```

Run them in the **OSGeo4W Shell**. Commands below create new output files unless an in-place warning is shown.

### Inspect an aerial TIFF

**What it does:** Reports the CRS, dimensions, pixel size, origin/geotransform, corner coordinates, bands, data type, NoData, metadata, compression, block layout, and overviews.

**When I would use it:** Always run this before converting, merging, clipping, or attaching an unfamiliar aerial in QGIS or CAD.

```bat
gdalinfo "C:\Path\Input.tif"
```

For quick approximate band statistics on a huge file:

```bat
gdalinfo -approx_stats "C:\Path\Input.tif"
```

**Watch out for:** `-stats`, `-hist`, and `-checksum` may read billions of pixels and take a long time.

### Get CRS information

**What it does:** Decodes the raster CRS and can look for a matching EPSG identifier.

```bat
gdalsrsinfo "C:\Path\Input.tif"
gdalsrsinfo -e "C:\Path\Input.tif"
```

**Watch out for:** A similar CRS name is not enough. Confirm the datum, State Plane/UTM zone, and meter versus international-foot versus US-survey-foot definition.

### Convert a COG or other TIFF to a standard GeoTIFF

**What it does:** Reads the source through GDAL and writes a conventional GTiff output with conservative, lossless settings.

**When I would use it:** When OpenRoads or MicroStation does not reliably read the source layout or codec.

```bat
gdal_translate -of GTiff -co TILED=YES -co BLOCKXSIZE=256 -co BLOCKYSIZE=256 -co COMPRESS=DEFLATE -co PREDICTOR=2 -co INTERLEAVE=PIXEL -co BIGTIFF=IF_NEEDED "C:\Path\Input.tif" "C:\Path\Output.tif"
```

This creates a new TIFF. A Cloud Optimized GeoTIFF is still a kind of GeoTIFF; this command rewrites its storage layout using the normal `GTiff` driver rather than the `COG` driver.

**Watch out for:** Do not add `-ot Byte` until `gdalinfo` confirms that the raster is RGB imagery or that a deliberate scale to 0–255 is acceptable. Terrain/elevation data should usually remain `Int16`, `UInt16`, or `Float32`.

### Convert RGB aerial imagery to a smaller JPEG-compressed tiled TIFF

**What it does:** Creates an 8-bit, three-band, pixel-interleaved aerial TIFF with lossy JPEG compression.

**When I would use it:** Visual aerial photography where smaller files and faster I/O matter more than preserving exact pixel values.

```bat
gdal_translate -of GTiff -ot Byte -b 1 -b 2 -b 3 -colorinterp_1 red -colorinterp_2 green -colorinterp_3 blue -co TILED=YES -co BLOCKXSIZE=256 -co BLOCKYSIZE=256 -co COMPRESS=JPEG -co JPEG_QUALITY=90 -co PHOTOMETRIC=YCBCR -co INTERLEAVE=PIXEL -co BIGTIFF=IF_NEEDED "C:\Path\Input.tif" "C:\Path\Output.tif"
```

**Watch out for:** This assumes bands 1–3 are already 0–255 RGB values. If the source is UInt16 or has another display range, inspect it first and use a deliberate `-scale <src_min> <src_max> 0 255`. Never use this recipe for DEMs, terrain, masks, class rasters, or data that must be pixel-exact. JPEG is lossy, and old V8i installations should be tested with one representative file.

### Reproject an aerial into the project CRS

**What it does:** Transforms the raster grid from the source CRS into the target CRS.

**When I would use it:** The aerial is correctly georeferenced, but its CRS differs from the OpenRoads/QGIS project CRS.

```bat
gdalwarp -s_srs EPSG:SOURCE -t_srs EPSG:TARGET -r cubic -multi -wo NUM_THREADS=ALL_CPUS -co TILED=YES -co BLOCKXSIZE=256 -co BLOCKYSIZE=256 -co COMPRESS=DEFLATE -co PREDICTOR=2 -co BIGTIFF=IF_NEEDED "C:\Path\Input.tif" "C:\Path\Output.tif"
```

`-s_srs` identifies or overrides the source CRS for this operation. `-t_srs` selects the target and causes the actual reprojection. If the embedded source CRS is correct, omit `-s_srs` rather than repeating it.

**Watch out for:** Reprojection changes horizontal coordinates and resamples the pixel grid. It does not necessarily convert elevation cell values between feet and meters.

### Assign a CRS without moving pixels

**What it does:** Writes a CRS label when the raster coordinates are already correct but the CRS metadata is missing or wrong.

```bat
gdal_edit -a_srs EPSG:XXXX "C:\Path\Input.tif"
```

> **In-place warning:** `gdal_edit` alters the input file. Back it up first. This command does **not** reproject, shift, or resample the raster.

Safer new-copy alternative:

```bat
gdal_translate -a_srs EPSG:XXXX "C:\Path\Input.tif" "C:\Path\Output.tif"
```

### Physically merge adjacent aerial TIFFs

**What it does:** Writes all source pixels into one physical TIFF.

```bat
gdal_merge.py -o "C:\Path\Merged.tif" -of GTiff -co TILED=YES -co BLOCKXSIZE=256 -co BLOCKYSIZE=256 -co COMPRESS=JPEG -co JPEG_QUALITY=90 -co PHOTOMETRIC=YCBCR -co INTERLEAVE=PIXEL -co BIGTIFF=YES "C:\Path\Tile1.tif" "C:\Path\Tile2.tif" "C:\Path\Tile3.tif"
```

Depending on the installation, the launcher may be `gdal_merge`, `gdal_merge.py`, or `gdal_merge.bat`.

**Watch out for:** Inputs should already have compatible CRS, band layout, resolution, and data type. A physical merge can become enormous. `gdal_merge` does not perform a true reprojection; use `gdalwarp` to align incompatible inputs first. `BIGTIFF=YES` avoids the Classic TIFF 4 GiB limit but may reduce compatibility with older V8i builds.

### Build a VRT instead of physically merging

**What it does:** Creates a tiny XML mosaic that references the source TIFFs without copying their pixels.

```bat
gdalbuildvrt "C:\Path\AerialMosaic.vrt" "C:\Path\Tile1.tif" "C:\Path\Tile2.tif" "C:\Path\Tile3.tif"
```

For a large list:

```bat
gdalbuildvrt -input_file_list "C:\Path\RasterList.txt" "C:\Path\AerialMosaic.vrt"
```

**Why I would use it:** VRTs are fast to create, take almost no extra disk space, and are excellent in QGIS or as input to `gdalwarp`/`gdal_translate`. They avoid committing to a gigantic merged TIFF.

**Watch out for:** A VRT depends on the source filenames and paths. Moving, renaming, disconnecting, or deleting the source TIFFs breaks it. A VRT does not reproject mismatched inputs.

### Build external overviews for faster display

**What it does:** Creates successively smaller raster pyramids in `Input.tif.ovr`.

```bat
gdaladdo -ro -r average --config COMPRESS_OVERVIEW DEFLATE --config BIGTIFF_OVERVIEW IF_SAFER "C:\Path\Input.tif" 2 4 8 16 32 64
```

Level `2` is half the source width and height, level `4` is one quarter, and so on. These levels make zoomed-out display much faster.

**Watch out for:** The `.ovr` supplements the image; it does not replace it. Keep it beside the TIFF. Attach the **`.tif`** in MicroStation/OpenRoads, not the `.ovr`.

### Set or change NoData

Create a new output while declaring NoData:

```bat
gdal_translate -a_nodata 0 "C:\Path\Input.tif" "C:\Path\Output.tif"
```

Set it in place only when intentional:

```bat
gdal_edit -a_nodata 0 "C:\Path\Input.tif"
```

Clear an in-place NoData declaration:

```bat
gdal_edit -unsetnodata "C:\Path\Input.tif"
```

**Watch out for:** Do not declare `0` as NoData if legitimate black pixels must remain visible. NoData is band-value metadata; it is different from an alpha band or a per-dataset mask.

### Remove a black aerial collar with transparency

**What it does:** Finds near-black pixels connected to the outside collar and creates transparency in a new lossless TIFF.

Alpha-band version:

```bat
nearblack -of GTiff -co TILED=YES -co COMPRESS=DEFLATE -setalpha -near 15 -o "C:\Path\Output.tif" "C:\Path\Input.tif"
```

Internal-mask version for GDAL-aware software:

```bat
nearblack --config GDAL_TIFF_INTERNAL_MASK YES -of GTiff -co TILED=YES -co COMPRESS=DEFLATE -setmask -near 15 -o "C:\Path\Output.tif" "C:\Path\Input.tif"
```

**Watch out for:** Preview the output. Genuine dark imagery touching the edge can be removed. Alpha/mask support varies in older CAD raster engines, so test the exact V8i build. Supplying `-o` is important: without an output, `nearblack` can update a compatible input in place.

### Create a world file with a new TIFF

```bat
gdal_translate -of GTiff -co TFW=YES "C:\Path\Input.tif" "C:\Path\Output.tif"
```

The resulting `.tfw` stores affine placement, not the full CRS. A GeoTIFF may already contain its placement and CRS internally.

### Clip an aerial to a project boundary

Vector cutline:

```bat
gdalwarp -cutline "C:\Path\Boundary.gpkg" -crop_to_cutline -dstalpha -r cubic -co TILED=YES -co COMPRESS=DEFLATE -co BIGTIFF=IF_NEEDED "C:\Path\Input.tif" "C:\Path\Output.tif"
```

Known rectangular extent:

```bat
gdalwarp -te XMIN YMIN XMAX YMAX -te_srs EPSG:XXXX -r cubic -co TILED=YES -co COMPRESS=DEFLATE "C:\Path\Input.tif" "C:\Path\Output.tif"
```

**Watch out for:** `-te` order is minimum X, minimum Y, maximum X, maximum Y. Confirm the cutline layer CRS and use `-cl <layer_name>` when the container has multiple layers.

### Turn on useful threading for the current shell

```bat
set GDAL_NUM_THREADS=ALL_CPUS
```

For a warp operation:

```bat
gdalwarp -multi -wo NUM_THREADS=ALL_CPUS -wm 1024 -co NUM_THREADS=ALL_CPUS "C:\Path\Input.tif" "C:\Path\Output.tif"
```

`GDAL_NUM_THREADS` and `-co NUM_THREADS=ALL_CPUS` help supported TIFF compression/decompression; `-wo NUM_THREADS=ALL_CPUS` controls warp computation; `-multi` overlaps I/O and processing. Not every GDAL stage or codec uses every core, and the GeoTIFF driver ignores compression threading for JPEG.

## Choosing the Right Raster Output

### Byte versus Float32

| Type | Best fit | Why | Do not use when |
|---|---|---|---|
| `Byte` | Normal red/green/blue aerial imagery and 0–255 masks/classes | Compact and compatible with JPEG compression | Elevations or continuous values require decimals or a range outside 0–255 |
| `Float32` | DEMs, terrain, elevations, calculated surfaces, ratios | Preserves fractional continuous values and a large numeric range | A source is ordinary 8-bit RGB imagery with no analytical values |

Converting a DEM to Byte can clamp, round, or rescale thousands of possible elevations into only 256 values. That can permanently destroy precision. Preserve the original type unless the output is intentionally only a visual rendering.

### JPEG versus DEFLATE and LZW

| Compression | Lossy? | Best use | Key tradeoff |
|---|---:|---|---|
| `JPEG` | Yes | 8-bit RGB aerial photographs | Usually much smaller; changes pixel values and is unsuitable for DEMs, masks, classes, or analytical imagery |
| `DEFLATE` | No | Aerials requiring exact values, DEMs, masks, maps, and categorical rasters | Often compact but slower to compress; `PREDICTOR` and threading can help |
| `LZW` | No | Conservative cross-software TIFF delivery | Broad compatibility; size may be larger than DEFLATE |

For JPEG RGB TIFFs, use `PHOTOMETRIC=YCBCR` and `INTERLEAVE=PIXEL`. For lossless integer data, try `PREDICTOR=2`; for floating-point data, use `PREDICTOR=3` if the target software supports the result.

### Physical TIFF mosaic versus VRT

| Choice | Advantages | Disadvantages | Good fit |
|---|---|---|---|
| Physical TIFF | Self-contained; straightforward to deliver and attach | Slow to build, duplicates pixels, may exceed 4 GiB, and may require BigTIFF | Final delivery after the mosaic, CRS, extent, and compression are settled |
| VRT | Nearly instant; tiny; easy to rebuild; avoids huge intermediate files | Depends on source paths; not a self-contained delivery; legacy CAD may not attach it | QGIS working mosaic and input to later clipping, warping, or translation |

### GeoTIFF creation options I use most

| Option | When it is useful | Watch out for |
|---|---|---|
| `TILED=YES` | Faster random access and zooming; normally preferred for large rasters | Test the exact legacy CAD reader |
| `BLOCKXSIZE=256` / `BLOCKYSIZE=256` | Conservative tiled layout for broad access patterns | Tile dimensions must be divisible by 16; 512×512 can reduce overhead for large modern workflows but uses more data per random read |
| `COMPRESS=JPEG` | Small visual RGB aerials | Lossy; Byte RGB only; not analytical data |
| `JPEG_QUALITY=85` to `95` | Controls aerial JPEG size/quality; 90 is a practical starting point | Higher quality means larger files; it does not make JPEG lossless |
| `COMPRESS=DEFLATE` | Lossless imagery, DEMs, masks, and classes | Compression can be CPU-heavy |
| `COMPRESS=LZW` | Lossless, conservative interoperability | Often larger than DEFLATE |
| `PREDICTOR=2` | Improves lossless compression of integer data | Do not blindly use it for every data type |
| `PREDICTOR=3` | Improves lossless compression of floating-point data | Verify reader compatibility |
| `BIGTIFF=YES` | Output will or may exceed Classic TIFF's approximately 4 GiB limit | Some V8i builds/codecs may reject BigTIFF |
| `BIGTIFF=IF_NEEDED` | Use BigTIFF when GDAL can determine it is required | With compression, final size may not be knowable in advance; `IF_SAFER` or `YES` may be safer for a huge output |
| `NUM_THREADS=ALL_CPUS` | Supported TIFF compression/decompression, especially DEFLATE | Not all codecs or commands use it; GeoTIFF JPEG compression ignores it |
| `SPARSE_OK=YES` | GDAL-only intermediate rasters with large untouched zero/NoData blocks | Produces TIFF sparse blocks that many non-GDAL readers treat as invalid; avoid for V8i/OpenRoads delivery |
| `INTERLEAVE=PIXEL` | Standard organization for RGB imagery | Band-interleaved layouts may be better for some analytical workloads |
| `TFW=YES` | Creates a world-file sidecar for software that needs one | Does not store the CRS |

### How `.tif`, `.ovr`, `.tfw`, `.prj`, and internal georeferencing relate

| File/data | Role |
|---|---|
| `.tif` / `.tiff` | The raster pixels. A GeoTIFF can also embed its affine placement and CRS. This is the file attached in MicroStation/OpenRoads. |
| `.tif.ovr` | Optional lower-resolution overview pyramids. It speeds display but does not replace the TIFF. |
| `.tfw` | Optional six-line affine world file that positions pixels. It normally does not define a CRS. |
| `.prj` | A CRS text sidecar commonly used with shapefiles. A raster `.prj` is not a universal TIFF convention and should not be relied on in place of embedded GeoTIFF CRS metadata. |
| `.aux.xml` | GDAL persistent auxiliary metadata that may contain statistics, georeferencing, NoData, or other metadata. It can take priority when GDAL chooses georeferencing. |

Internal GeoTIFF georeferencing is often sufficient, so a `.tfw` is not always required. When internal tags, `.aux.xml`, and `.tfw` disagree, different applications can choose different sources. Verify the placement reported by `gdalinfo` and test in the target CAD application.

### Raster performance checklist

1. Work from a local SSD rather than a network share or ProjectWise cache while testing.
2. Use a VRT until a physical mosaic is truly needed.
3. Clip to the project area before writing a final massive TIFF.
4. Use tiled GeoTIFF output with 256×256 or 512×512 blocks.
5. Use JPEG for ordinary RGB aerials when lossy compression is acceptable; use DEFLATE/LZW for lossless data.
6. Build external overviews for large deliverable TIFFs.
7. Use BigTIFF deliberately when the estimated output can exceed 4 GiB.
8. Enable `GDAL_NUM_THREADS`, warp threads, and compression threads where the specific command/codec supports them.
9. Avoid unnecessary uncompressed intermediate TIFFs; a VRT can often serve as the intermediate.
10. Keep enough free space for temporary files and the final raster.

## Contents

- [My Most Used OSGeo4W Commands](#my-most-used-osgeo4w-commands)
- [Choosing the Right Raster Output](#choosing-the-right-raster-output)
- [Starting the correct shell](#1-starting-the-correct-shell)
- [Command map](#2-command-map)
- [Inspecting raster files and coordinate systems](#3-inspect-rasters-and-coordinate-systems)
- [Converting and translating TIFFs](#4-convert-copy-resize-and-compress-rasters)
- [Assigning or editing georeferencing metadata](#5-edit-georeferencing-metadata)
- [Building overviews (`.ovr`)](#6-build-overviews-for-faster-display)
- [Reprojecting, resampling, and clipping](#7-reproject-resample-and-clip-rasters)
- [Physical mosaics and VRTs](#8-physical-mosaics-and-vrts)
- [DEM and terrain tools](#9-dem-and-terrain-tools)
- [Raster math and classification](#10-raster-math-and-classification)
- [OGR and vector commands](#11-vector-inspection-and-conversion)
- [PROJ and coordinate transformation commands](#proj-and-coordinate-transformation-commands)
- [Other GDAL and OSGeo4W utilities](#12-other-useful-utilities)
- [Common civil workflow recipes](#13-common-civil-workflow-recipes)
- [Troubleshooting and less-frequent reference material](#14-troubleshooting)
- [Official documentation](#15-official-documentation)

## 1. Starting the correct shell

Use the **OSGeo4W Shell** installed with OSGeo4W or QGIS. It initializes GDAL, PROJ, Python, and their data paths. A normal PowerShell or Command Prompt may say that `gdalinfo` is not recognized even when QGIS has GDAL installed.

Typical ways to start it:

- Windows Start menu: search for **OSGeo4W Shell**.
- QGIS Start menu group: open its OSGeo4W shell shortcut.
- A standalone install commonly has `C:\OSGeo4W\OSGeo4W.bat`.
- A QGIS install commonly has an `OSGeo4W.bat` under `C:\Program Files\QGIS <version>\`.

Verify the environment:

```cmd
where gdalinfo
gdalinfo --version
gdalinfo --formats
ogrinfo --formats
```

Useful command help:

```cmd
gdalinfo --help
gdal_translate --help
gdalwarp --help
ogr2ogr --help
```

This reference uses one-line commands that work in the OSGeo4W `cmd.exe` shell. Always quote Windows paths containing spaces.

### Path placeholders used below

```text
C:\GIS\Input\source.tif
C:\GIS\Output\result.tif
C:\GIS\Vectors\boundary.gpkg
```

Replace them with real paths. The output directory must already exist.

## 2. Command map

| Goal | Primary tool | Important distinction |
|---|---|---|
| Inspect raster properties | `gdalinfo` | Read-only |
| Inspect or validate a CRS | `gdalsrsinfo` | Accepts a dataset or CRS definition |
| Copy, convert, resize, or compress | `gdal_translate` | Does not perform a true CRS reprojection |
| Assign or correct metadata in place | `gdal_edit` / `gdal_edit.py` | Changes metadata, not pixels |
| Create display pyramids | `gdaladdo` | `-ro` forces external `.ovr` overviews |
| Reproject or resample | `gdalwarp` | Transforms the pixel grid |
| Create a lightweight virtual mosaic | `gdalbuildvrt` | VRT references sources; it does not copy pixels |
| Create a physical mosaic | `gdal_translate`, `gdalwarp`, or `gdal_merge` | Produces a new raster |
| Terrain analysis | `gdaldem` | Hillshade, slope, aspect, color relief, TRI, TPI |
| Raster algebra | `gdal_calc` / `gdal_calc.py` | NumPy-style cell calculations |
| Inspect vectors | `ogrinfo` | Read-only unless SQL editing is explicitly used |
| Convert/reproject vectors | `ogr2ogr` | Vector equivalent of several raster workflows |
| Query CRS operations | `projinfo` | Shows candidate transformations, accuracy, grids, and area of use |
| Transform coordinate lists | `cs2cs` | Converts typed or file-based coordinate pairs between CRSs |
| Raster tile index | `gdaltindex` | Creates a vector footprint/index of rasters |
| Vector contours from a DEM | `gdal_contour` | Creates lines or polygons |
| Burn vectors into a raster | `gdal_rasterize` | Rasterizes attributes or fixed values |
| Remove image collars | `nearblack` | Creates cleaned pixels plus an alpha band or mask |

## 3. Inspect rasters and coordinate systems

### `gdalinfo` — inspect a raster

**What it does:** Reports the driver, raster dimensions, band count and type, coordinate system, origin, pixel size, corner coordinates, NoData, color interpretation, compression, tiling, and overview levels.

**When I would use it:** Before any aerial/terrain operation and again after output creation.

**Example — basic inspection:**

```cmd
gdalinfo "C:\GIS\Input\source.tif"
```

Inspect one of the aerials from the original workflow:

```cmd
gdalinfo "C:\Users\cwinstead\Downloads\Aerial-Dorcheat\Aerial-Dorcheat\22_1009815_segment_1_mosaic_visual_V8i.tif"
```

**Important options:**

| Flag | Purpose |
|---|---|
| `-stats` | Compute and store band min/max/mean/stddev. This can be slow on huge rasters. |
| `-approx_stats` | Use overviews or samples when possible for faster approximate statistics. |
| `-mm` | Compute actual minimum and maximum values. |
| `-hist` | Compute a histogram; potentially slow. |
| `-checksum` | Calculate a checksum for each band; useful when comparing copies. |
| `-json` | Return machine-readable JSON. |
| `-proj4` | Also show a PROJ-style CRS string. |
| `-norat` | Suppress raster attribute table output. |
| `-nomd` | Suppress metadata output. |

Focused checks:

```cmd
gdalinfo -approx_stats "C:\GIS\Input\source.tif"
gdalinfo -checksum "C:\GIS\Input\source.tif"
gdalinfo -json "C:\GIS\Input\source.tif"
```

**Watch out for:** Full statistics, histograms, min/max scans, and checksums can be slow on enormous imagery.

What to check before processing:

- `Size is` — width and height in pixels.
- `Coordinate System is` — the declared horizontal CRS.
- `Origin` and `Pixel Size` — placement and ground resolution.
- Corner coordinates — whether the bounds are reasonable for the project.
- Band type — aerial imagery is commonly three `Byte` bands; elevation is often `Int16` or `Float32`.
- `NoData Value` — important for mosaics and clipping.
- `COMPRESSION`, `INTERLEAVE`, and block size — important for performance and compatibility.
- `Overviews` — confirms whether pyramid levels are visible to GDAL.

### `gdalsrsinfo` — decode and validate a CRS

**What it does:** Reports a spatial reference system in WKT, PROJ, EPSG, or other forms.

**When I would use it:** Confirm an aerial or project CRS, validate an EPSG definition, or diagnose legacy WKT compatibility.

**Examples:**

```cmd
gdalsrsinfo "C:\GIS\Input\source.tif"
gdalsrsinfo EPSG:26915
gdalsrsinfo -V EPSG:26915
gdalsrsinfo -o wkt2 EPSG:26915
gdalsrsinfo -o wkt1 EPSG:26915
gdalsrsinfo -e "C:\GIS\Input\source.tif"
```

**Important options:**

| Flag | Purpose |
|---|---|
| `-V` | Validate the CRS definition. |
| `-e` | Search for matching EPSG identifier(s). |
| `-o wkt2` | Output modern WKT2. |
| `-o wkt1` | Output older WKT1, sometimes useful when diagnosing legacy software. |
| `-o proj4` | Output a PROJ-style string. |
| `--single-line` | Put WKT on one line. |

**Watch out for:** Do not choose an EPSG code merely because its name looks close. State Plane systems may have separate meter, international-foot, or US-survey-foot definitions.

## 4. Convert, copy, resize, and compress rasters

### `gdal_translate` — create a converted copy

**What it does:** Changes file format, data type, band selection, compression, tiling, resolution, pixel dimensions, or a rectangular subset. It creates a new dataset.

**When I would use it:** Rewrite an aerial into a CAD-friendly GeoTIFF, make a compressed copy, extract RGB bands, resize, or create a world file.

**Example — simple GeoTIFF copy:**

```cmd
gdal_translate -of GTiff "C:\GIS\Input\source.tif" "C:\GIS\Output\copy.tif"
```

Lossless tiled aerial optimized for general desktop use:

```cmd
gdal_translate -of GTiff -co TILED=YES -co COMPRESS=DEFLATE -co PREDICTOR=2 -co BIGTIFF=IF_SAFER -co INTERLEAVE=PIXEL "C:\GIS\Input\aerial.tif" "C:\GIS\Output\aerial_optimized.tif"
```

Conservative lossless copy for testing in older CAD software:

```cmd
gdal_translate -of GTiff -ot Byte -co COMPRESS=LZW -co INTERLEAVE=PIXEL -co BIGTIFF=IF_NEEDED "C:\GIS\Input\aerial.tif" "C:\GIS\Output\aerial_v8i_test.tif"
```

Create a TIFF world file along with the output:

```cmd
gdal_translate -of GTiff -co TFW=YES "C:\GIS\Input\source.tif" "C:\GIS\Output\source_with_tfw.tif"
```

Resize to 50 percent in both dimensions:

```cmd
gdal_translate -outsize 50% 50% -r average "C:\GIS\Input\source.tif" "C:\GIS\Output\source_half_size.tif"
```

Set a target pixel size in the current CRS units:

```cmd
gdal_translate -tr 1 1 -r average "C:\GIS\Input\source.tif" "C:\GIS\Output\source_1unit_pixels.tif"
```

Extract bands 1, 2, and 3 as an RGB copy:

```cmd
gdal_translate -b 1 -b 2 -b 3 -colorinterp_1 red -colorinterp_2 green -colorinterp_3 blue "C:\GIS\Input\source.tif" "C:\GIS\Output\rgb.tif"
```

Clip by georeferenced bounds without changing CRS:

```cmd
gdal_translate -projwin 500000 3650000 510000 3640000 "C:\GIS\Input\source.tif" "C:\GIS\Output\clip.tif"
```

`-projwin` order is **upper-left X, upper-left Y, lower-right X, lower-right Y**. The values are normally in the dataset CRS.

Create a JPEG-compressed RGB GeoTIFF for a smaller visual-delivery copy:

```cmd
gdal_translate -of GTiff -co TILED=YES -co COMPRESS=JPEG -co JPEG_QUALITY=90 -co PHOTOMETRIC=YCBCR -co INTERLEAVE=PIXEL -co BIGTIFF=IF_SAFER "C:\GIS\Input\aerial_rgb_byte.tif" "C:\GIS\Output\aerial_jpeg90.tif"
```

JPEG-in-TIFF is lossy and is appropriate only for 8-bit visual imagery. Do not use it for DEMs, categorical data, masks, or imagery that must remain pixel-exact.

**Important options:**

| Flag | Purpose |
|---|---|
| `-of GTiff` | Choose the GeoTIFF output driver. |
| `-ot Byte/UInt16/Int16/Float32/...` | Set output pixel data type. |
| `-b n` | Select an input band; repeat for multiple bands. |
| `-r nearest/average/bilinear/cubic/lanczos` | Resampling algorithm when resizing. |
| `-outsize x y` | Set output pixel dimensions or percentages. |
| `-tr xres yres` | Set output ground resolution in CRS units. |
| `-srcwin xoff yoff xsize ysize` | Clip using pixel coordinates. |
| `-projwin ulx uly lrx lry` | Clip using georeferenced coordinates. |
| `-a_srs EPSG:n` | Assign a CRS label; does **not** reproject. |
| `-a_ullr ulx uly lrx lry` | Assign bounds; does **not** warp pixels. |
| `-a_nodata value` | Assign an output NoData value. |
| `-scale` | Scale input values to output values. |
| `-co NAME=VALUE` | Pass a format-specific creation option. |

Useful GeoTIFF creation options:

| Option | Guidance |
|---|---|
| `TILED=YES` | Usually improves random access and zooming. Test with the exact V8i build. |
| `COMPRESS=LZW` | Lossless and broadly compatible. |
| `COMPRESS=DEFLATE` | Lossless; often smaller than LZW and broadly supported. |
| `COMPRESS=JPEG` | Lossy; visual RGB Byte imagery only. |
| `JPEG_QUALITY=85` to `95` | JPEG size/quality tradeoff; 90 is a practical aerial starting point. |
| `PREDICTOR=2` | Often helps integer imagery with LZW/DEFLATE. |
| `PREDICTOR=3` | Floating-point predictor for float rasters. |
| `INTERLEAVE=PIXEL` | Conventional organization for RGB imagery. |
| `BLOCKXSIZE=256` / `BLOCKYSIZE=256` | Conservative tile dimensions; tiled values must be divisible by 16. |
| `BIGTIFF=YES` | Force BigTIFF for a known huge output; test legacy CAD compatibility. |
| `BIGTIFF=IF_NEEDED` | Use BigTIFF only if the uncompressed size requires it. |
| `BIGTIFF=IF_SAFER` | Use BigTIFF when GDAL judges it prudent; useful with compression. |
| `TFW=YES` | Also write a `.tfw` world file. |
| `NUM_THREADS=ALL_CPUS` | Allow supported compression/decompression work to use multiple CPU threads; TIFF JPEG compression ignores it. |
| `SPARSE_OK=YES` | Omit untouched zero/NoData blocks in GDAL-oriented intermediates; avoid for non-GDAL and V8i delivery. |

**Watch out for:** `-a_srs` only assigns a label, `-tr` resamples in the current CRS, and `-ot Byte` can destroy DEM precision. Use `gdalwarp` for true CRS reprojection.

## 5. Edit georeferencing metadata

### `gdal_edit` / `gdal_edit.py` — modify metadata in place

**What it does:** Changes supported raster metadata without creating a new raster grid.

**When I would use it:** The pixel coordinates are correct, but CRS, bounds, NoData, or descriptive metadata needs correction.

Depending on the OSGeo4W/GDAL version, the command may be exposed as `gdal_edit`, `gdal_edit.py`, or a batch wrapper. Check what exists:

```cmd
where gdal_edit
where gdal_edit.py
gdal_edit --help
gdal_edit.py --help
```

**Watch out for:** This utility edits a supported file **in place**. Make a backup first. It does not move or resample pixels.

Assign the correct CRS to a raster whose pixel coordinates are already correct:

```cmd
gdal_edit -a_srs EPSG:26915 "C:\GIS\Input\source.tif"
```

Equivalent on installations using the Python-script name:

```cmd
gdal_edit.py -a_srs EPSG:26915 "C:\GIS\Input\source.tif"
```

Assign known outer bounds:

```cmd
gdal_edit -a_ullr 500000 3650000 510000 3640000 "C:\GIS\Input\source.tif"
```

Set or clear NoData:

```cmd
gdal_edit -a_nodata 0 "C:\GIS\Input\source.tif"
gdal_edit -unsetnodata "C:\GIS\Input\source.tif"
```

Set metadata:

```cmd
gdal_edit -mo "PROJECT=Dorcheat" -mo "SOURCE=Original aerial" "C:\GIS\Input\source.tif"
```

Key distinction:

- Use `-a_srs` when the coordinates are right and only the CRS declaration is missing or wrong.
- Use `gdalwarp -t_srs ...` when coordinates must actually be transformed into another CRS.
- Use `-a_ullr` only when the correct outer bounds are known. It assigns a new geotransform; it does not rectify a rotated, skewed, or distorted image.

### Assign metadata while creating a new copy

To avoid editing the source in place, use `gdal_translate`:

```cmd
gdal_translate -a_srs EPSG:26915 -a_ullr 500000 3650000 510000 3640000 "C:\GIS\Input\unreferenced.tif" "C:\GIS\Output\referenced_copy.tif"
```

## 6. Build overviews for faster display

### `gdaladdo` — build raster pyramids

**What it does:** Builds lower-resolution copies used while zoomed out.

**When I would use it:** Large aerials draw or pan slowly in QGIS, OpenRoads, or MicroStation.

Overviews can dramatically improve display speed for multi-billion-pixel aerials.

Commands used for the two original aerials:

```cmd
gdaladdo -ro -r average "C:\Users\cwinstead\Downloads\Aerial-Dorcheat\Aerial-Dorcheat\22_1009815_segment_1_mosaic_visual_V8i.tif" 2 4 8 16 32 64
gdaladdo -ro -r average "C:\Users\cwinstead\Downloads\Aerial-Dorcheat\Aerial-Dorcheat\22_1009646_segment_0_mosaic_visual_V8i.tif" 2 4 8 16 32 64
```

The `-ro` option opens the source read-only and, for GeoTIFF, forces an external overview file such as:

```text
22_1009815_segment_1_mosaic_visual_V8i.tif.ovr
```

Keep the files together:

```text
22_1009815_segment_1_mosaic_visual_V8i.tif
22_1009815_segment_1_mosaic_visual_V8i.tfw
22_1009815_segment_1_mosaic_visual_V8i.tif.ovr
```

Attach the **`.tif`** in MicroStation/OpenRoads. Do not attach the `.ovr`; compatible software discovers it automatically.

Create compressed external overviews:

```cmd
gdaladdo -ro -r average --config COMPRESS_OVERVIEW DEFLATE --config BIGTIFF_OVERVIEW IF_SAFER "C:\GIS\Input\aerial.tif" 2 4 8 16 32 64
```

Create internal overviews by omitting `-ro`:

```cmd
gdaladdo -r average "C:\GIS\Output\aerial_copy.tif" 2 4 8 16 32 64
```

Remove overviews:

```cmd
gdaladdo -clean "C:\GIS\Output\aerial_copy.tif"
```

For internal TIFF overviews, `-clean` does not reduce the physical TIFF file size. Re-create a clean copy with `gdal_translate` if reclaiming space matters.

Resampling guidance:

| Data | Good overview resampler |
|---|---|
| Aerial/photo imagery | `average`, `gauss`, or `cubic` |
| Continuous elevation | `average` |
| Land-use classes, masks, IDs | `nearest` or `mode` |
| Sharp linework | `nearest`, `bilinear`, or `cubic`, depending on appearance |

Verify overviews:

```cmd
gdalinfo "C:\GIS\Input\aerial.tif"
```

Look for an `Overviews:` line under each band.

## 7. Reproject, resample, and clip rasters

### `gdalwarp` — transform the raster grid

**What it does:** Performs real CRS transformation, alignment, resampling, cutline clipping, or construction of a new raster grid.

**When I would use it:** Move imagery into the project CRS, align it to a known grid, or clip it to a project boundary.

Reproject to a known target CRS:

```cmd
gdalwarp -s_srs EPSG:4326 -t_srs EPSG:26915 -r bilinear -co TILED=YES -co COMPRESS=DEFLATE -co BIGTIFF=IF_SAFER "C:\GIS\Input\source.tif" "C:\GIS\Output\source_26915.tif"
```

If the source CRS is correctly embedded, omit `-s_srs`:

```cmd
gdalwarp -t_srs EPSG:26915 -r bilinear "C:\GIS\Input\source.tif" "C:\GIS\Output\source_26915.tif"
```

Reproject and force a 1-unit target pixel grid:

```cmd
gdalwarp -t_srs EPSG:26915 -tr 1 1 -tap -r cubic -dstnodata 0 -co TILED=YES -co COMPRESS=DEFLATE "C:\GIS\Input\source.tif" "C:\GIS\Output\source_1unit.tif"
```

Clip to a vector boundary:

```cmd
gdalwarp -cutline "C:\GIS\Vectors\boundary.gpkg" -crop_to_cutline -dstalpha -r cubic -co TILED=YES -co COMPRESS=DEFLATE "C:\GIS\Input\aerial.tif" "C:\GIS\Output\aerial_clipped.tif"
```

Clip to target bounds:

```cmd
gdalwarp -te 500000 3640000 510000 3650000 -te_srs EPSG:26915 "C:\GIS\Input\source.tif" "C:\GIS\Output\bounded.tif"
```

`-te` order is **minimum X, minimum Y, maximum X, maximum Y**. This differs from `gdal_translate -projwin` ordering.

Use more CPU threads where supported:

```cmd
gdalwarp -multi -wo NUM_THREADS=ALL_CPUS -wm 1024 -t_srs EPSG:26915 -r cubic -co TILED=YES -co COMPRESS=DEFLATE -co NUM_THREADS=ALL_CPUS "C:\GIS\Input\source.tif" "C:\GIS\Output\warped.tif"
```

**Important options:**

| Flag | Purpose |
|---|---|
| `-s_srs` | Override/declare source CRS for this operation. |
| `-t_srs` | Target CRS. |
| `-r` | Resampling method. |
| `-tr xres yres` | Target resolution in target CRS units. |
| `-tap` | Align output bounds to the target pixel grid. |
| `-te xmin ymin xmax ymax` | Target extent. |
| `-te_srs` | CRS used by the `-te` coordinates. |
| `-srcnodata` / `-dstnodata` | Source and output NoData values. |
| `-cutline` | Vector dataset used for clipping. |
| `-cl layer` | Select the cutline layer. |
| `-crop_to_cutline` | Crop output extent to the cutline. |
| `-dstalpha` | Add an alpha band for transparent areas. |
| `-overwrite` | Permit overwriting an existing destination. Use carefully. |
| `-multi` | Multithread I/O and processing overlap. |
| `-wo NUM_THREADS=ALL_CPUS` | Use multiple threads in the warp operation. |
| `-wm 1024` | Set warp memory limit; suffix-free modern values are MB, but check version help. |

Resampling guidance:

- `nearest`: categorical data, masks, color tables, IDs; preserves original class values.
- `bilinear`: smooth continuous data and moderate-speed aerial resampling.
- `cubic`: good visual aerial output; slower and may overshoot values.
- `lanczos`: sharp visual output; slower and may ring near hard edges.
- `average`: good when reducing resolution.
- `mode`: categorical downsampling.

**Watch out for:** A wrong `-s_srs` produces a wrong transformation. If the source CRS is already embedded correctly, omit `-s_srs`. Use a new output path unless overwrite is explicitly intended.

## 8. Physical mosaics and VRTs

### `gdal_merge.py`, `gdal_merge`, or `gdal_merge.bat` — physical mosaic

**What it does:** Copies adjacent input rasters into one new physical raster.

**When I would use it:** A final, self-contained aerial TIFF is required after the input CRS, resolution, bands, NoData, and overlap behavior are confirmed.

The installed wrapper name varies. Check it:

```cmd
where gdal_merge
where gdal_merge.py
where gdal_merge.bat
```

**Example:** Run the form available in the OSGeo4W shell:

```cmd
gdal_merge.py -o "C:\GIS\Output\merged.tif" -of GTiff -co TILED=YES -co COMPRESS=DEFLATE -co BIGTIFF=IF_SAFER -n 0 -a_nodata 0 "C:\GIS\Input\tile1.tif" "C:\GIS\Input\tile2.tif"
```

or:

```cmd
gdal_merge.bat -o "C:\GIS\Output\merged.tif" -of GTiff -co TILED=YES -co COMPRESS=DEFLATE "C:\GIS\Input\tile1.tif" "C:\GIS\Input\tile2.tif"
```

**Important options:**

| Flag | Purpose |
|---|---|
| `-o filename` | Output file. |
| `-of GTiff` | Output format. |
| `-n value` | Ignore this input value as NoData. |
| `-a_nodata value` | Assign output NoData. |
| `-init value` | Initialize output pixels. |
| `-ps xres yres` | Output pixel size. |
| `-ul_lr ulx uly lrx lry` | Output bounds. |
| `-separate` | Put each input into a separate output band. |
| `-co NAME=VALUE` | Output driver creation option. |

**Watch out for:** This is not a full reprojection/alignment tool. Prepare incompatible inputs with `gdalwarp`. A merge can become huge; use a VRT first and choose BigTIFF/compression deliberately.

### `gdalbuildvrt` — create a virtual mosaic

**What it does:** Writes a small XML dataset that references source rasters rather than copying their pixels.

**When I would use it:** Checking aerial coverage, working with a large tile collection in QGIS, resolving NoData/overlap behavior, or feeding a later `gdalwarp` or `gdal_translate` operation.

A VRT is fast to create. Keep the source rasters in place.

**Examples:**

Build from explicit inputs:

```cmd
gdalbuildvrt -resolution highest -srcnodata 0 -vrtnodata 0 "C:\GIS\Output\aerial_mosaic.vrt" "C:\GIS\Input\tile1.tif" "C:\GIS\Input\tile2.tif"
```

Build from every TIFF in a folder:

```cmd
gdalbuildvrt "C:\GIS\Output\aerial_mosaic.vrt" "C:\GIS\Input\*.tif"
```

Build from a text list containing one raster path per line:

```cmd
gdalbuildvrt -input_file_list "C:\GIS\Input\raster_list.txt" "C:\GIS\Output\aerial_mosaic.vrt"
```

Then create a physical GeoTIFF:

```cmd
gdal_translate -of GTiff -co TILED=YES -co COMPRESS=DEFLATE -co BIGTIFF=IF_SAFER "C:\GIS\Output\aerial_mosaic.vrt" "C:\GIS\Output\aerial_mosaic.tif"
```

**Important options and notes:**

- `-resolution highest`, `lowest`, or `average` controls how differing source resolutions are handled.
- `-srcnodata` defines ignored source values; `-vrtnodata` declares VRT output NoData.
- `-input_file_list` is preferable to an extremely long command line.
- `-separate` places each source in a separate output band rather than mosaicking spatially.
- Overlap priority normally follows input order, with later inputs taking precedence.

**Watch out for:** Inputs should normally share the same CRS. `-allow_projection_difference` does not reproject them; it only ignores the difference. Use `gdalwarp` when tiles need reprojection or grid alignment. Moving or renaming source rasters breaks their VRT references.

For very large mosaics, the VRT-then-translate workflow is usually easier to inspect and troubleshoot.

## 9. DEM and terrain tools

### `gdaldem` — hillshade, slope, aspect, and terrain indices

Hillshade:

```cmd
gdaldem hillshade "C:\GIS\Input\dem.tif" "C:\GIS\Output\hillshade.tif" -az 315 -alt 45 -compute_edges -co COMPRESS=DEFLATE
```

Multidirectional hillshade:

```cmd
gdaldem hillshade "C:\GIS\Input\dem.tif" "C:\GIS\Output\hillshade_multi.tif" -multidirectional -compute_edges -co COMPRESS=DEFLATE
```

Slope in percent:

```cmd
gdaldem slope "C:\GIS\Input\dem.tif" "C:\GIS\Output\slope_percent.tif" -p -compute_edges -co COMPRESS=DEFLATE
```

Slope in degrees:

```cmd
gdaldem slope "C:\GIS\Input\dem.tif" "C:\GIS\Output\slope_degrees.tif" -compute_edges -co COMPRESS=DEFLATE
```

Aspect:

```cmd
gdaldem aspect "C:\GIS\Input\dem.tif" "C:\GIS\Output\aspect.tif" -zero_for_flat -compute_edges -co COMPRESS=DEFLATE
```

Terrain Ruggedness Index, Topographic Position Index, and roughness:

```cmd
gdaldem TRI "C:\GIS\Input\dem.tif" "C:\GIS\Output\tri.tif" -compute_edges -co COMPRESS=DEFLATE
gdaldem TPI "C:\GIS\Input\dem.tif" "C:\GIS\Output\tpi.tif" -compute_edges -co COMPRESS=DEFLATE
gdaldem roughness "C:\GIS\Input\dem.tif" "C:\GIS\Output\roughness.tif" -compute_edges -co COMPRESS=DEFLATE
```

Color relief using a text color table:

```cmd
gdaldem color-relief "C:\GIS\Input\dem.tif" "C:\GIS\Input\elevation_colors.txt" "C:\GIS\Output\color_relief.tif" -alpha -co COMPRESS=DEFLATE
```

Example `elevation_colors.txt`:

```text
0 0 80 0
100 60 140 60
250 180 160 100
500 220 220 220
nv 0 0 0 0
```

### Horizontal and vertical unit warning

Terrain derivatives compare horizontal distance with elevation values. Confirm both independently.

- Reprojecting a DEM changes its horizontal coordinate grid; it does not automatically mean the elevation cell values were converted.
- A CRS in feet does not prove the elevation band is in feet.
- A CRS in meters does not prove the elevation band is in meters.
- For older GDAL versions or ambiguous data, explicitly use the appropriate scale ratio supported by that version. Check `gdaldem --help`.
- Never guess whether the source uses international feet or US survey feet.

## 10. Raster math and classification

### `gdal_calc` / `gdal_calc.py` — pixel-by-pixel calculations

**What it does:** Performs NumPy-style pixel calculations on raster bands.

**When I would use it:** Convert verified elevation units, create masks, classify values, or combine aligned surfaces.

The available command name varies by installation.

```cmd
where gdal_calc
where gdal_calc.py
```

Average two aligned rasters:

```cmd
gdal_calc.py -A "C:\GIS\Input\a.tif" -B "C:\GIS\Input\b.tif" --outfile="C:\GIS\Output\average.tif" --calc="(A.astype(numpy.float32)+B)/2" --type=Float32 --co="COMPRESS=DEFLATE"
```

Convert elevation values from meters to US survey feet:

```cmd
gdal_calc.py -A "C:\GIS\Input\dem_meters.tif" --outfile="C:\GIS\Output\dem_usft.tif" --calc="A*3.280833333333333" --type=Float32 --co="COMPRESS=DEFLATE"
```

Convert elevation values from meters to international feet:

```cmd
gdal_calc.py -A "C:\GIS\Input\dem_meters.tif" --outfile="C:\GIS\Output\dem_ft.tif" --calc="A/0.3048" --type=Float32 --co="COMPRESS=DEFLATE"
```

Create a mask where elevation is at least 100:

```cmd
gdal_calc.py -A "C:\GIS\Input\dem.tif" --outfile="C:\GIS\Output\ge100_mask.tif" --calc="A>=100" --type=Byte --NoDataValue=0 --co="COMPRESS=DEFLATE"
```

Keep values between 100 and 150:

```cmd
gdal_calc.py -A "C:\GIS\Input\dem.tif" --outfile="C:\GIS\Output\range_100_150.tif" --calc="A*logical_and(A>=100,A<=150)" --NoDataValue=0 --co="COMPRESS=DEFLATE"
```

Replace one NoData-like value with another:

```cmd
gdal_calc.py -A "C:\GIS\Input\source.tif" --outfile="C:\GIS\Output\nodata_fixed.tif" --calc="where(A==-9999,-32768,A)" --NoDataValue=-32768 --type=Int16 --co="COMPRESS=DEFLATE"
```

**Watch out for:**

- Inputs should be on the same grid: CRS, extent, pixel size, alignment, and dimensions.
- Traditional `gdal_calc` does not necessarily check projection unless `--projectionCheck` is used.
- Integer arithmetic can truncate or overflow. Cast to `numpy.float32` or `numpy.float64` where appropriate.
- Quote the calculation expression in Windows shells.
- Use `gdalwarp` to align inputs before raster math.

## 11. Vector inspection and conversion

### `ogrinfo` — inspect vector data

**What it does:** Lists vector layers, schemas, feature counts, extents, CRS definitions, and filtered feature information.

**When I would use it:** Confirm a cutline/boundary layer before using it with `gdalwarp` or inspect civil vector deliveries.

**Example — list layers in a GeoPackage:**

```cmd
ogrinfo -ro "C:\GIS\Vectors\project.gpkg"
```

Show a layer summary without printing every feature:

```cmd
ogrinfo -ro -so "C:\GIS\Vectors\project.gpkg" drainage_areas
```

Show summaries for all layers:

```cmd
ogrinfo -ro -al -so "C:\GIS\Vectors\project.gpkg"
```

Filter features by attribute:

```cmd
ogrinfo -ro -al -where "STATUS='ACTIVE'" "C:\GIS\Vectors\project.gpkg" drainage_areas
```

Return JSON where supported:

```cmd
ogrinfo -ro -json "C:\GIS\Vectors\project.gpkg"
```

**Important options:**

| Flag | Purpose |
|---|---|
| `-ro` | Open read-only. |
| `-so` | Summary only. |
| `-al` | Inspect all layers. |
| `-where` | Attribute filter. |
| `-spat xmin ymin xmax ymax` | Spatial filter. |
| `-sql` | Run a SQL query; use cautiously because some drivers allow edits. |
| `-dialect SQLite` | Use the SQLite SQL dialect where supported. |

**Watch out for:** Use `-so` on large layers so every feature is not printed. Keep `-ro` when inspection only is intended.

### `ogr2ogr` — convert, filter, clip, and reproject vectors

**What it does:** Converts, filters, clips, repairs, appends, or reprojects vector datasets.

**When I would use it:** Prepare project boundaries, drainage areas, alignments, or other vectors for QGIS and raster cutlines.

**Example — convert a shapefile to GeoPackage:**

```cmd
ogr2ogr -f GPKG "C:\GIS\Output\boundary.gpkg" "C:\GIS\Input\boundary.shp" -nln boundary
```

Reproject a vector layer:

```cmd
ogr2ogr -f GPKG -t_srs EPSG:26915 "C:\GIS\Output\boundary_26915.gpkg" "C:\GIS\Input\boundary.shp"
```

Clip a vector using another vector dataset:

```cmd
ogr2ogr -f GPKG -clipsrc "C:\GIS\Vectors\clip_boundary.gpkg" "C:\GIS\Output\roads_clipped.gpkg" "C:\GIS\Input\roads.gpkg"
```

Select fields and filter records:

```cmd
ogr2ogr -f GPKG -select "ID,NAME,STATUS" -where "STATUS='ACTIVE'" "C:\GIS\Output\active_features.gpkg" "C:\GIS\Input\features.gpkg"
```

Repair geometries while converting:

```cmd
ogr2ogr -f GPKG -makevalid "C:\GIS\Output\valid.gpkg" "C:\GIS\Input\possibly_invalid.gpkg"
```

**Important options:**

| Flag | Purpose |
|---|---|
| `-f GPKG` | Choose output driver. |
| `-s_srs` / `-t_srs` | Override source CRS / set target CRS. |
| `-nln name` | Output layer name. |
| `-where` | Attribute filter. |
| `-select` | Retain selected fields. |
| `-clipsrc` / `-clipdst` | Clip features. |
| `-spat xmin ymin xmax ymax` | Select features intersecting a box. |
| `-makevalid` | Attempt to repair invalid geometry. |
| `-append` | Append to an existing layer. |
| `-overwrite` | Replace an existing output layer. Use carefully. |

**Watch out for:** `-overwrite` and `-append` change destination content. Prefer a new output while testing and confirm the source layer CRS before reprojecting.

## PROJ and coordinate transformation commands

GDAL uses the PROJ library for coordinate reference systems and transformations. These utilities help diagnose the exact operation, required datum grids, accuracy, and coordinate-axis behavior before a raster or vector reprojection.

### `projinfo` — inspect CRSs and candidate transformations

**What it does:** Queries the PROJ database for a CRS definition or for available coordinate operations between source and target CRSs.

**When I would use it:** Before transforming survey, State Plane, UTM, or datum-sensitive project data; especially when more than one transformation may exist.

**Examples:**

Inspect a CRS:

```bat
projinfo EPSG:26915
```

Summarize candidate transformations:

```bat
projinfo -s EPSG:SOURCE -t EPSG:TARGET --summary
```

Restrict the search to a project area. The bounding box is west longitude, south latitude, east longitude, north latitude in degrees:

```bat
projinfo -s EPSG:SOURCE -t EPSG:TARGET --summary --bbox WEST,SOUTH,EAST,NORTH
```

Check PROJ data search paths and remote-grid availability:

```bat
projinfo --searchpaths
projinfo --remote-data
```

**Important options:**

| Option | Meaning |
|---|---|
| `-s` | Source CRS. |
| `-t` | Target CRS. |
| `--summary` | List candidate operations compactly with accuracy and area of use. |
| `--bbox` | Restrict operations to the project’s geographic area of interest. |
| `--area` | Restrict by a named or coded area of use. |
| `--grid-check` | Control handling/ranking of transformations needing grids. |
| `--hide-ballpark` | Exclude approximate ballpark operations from the results. |
| `-o WKT2:2019` | Request a particular definition format. |
| `--identify` | Look for known database objects matching a supplied definition. |

**Watch out for:** The first listed transformation is not automatically suitable everywhere. Check its accuracy, area of use, and whether required grids are installed. Do not silently accept a ballpark datum shift for survey-grade work.

### `cs2cs` — transform coordinate pairs

**What it does:** Converts coordinate pairs or coordinate files between two CRSs, including projection and datum transformations.

**When I would use it:** Spot-check a known control point before committing to a full raster/vector reprojection or transform a short coordinate list.

**Example:**

Interactive mode:

```bat
cs2cs -f "%.3f" EPSG:SOURCE EPSG:TARGET
```

Pipe a coordinate pair:

```bat
echo X Y | cs2cs -f "%.3f" EPSG:SOURCE EPSG:TARGET
```

Example from WGS 84 to a projected CRS using EPSG axis order:

```bat
echo 33 -93 | cs2cs -f "%.3f" EPSG:4326 EPSG:26915
```

With a recent PROJ version, require the best known operation and reject ballpark transformations:

```bat
echo X Y | cs2cs --only-best --no-ballpark -f "%.3f" EPSG:SOURCE EPSG:TARGET
```

**Important options:**

| Option | Meaning |
|---|---|
| `-f "%.3f"` | Print numeric output to three decimal places. |
| `-d n` | Round output to a selected number of decimals on supported versions. |
| `-I` | Run the inverse transformation. |
| `-r` | Reverse the order of the first two input values. |
| `-s` | Reverse the order of the first two output values. |
| `--area` / `--bbox` | Restrict the transformation search to an area of interest. |
| `--only-best` | Require the most accurate known operation; recent PROJ versions only. |
| `--no-ballpark` | Reject approximate ballpark transformations. |

**Watch out for:** EPSG axis order is enforced. For `EPSG:4326`, modern PROJ expects latitude then longitude unless the order is explicitly reversed. Confirm the input/output order with a known point. Missing datum grids can change accuracy or cause `--only-best` to fail, which is preferable to silently using a poor approximation for high-accuracy work.

### `gdaltransform` — GDAL coordinate spot checks

`gdaltransform` provides a similar interactive check using GDAL dataset/CRS handling:

```bat
gdaltransform -s_srs EPSG:SOURCE -t_srs EPSG:TARGET
```

Type `X Y` pairs. In Windows Command Prompt, press `Ctrl+Z`, then Enter, to end input. The older detailed example remains in [Other useful utilities](#12-other-useful-utilities).

## 12. Other useful utilities

### `gdaltindex` — create a raster footprint index

**What it does:** Creates a vector layer containing raster extents and source paths.

**When I would use it:** Locate aerial tiles, map delivery coverage, or manage a large raster collection.

```cmd
gdaltindex -f GPKG -t_srs EPSG:26915 "C:\GIS\Output\aerial_index.gpkg" "C:\GIS\Input\*.tif"
```

### `gdal_contour` — make contours from a DEM

**What it does:** Converts DEM values into vector contour lines or polygons.

**When I would use it:** Create preliminary terrain contours for GIS review. Survey/design requirements may demand a more controlled terrain workflow.

**Example — create 1-unit interval contours with an elevation attribute:**

```cmd
gdal_contour -a ELEV -i 1 -snodata -9999 -f GPKG "C:\GIS\Input\dem.tif" "C:\GIS\Output\contours.gpkg"
```

Create fixed contour levels:

```cmd
gdal_contour -a ELEV -fl 100 110 120 130 140 150 -f GPKG "C:\GIS\Input\dem.tif" "C:\GIS\Output\fixed_contours.gpkg"
```

**Watch out for:** The interval and elevation values use the DEM band’s vertical value units, not automatically the horizontal CRS units.

### `gdal_rasterize` — burn vector features into a raster

**What it does:** Burns vector geometry or attribute values into a new or existing raster grid.

**When I would use it:** Create a project-boundary mask, classification raster, or analysis grid from civil vector data.

**Example — burn a fixed value of 1, aligned to a known grid:**

```cmd
gdal_rasterize -burn 1 -ot Byte -a_nodata 0 -tr 1 1 -te 500000 3640000 510000 3650000 -a_srs EPSG:26915 -co TILED=YES -co COMPRESS=DEFLATE "C:\GIS\Vectors\boundary.gpkg" "C:\GIS\Output\boundary_mask.tif"
```

Burn values from an attribute:

```cmd
gdal_rasterize -a CLASS_ID -ot Int16 -a_nodata 0 -tr 1 1 -tap -co COMPRESS=DEFLATE "C:\GIS\Vectors\classes.gpkg" "C:\GIS\Output\classes.tif"
```

**Watch out for:** Choose pixel size, extent, alignment, data type, and NoData deliberately. `-tap` requires a target resolution, and overlapping features may need `-at` or SQL/order preparation depending on the intended result.

### `gdaltransform` — transform coordinate pairs

Transform typed X/Y pairs between coordinate systems:

```cmd
gdaltransform -s_srs EPSG:4326 -t_srs EPSG:26915
```

After the command starts, type a pair such as:

```text
-93.0 33.0
```

Press `Ctrl+Z`, then Enter, to end interactive input in Windows Command Prompt.

### `gdalmanage` — identify, copy, rename, or delete datasets

Identify the raster driver:

```cmd
gdalmanage identify "C:\GIS\Input\source.tif"
```

Prefer normal File Explorer operations for ordinary file management. Dataset-aware copy/rename can be useful when a format has required sidecar files, but inspect the command help and target carefully before mutation.

### `nearblack` — clean dark or light image collars

**What it does:** Finds near-black, near-white, or chosen-color image collars and can create an alpha band or mask.

**When I would use it:** Remove black wedges/collars around rectified aerial imagery before mosaicking or display.

**Example — remove a near-black border by adding an alpha band:**

```cmd
nearblack -of GTiff -setalpha -near 15 -o "C:\GIS\Output\aerial_cleaned.tif" "C:\GIS\Input\aerial_with_black_border.tif"
```

Use only after visually confirming that genuine dark content near the image edge will not be removed.

### `gdal_fillnodata` / `gdal_fillnodata.py` — interpolate small gaps

```cmd
gdal_fillnodata.py -md 10 -si 1 "C:\GIS\Input\dem_with_small_holes.tif" "C:\GIS\Output\dem_filled.tif"
```

This interpolates from surrounding pixels. It is not a substitute for valid survey/elevation data and should be documented when used.

### `gdal_viewshed` — visibility from an observer point

```cmd
gdal_viewshed -ox 505000 -oy 3645000 -oz 2 -tz 0 -md 5000 "C:\GIS\Input\dem.tif" "C:\GIS\Output\viewshed.tif"
```

Coordinates and distance parameters must match the intended horizontal units. Confirm vertical units separately.

## 13. Common civil-workflow recipes

### Recipe A — inspect a slow aerial before changing anything

```cmd
gdalinfo "C:\GIS\Input\aerial.tif"
gdalsrsinfo -e "C:\GIS\Input\aerial.tif"
```

Record:

- dimensions and band types;
- CRS/EPSG match, if any;
- origin, pixel size, and corners;
- internal compression and block size;
- existing overviews;
- file size in File Explorer.

### Recipe B — speed up a huge aerial without altering its TIFF

```cmd
gdaladdo -ro -r average --config COMPRESS_OVERVIEW DEFLATE --config BIGTIFF_OVERVIEW IF_SAFER "C:\GIS\Input\aerial.tif" 2 4 8 16 32 64
gdalinfo "C:\GIS\Input\aerial.tif"
```

Keep `aerial.tif.ovr` beside `aerial.tif`. Attach `aerial.tif`, not the `.ovr`.

### Recipe C — make a lossless working copy and add overviews

```cmd
gdal_translate -of GTiff -co TILED=YES -co COMPRESS=DEFLATE -co PREDICTOR=2 -co BIGTIFF=IF_SAFER -co INTERLEAVE=PIXEL -co TFW=YES "C:\GIS\Input\aerial.tif" "C:\GIS\Output\aerial_working.tif"
gdaladdo -ro -r average --config COMPRESS_OVERVIEW DEFLATE "C:\GIS\Output\aerial_working.tif" 2 4 8 16 32 64
gdalinfo "C:\GIS\Output\aerial_working.tif"
```

### Recipe D — mosaic many same-CRS tiles, then optimize

```cmd
gdalbuildvrt -srcnodata 0 -vrtnodata 0 "C:\GIS\Output\mosaic.vrt" "C:\GIS\Input\*.tif"
gdal_translate -of GTiff -co TILED=YES -co COMPRESS=DEFLATE -co BIGTIFF=IF_SAFER "C:\GIS\Output\mosaic.vrt" "C:\GIS\Output\mosaic.tif"
gdaladdo -ro -r average --config COMPRESS_OVERVIEW DEFLATE "C:\GIS\Output\mosaic.tif" 2 4 8 16 32 64
gdalinfo "C:\GIS\Output\mosaic.tif"
```

### Recipe E — reproject a mosaic to the project grid

```cmd
gdalwarp -t_srs EPSG:26915 -tr 1 1 -tap -r cubic -srcnodata 0 -dstnodata 0 -multi -wo NUM_THREADS=ALL_CPUS -co TILED=YES -co COMPRESS=DEFLATE -co BIGTIFF=IF_SAFER "C:\GIS\Output\mosaic.vrt" "C:\GIS\Output\mosaic_project_grid.tif"
gdaladdo -ro -r average "C:\GIS\Output\mosaic_project_grid.tif" 2 4 8 16 32 64
```

Replace the example EPSG code and 1-unit resolution with the verified project requirements.

### Recipe F — split a huge image into manageable tiles

`gdal_retile.py` may be available with the GDAL Python utilities:

```cmd
gdal_retile.py -ps 10000 10000 -overlap 0 -levels 0 -useDirForEachRow -targetDir "C:\GIS\Output\tiles" "C:\GIS\Input\huge_aerial.tif"
```

The pixel tile size above is an example. Confirm output naming, sidecars, and georeferencing with a small test before processing a large delivery.

## 14. Troubleshooting

### Command is not recognized

1. Open the OSGeo4W Shell supplied by OSGeo4W or QGIS.
2. Run `where gdalinfo` and `gdalinfo --version`.
3. For Python utilities, try both names, such as `gdal_calc` and `gdal_calc.py`.
4. Do not copy individual GDAL executables out of their installation; they rely on matching libraries and data directories.

### CRS is missing or wrong

- Inspect the file with `gdalinfo` and `gdalsrsinfo -e`.
- If the coordinates are already correct and only the declaration is missing, assign it with `gdal_edit -a_srs` or create a copy with `gdal_translate -a_srs`.
- If the dataset must move to a different coordinate system, use `gdalwarp -t_srs`.
- `-a_srs` relabels; it does not transform.
- Do not guess a State Plane zone, datum, or unit variant from approximate coordinates alone.

### Feet versus meters

Treat these as separate questions:

1. What units does the **horizontal CRS** use?
2. What units do the **pixel values/elevations** use?

Useful exact relationships:

```text
1 international foot = 0.3048 meter exactly
1 US survey foot = 1200 / 3937 meter
1 meter = 3.280839895013123 international feet
1 meter = 3.280833333333333 US survey feet
```

Reprojection handles horizontal coordinates. It does not automatically convert a plain elevation band merely because the target horizontal CRS uses different units. Verify delivery metadata before converting elevations.

### GeoTIFF versus world file

- A GeoTIFF can store CRS and geotransform internally.
- A `.tfw` stores only six affine placement numbers; it does **not** identify the CRS.
- Keep a world file beside the image with the matching base filename.
- GDAL may prefer internal GeoTIFF or `.aux.xml` georeferencing over a world file. `gdalinfo` shows the georeferencing GDAL actually selected.
- Use `-co TFW=YES` with `gdal_translate` to generate a world file for the output.
- Attach the image file in CAD, not the `.tfw`.

A world file contains:

```text
line 1: X pixel size
line 2: row rotation
line 3: column rotation
line 4: Y pixel size, usually negative for north-up imagery
line 5: X coordinate of the center of the upper-left pixel
line 6: Y coordinate of the center of the upper-left pixel
```

### Overviews are not being used

- Ensure `image.tif.ovr` is beside `image.tif` and retains the exact name.
- Attach/open `image.tif`, never the `.ovr` directly.
- Run `gdalinfo image.tif` and look for `Overviews:`.
- Close and reopen the raster after generating overviews; some applications cache the open dataset.
- Test on a local SSD before blaming the raster. Network and ProjectWise latency can dominate performance.
- External overviews add disk space but preserve the source TIFF.
- Software outside GDAL is not guaranteed to support every external overview layout or compression. Test the exact V8i installation.

### Compression choices

- **LZW:** lossless, conservative, broadly compatible.
- **DEFLATE:** lossless, often compact, broadly supported.
- **JPEG:** lossy and compact for 8-bit RGB aerial imagery; not for elevation or categorical data.
- **ZSTD/WEBP/JXL:** useful in modern GDAL workflows but less suitable when old CAD compatibility is the priority.
- Compression reduces storage and I/O but adds decoding work. Overviews and tiling often matter more for interactive display.
- If a compressed `gdalwarp` output becomes unexpectedly large or slow, warp to a VRT or temporary uncompressed raster and use `gdal_translate` for final compression.

### Classic TIFF, BigTIFF, and V8i compatibility

- Classic TIFF has a roughly 4 GiB structural limit.
- Huge imagery may require BigTIFF, especially after expansion or overview creation.
- Older MicroStation V8i builds or raster codecs may not accept every BigTIFF, tiled layout, compression codec, alpha band, or newer GeoTIFF CRS encoding.
- There is no single guaranteed V8i recipe across all builds. Test one representative output before processing the entire delivery.
- Conservative starting point: 8-bit RGB, pixel interleave, LZW or DEFLATE, no unusual codec, a `.tfw`, and external overviews.
- If the needed image would exceed Classic TIFF and V8i rejects BigTIFF, tile it into multiple Classic TIFFs instead of forcing an oversized file.
- Do not reduce or recompress the only source copy merely to accommodate V8i.

### Image loads but is in the wrong place

Check, in this order:

1. The design file/project CRS.
2. Raster CRS, origin, pixel size, rotation, and corner coordinates in `gdalinfo`.
3. Whether internal GeoTIFF placement conflicts with `.tfw` or `.aux.xml` placement.
4. The drawing’s working units and geographic coordinate system assignment.
5. Whether the file was merely relabeled with `-a_srs` when it actually needed `gdalwarp`.

### Output is black, washed out, or transparent

- Run `gdalinfo -approx_stats` and inspect band ranges and NoData.
- Confirm the correct bands and data type.
- Do not set `0` as NoData if black pixels are legitimate imagery.
- Use `-scale` only after inspecting source ranges.
- Check whether an alpha band or mask is hiding valid pixels.

### Processing is extremely slow

- Work on a local SSD.
- Build overviews for viewing.
- Clip to the needed project area.
- Use tiled output.
- Use `-multi`, `-wo NUM_THREADS=ALL_CPUS`, or `-co NUM_THREADS=ALL_CPUS` where supported.
- Avoid `-stats`, `-hist`, or checksums on billion-pixel files unless needed.
- Use a VRT to test a mosaic before writing a huge physical TIFF.
- Ensure enough free disk space for both temporary and final outputs.

## 15. Official documentation

- [GDAL command-line program index](https://gdal.org/en/stable/programs/index.html)
- [`gdalinfo`](https://gdal.org/en/stable/programs/gdalinfo.html)
- [`gdal_translate`](https://gdal.org/en/stable/programs/gdal_translate.html)
- [`gdal_edit`](https://gdal.org/en/stable/programs/gdal_edit.html)
- [`gdaladdo`](https://gdal.org/en/stable/programs/gdaladdo.html)
- [`gdalwarp`](https://gdal.org/en/stable/programs/gdalwarp.html)
- [`gdalbuildvrt`](https://gdal.org/en/stable/programs/gdalbuildvrt.html)
- [`gdal_merge`](https://gdal.org/en/stable/programs/gdal_merge.html)
- [`gdaldem`](https://gdal.org/en/stable/programs/gdaldem.html)
- [`gdal_calc`](https://gdal.org/en/stable/programs/gdal_calc.html)
- [`ogrinfo`](https://gdal.org/en/stable/programs/ogrinfo.html)
- [`ogr2ogr`](https://gdal.org/en/stable/programs/ogr2ogr.html)
- [`gdalsrsinfo`](https://gdal.org/en/stable/programs/gdalsrsinfo.html)
- [`gdal_rasterize`](https://gdal.org/en/stable/programs/gdal_rasterize.html)
- [`gdal_contour`](https://gdal.org/en/stable/programs/gdal_contour.html)
- [`gdaltindex`](https://gdal.org/en/stable/programs/gdaltindex.html)
- [`nearblack`](https://gdal.org/en/stable/programs/nearblack.html)
- [GeoTIFF driver and creation options](https://gdal.org/en/stable/drivers/raster/gtiff.html)
- [`projinfo`](https://proj.org/en/stable/apps/projinfo.html)
- [`cs2cs`](https://proj.org/en/stable/apps/cs2cs.html)

### Version note

GDAL utilities evolve. Starting with GDAL 3.11, GDAL also provides a newer unified `gdal` command with subcommands, but the traditional commands documented here remain common in QGIS/OSGeo4W and are often easier to match with existing civil-production procedures. Run `<command> --help` on the actual workstation before relying on a newer flag.
