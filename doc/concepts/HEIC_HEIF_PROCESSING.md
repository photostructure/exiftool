# HEIC/HEIF Processing in ExifTool

**Status**: Research Complete  
**Last Updated**: July 2025  
**Source Analysis**: Based on ExifTool 12.97+ (submodule commit in exif-oxide)

## Overview

This document provides a comprehensive analysis of how ExifTool processes HEIC (High Efficiency Image Container) and HEIF (High Efficiency Image Format) files. This research supports the implementation of HEIC/HEIF support in exif-oxide.

## Key Findings

### Architecture Decision
ExifTool does **not** have a dedicated HEIF.pm module. Instead, HEIF/HEIC files are processed entirely through the **QuickTime.pm** module, treating them as variants of the QuickTime/MP4 container format family. This approach is architecturally sound because HEIF is based on the ISO Base Media File Format (ISO/IEC 14496-12), the same foundation as MP4.

### Processing Pipeline

```
File Detection (ftyp atom) 
    ↓
QuickTime.pm Processing
    ↓
Item-Based Metadata Extraction (HandleItemInfo)
    ↓
Binary Data Location (iloc/mdat/idat)
    ↓
Format-Specific Processing (EXIF/XMP/Apple)
```

## File Type Detection

**Location**: `third-party/exiftool/lib/Image/ExifTool/QuickTime.pm` lines 119-126, 227-231

### Major Brands Recognition
```perl
%mimeLookup = (
    HEIC => 'image/heic',           # Single image
    HEVC => 'image/heic-sequence',  # Image sequence
    HEICS=> 'image/heic-sequence',  # Image sequence
);

%ftypLookup = (
    'heic' => 'High Efficiency Image Format HEVC still image (.HEIC)',
    'hevc' => 'High Efficiency Image Format HEVC sequence (.HEICS)', 
    'mif1' => 'High Efficiency Image Format still image (.HEIF)',
    'msf1' => 'High Efficiency Image Format sequence (.HEIFS)',
    'heix' => 'High Efficiency Image Format still image (.HEIF)', # Canon variant
);
```

### Detection Algorithm
1. **Read ftyp atom** (first 20+ bytes of file)
2. **Extract major brand** (bytes 8-11 after ftyp size/type)
3. **Lookup in ftypLookup table** for file type determination
4. **Set FileType** to 'HEIC' for processing route
5. **Route through QuickTime processing pipeline**

## Container Structure

HEIC/HEIF files use the ISO Base Media File Format (ISOBMFF) atom/box structure:

```
File Structure:
├── ftyp (File Type - heic/heif/mif1/msf1/heix)
├── meta (Metadata Container)
│   ├── hdlr (Handler - 'pict' for images, 'vide' for video)
│   ├── pitm (Primary Item Reference - identifies main image)
│   ├── iinf (Item Information - list of all items)
│   │   └── infe (Item Information Entry - per item)
│   ├── iloc (Item Location - offsets and sizes)
│   ├── iprp (Item Properties - image properties)
│   │   ├── ipco (Item Property Container)
│   │   └── ipma (Item Property Association)
│   └── iref (Item References - relationships between items)
├── mdat (Media Data - actual image/video bytes)
├── idat (Item Data - HEIC-specific image data)
└── uuid (Universal Unique Identifier - XMP and other metadata)
```

## Item-Based Architecture

**Key Innovation**: Unlike traditional QuickTime files that use track-based media storage, HEIC uses an **item-based system** where each image or metadata block is an "item" with a unique ID.

### Core Components

#### 1. Item Information (iinf/infe)
- **Purpose**: Catalog of all items in the file
- **Structure**: List of Item Information Entries (infe)
- **Content**: Item ID, item type, item name, content type

#### 2. Item Location (iloc)
**Processing Function**: `ParseItemLocation()` at QuickTime.pm:9014

```perl
# Item location structure
{
    ConstructionMethod => 0,  # 0=file offset, 1=idat, 2=item reference
    BaseOffset => 0,          # Base address for calculations
    Extents => [              # Array of data fragments
        [index, offset, length, size_bytes, position]
    ]
}
```

#### 3. Primary Item (pitm)
**Location**: QuickTime.pm:2818-2827
```perl
pitm => [{
    Name => 'PrimaryItemReference',
    RawConv => '$$self{PrimaryItem} = unpack("x4N",$val)',
}]
```

## Processing Functions

### HandleItemInfo() - Core Processing
**Location**: QuickTime.pm:9226+
**Purpose**: Extracts metadata and binary data from HEIC items

**Algorithm**:
1. **Iterate through all items** in `$$et{ItemInfo}`
2. **Identify primary item** using `$$et{PrimaryItem}`
3. **Locate item data** using iloc extents
4. **Assemble binary data** from multiple fragments
5. **Detect content type** (EXIF, XMP, JPEG, etc.)
6. **Process metadata** using appropriate parsers

### ParseItemLocation() - Binary Location
**Location**: QuickTime.pm:9014
**Purpose**: Parse iloc atom to determine item data locations

**Key Data Structures**:
- **ConstructionMethod**: Determines how data is stored
- **BaseOffset**: Starting point for offset calculations  
- **Extents**: Array of data fragments with positions and sizes

## Special HEIC Handling

### 1. Construction Methods
Different ways items can be stored:
- **Method 0**: Direct file offset (traditional)
- **Method 1**: Data in idat container (HEIC-specific)
- **Method 2**: Item reference (derived data)

### 2. idat vs mdat Distinction
**Critical Code**: QuickTime.pm:9266
```perl
# Note: In HEIC's, these seem to indicate data in 'idat' instead of 'mdat'
my $constMeth = $$item{ConstructionMethod} || 0;
```

HEIC files often use `idat` (item data) containers instead of traditional `mdat` (media data) containers.

### 3. Multi-Document Processing
**Location**: QuickTime.pm:9399-9405
For files with multiple images (burst photos, Live Photos):
- **Document Numbers**: `$$et{DOC_NUM}` organizes multiple images
- **Property Inheritance**: Rules for sharing metadata across items
- **Reference Tracking**: Complex system for derived and related images

## Performance Optimizations

### Fast Scan Mode
**Location**: QuickTime.pm:10046
```perl
# stop processing at mdat/idat if -fast2 is used
last if $fast > 1 and ($tag eq 'mdat' or ($tag eq 'idat' and $$et{FileType} ne 'HEIC'));
```

ExifTool has special fast-scan handling for HEIC files to avoid processing large image data when only metadata is needed.

### Memory Management
- **Streaming reads** for large image data
- **Extent-based processing** to handle fragmented data
- **Selective extraction** based on requested metadata groups

## Integration Points for exif-oxide

### 1. Detection Integration
- Implement ftyp atom parsing in `src/formats/mod.rs`
- Add HEIC/HEIF major brand recognition
- Route to QuickTime processor infrastructure

### 2. Container Processing
- Leverage existing atom/box parsing if available
- Implement item-based metadata extraction
- Handle iloc/mdat/idat data location logic

### 3. Metadata Processing
- Extract EXIF from item containers
- Process XMP from UUID boxes
- Handle Apple mdta metadata

### 4. Binary Extraction
- Implement iloc parsing for PhotoStructure use case
- Handle primary item selection
- Support multi-image files (burst, Live Photos)

## References

**Primary Source**: `/home/mrm/src/exif-oxide/third-party/exiftool/lib/Image/ExifTool/QuickTime.pm`
- Lines 119-126: MIME type mapping
- Lines 227-231: File type descriptions
- Lines 2770-3021: HEIF-specific atom definitions
- Lines 9014+: ParseItemLocation function
- Lines 9226+: HandleItemInfo function

**Standards**:
- ISO/IEC 23008-12:2017 (HEIF specification)
- ISO/IEC 14496-12 (ISO Base Media File Format)
- Nokia HEIF Technical Documentation: https://nokiatech.github.io/heif/technical.html

**Related Documentation**:
- `HEIC_IMAGE_VIDEO_DETECTION.md` - Detection algorithms
- `HEIC_METADATA_EXTRACTION.md` - Metadata processing
- `HEIC_BINARY_EXTRACTION.md` - Binary data extraction