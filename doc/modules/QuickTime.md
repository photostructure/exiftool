# QuickTime.pm Module Outline

**ExifTool Version:** 13.26  
**Module Version:** 2.94  
**Document Version:** 1.0  
**Last Updated:** 2025-06-29

## Overview

The QuickTime module handles the QuickTime/MP4 container format family, including MOV, MP4, M4V, 3GP, HEIC, AVIF, and 29 other variants. It's one of ExifTool's most complex modules, supporting hierarchical atom structures, multiple media types, and extensive metadata systems.

## Module Structure

### Tag Tables (86 total)

**Core Structural Tables:**
- **%QuickTime::Main** - Root atoms (ftyp, moov, mdat, etc.)
- **%QuickTime::Movie** - Movie container (mvhd, trak, etc.)
- **%QuickTime::Track** - Track-level metadata
- **%QuickTime::Media** - Media information
- **%QuickTime::SampleTable** - Sample timing/description

**Metadata Tables:**
- **%QuickTime::UserData** - Classic QuickTime metadata
- **%QuickTime::Meta** - Modern metadata container
- **%QuickTime::ItemList** - iTunes-style tags
- **%QuickTime::Keys** - Dynamic key definitions

**Codec Tables:**
- **%QuickTime::HEVCConfig** - HEVC/H.265 configuration
- **%QuickTime::AV1Config** - AV1 codec parameters
- **%QuickTime::VisualSampleDesc** - Video codec info
- **%QuickTime::AudioSampleDesc** - Audio codec info

### Atom/Box Structure

QuickTime uses nested atoms (boxes) with:
```
[4-byte size][4-byte type][variable data]
```
- Size can be extended to 64-bit for large atoms
- Data may contain sub-atoms recursively
- Context-sensitive processing based on parent atoms

## Processing Flow

### 1. Main Processing Loop
```perl
ProcessMOV($$$)
  ├── Read atom header (size + type)
  ├── Handle extended sizes
  ├── Dispatch to handler based on type
  ├── Process sub-atoms recursively
  └── Track handler context
```

### 2. Context Management
- **HandlerType**: vide, soun, text, meta, etc.
- **MediaType**: Video, Audio, Text, etc.
- **MetaFormat**: Format of metadata atoms
- Context determines tag interpretation

### 3. Special Processing Functions

**ProcessHybrid($$$)**
- Handles atoms with both data and sub-atoms
- Common in sample descriptions
- Extracts codec parameters

**HandleItemInfo($$$)**
- HEIC/AVIF item-based images
- Multiple images in single file
- Image property associations

**ProcessSampleDesc($$$)**
- Codec-specific information
- Video/audio parameters
- Format detection

## Key Features

### Multi-Format Support

**29 File Types via ftyp:**
- MP4 variants: mp41, mp42, isom
- Apple: qt, M4V, M4A, M4B
- 3GPP: 3gp4, 3gp5, 3gp6
- Modern: heic, avif, crx
- Camera: XAVC, dji, insta360

### GPS Track Extraction

**Supported Formats:**
- Novatek dashcams
- Garmin VIRB
- Pittasoft BlackVue
- 70mai dashcams
- Nextbase systems

**GPS Features:**
- Time-synchronized coordinates
- Speed and direction
- Satellite information
- Requires ExtractEmbedded option

### Timed Metadata

**ExtractEmbedded Processing:**
```perl
%eeBox = (
    vide => { ... },  # Video frames
    text => { ... },  # Subtitles
    meta => { ... },  # Metadata stream
    camm => { ... },  # Camera motion
);
```

### Modern Image Formats

**HEIC/HEIF Support:**
- Multiple images per file
- Image sequences
- Derived images
- Apple Live Photos

**AVIF Features:**
- AV1-based compression
- HDR metadata
- Alpha channels
- Color profiles

## Unique Patterns

### Handler Type Switching
```perl
# Context changes interpretation
if ($$et{HandlerType} eq 'vide') {
    # Video-specific processing
} elsif ($$et{HandlerType} eq 'soun') {
    # Audio-specific processing
}
```

### Dynamic Metadata
```perl
# Keys atom defines custom tags
if ($tag eq 'keys') {
    LoadKeysTable($et, $dataPt, $dirStart, $dirLen);
}
```

### Matrix Transformations
```perl
# Video rotation/orientation
MatrixConv => q{
    my @a = split ' ', $val;
    $_ /= 0x10000 foreach @a[0,1,3,4];
    $_ /= 0x40000000 foreach @a[6,7];
    "@a";
}
```

## Container Complexity

### Hierarchical Structure
- Unlimited nesting depth
- Context-sensitive atoms
- Cross-references between atoms
- Optional atom ordering

### Media Multiplexing
- Multiple tracks per file
- Different media types
- Synchronization data
- Edit lists for timing

### Extensibility
- Custom atoms allowed
- Manufacturer extensions
- UUID-based atoms
- Free-form metadata

## Comparison with Other Formats

| Feature | QuickTime | AVI | MKV |
|---------|-----------|-----|-----|
| Structure | Hierarchical | Linear | EBML |
| Complexity | Very High | Low | High |
| Metadata | Extensive | Limited | Good |
| Codecs | Many | Limited | Any |
| GPS Support | Yes | No | Possible |

## Usage Notes

- Large file support (64-bit atoms)
- Fast scan mode available
- ExtractEmbedded for GPS/timed data
- Validate option for structure checking
- Some atoms require specific options

## Debugging Tips

- Use `-v3` to see atom hierarchy
- `-htmlDump` shows structure visually
- Check HandlerType for context
- GPS extraction needs ExtractEmbedded
- Large files may have 64-bit atoms