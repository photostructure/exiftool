# HEIC Metadata Extraction in ExifTool

**Status**: Research Complete  
**Last Updated**: July 2025  
**Critical for**: Comprehensive metadata extraction from HEIC/HEIF files

## Overview

This document details ExifTool's approach to extracting metadata from HEIC files. Unlike traditional formats where metadata has fixed locations, HEIC stores metadata in multiple locations using different container formats: item-based containers for EXIF, UUID boxes for XMP, and mdta handlers for Apple-specific metadata.

## Metadata Storage Architecture

### Three-Tier Metadata System
1. **EXIF Data**: Stored in item-based containers within the HEIC structure
2. **XMP Data**: Stored in UUID boxes with specific identifiers
3. **Apple Metadata**: Stored in mdta (metadata tags) handlers with dynamic keys

### Processing Priority
**Location**: QuickTime.pm:9897-9898
```perl
# Special handling: XMP normally takes priority over EXIF, but not for HEIC
# (this allows EXIF to override XMP for HEIC files)
```

HEIC files have modified priority rules where EXIF takes precedence over XMP, unlike other formats.

## EXIF Data Extraction

### Storage Location
EXIF data in HEIC files is **not** stored in traditional Exif boxes. Instead, it's stored as **items** within the HEIC container structure, identified through the item information system.

### Processing Function
**Location**: `HandleItemInfo()` at QuickTime.pm:9345-9367

```perl
# EXIF processing within item handler
if ($name eq 'EXIF' and length $buff >= 4) {
    if ($buff =~ /^(MM\0\x2a|II\x2a\0)/) {
        $et->Warn('Missing Exif header');
    } elsif ($buff =~ /^Exif\0\0/) {
        $et->Warn('Missing Exif header size');
        $start = 6;
    } else {
        my $n = unpack('N', $buff);
        $start = 4 + $n; # skip "Exif\0\0" header if it exists
    }
    $subTable = GetTagTable('Image::ExifTool::Exif::Main');
    $proc = \&Image::ExifTool::ProcessTIFF;
}
```

### EXIF Processing Flow
1. **Item Identification**: EXIF items identified by content type mapping
2. **Header Validation**: Multiple EXIF header formats supported
3. **TIFF Processing**: Standard TIFF/EXIF processing after header extraction
4. **Error Handling**: Graceful handling of malformed headers

### Content Type Mapping
**Location**: QuickTime.pm:9254
```perl
# Content type to processor mapping
my %processByType = (
    Exif => 'EXIF',
    'application/rdf+xml' => 'XMP',
    # ... other types
);
```

## XMP Data Extraction

### Storage Locations

#### 1. UUID Boxes (Primary)
**Location**: QuickTime.pm:681-690
```perl
uuid => [{
    Name => 'XMP',
    # *** this is where ExifTool writes XMP in MP4 videos (as per XMP spec) ***
    Condition => '$$valPt=~/^\xbe\x7a\xcf\xcb\x97\xa9\x42\xe8\x9c\x71\x99\x94\x91\xe3\xaf\xac/',
    WriteGroup => 'XMP',
    PreservePadding => 1,
    SubDirectory => {
        TagTable => 'Image::ExifTool::XMP::Main',
        Start => 16, # Skip 16-byte UUID
    },
}]
```

**UUID Identifier**: `be7acfcb-97a9-42e8-9c71-999491e3afac`
This specific UUID identifies XMP metadata within the HEIC container.

#### 2. UserData XMP (Alternative)
**Location**: QuickTime.pm:1669-1674
```perl
'XMP_' => {
    Name => 'XMP',
    SubDirectory => { TagTable => 'Image::ExifTool::XMP::Main' },
}
```

XMP can also be stored in traditional QuickTime UserData atoms with the 'XMP_' tag.

### XMP Processing
1. **UUID Recognition**: 16-byte UUID header identifies XMP data
2. **Header Skipping**: First 16 bytes (UUID) skipped for XML processing
3. **Standard XMP Processing**: Uses XMP::Main table for parsing
4. **Padding Preservation**: Maintains XMP padding for writing operations

## Apple-Specific Metadata

### mdta Handler System
**Location**: QuickTime.pm:9659-9686

Apple-specific metadata uses a dynamic key system through the mdta (metadata tags) handler:

```perl
# Read Meta Keys and add tags to ItemList table ('mdta' handler)
$tag =~ s/^com\.(apple\.quicktime\.)?// if $ns eq 'mdta'; # remove apple quicktime domain
```

### Key Apple Tags
**Locations**: Various lines in QuickTime.pm

```perl
# Live Photo metadata
'live-photo.auto' => { Name => 'LivePhotoAuto' },
'live-photo-info' => { Name => 'LivePhotoInfo' },
'live-photo-paired-identifier' => { Name => 'LivePhotoPairedIdentifier' },

# Camera and capture settings
'video-orientation' => { Name => 'VideoOrientation' },
'scene-illuminance' => { Name => 'SceneIlluminance' },
'hdr-gain-map' => { Name => 'HDRGainMap' },

# ProRes RAW EXIF mappings
'com.apple.proapps.exif.{Exif}.*' => 'ProResRAWEXIFMapping',
```

### Keys Table Structure
**Location**: QuickTime.pm:6577-6776

The `keys` atom contains dynamic key definitions that map numeric IDs to metadata keys:
```perl
# Keys atom provides string keys for metadata items
keys => {
    Name => 'Keys',
    SubDirectory => {
        TagTable => 'Image::ExifTool::QuickTime::Keys',
        ProcessProc => \&ProcessKeys,
    },
},
```

### Domain Processing
Apple metadata keys use reverse domain notation:
- `com.apple.quicktime.*` - Standard QuickTime metadata
- `com.apple.proapps.*` - Professional applications metadata
- `mdta` namespace - Metadata tags namespace

## GPS and Location Data

### GPS Storage Formats

#### 1. ISO 6709 Coordinates
**Location**: QuickTime.pm:1616-1623
```perl
"\xa9xyz" => {
    Name => 'GPSCoordinates',
    Groups => { 2 => 'Location' },
    ValueConv => \&ConvertISO6709,
    PrintConv => \&PrintGPSCoordinates,
}
```

**Format**: ISO 6709 standard geographic coordinates
**Processing**: Custom conversion functions for coordinate parsing

#### 2. 3GPP Location Information
**Location**: QuickTime.pm:1733-1793

Complex structured GPS data in `loci` atoms:
```perl
loci => {
    Name => 'LocationInformation',
    SubDirectory => { TagTable => 'Image::ExifTool::QuickTime::LocationInfo' },
}
```

**Structure**: Comprehensive location data including:
- Latitude/longitude coordinates
- Altitude information
- Location accuracy
- Celestial body reference
- Additional location metadata

#### 3. GPS Track Data
**Processing**: Available with `ExtractEmbedded` option
- **Timed GPS Data**: GPS coordinates with timestamps
- **Track Extraction**: Complete GPS tracks from video metadata
- **Sample-based Data**: GPS information per media sample

### Timestamp Handling

#### Content Creation Date
**Location**: QuickTime.pm:1566-1570
```perl
"\xa9day" => {
    Name => 'ContentCreateDate',
    Groups => { 2 => 'Time' },
    %iso8601Date, # ISO 8601 date conversion
}
```

#### Apple-Specific Timestamps
Through mdta handler:
- Creation time with timezone information
- Capture timestamp from camera
- Edit history timestamps

## Metadata Box Hierarchy

### Complete Structure
```
meta (Metadata Container)
├── hdlr (Handler - determines processing type)
├── keys (Key Definitions - for mdta namespace)
├── ilst (Item List - Apple metadata values)
├── iinf (Item Information)
│   └── infe (Item Info Entries - includes EXIF items)
├── iloc (Item Location - EXIF data locations)
└── dinf (Data Information - data references)

uuid (XMP Container)
├── [16-byte UUID: be7a-cfcb-97a9-42e8-9c71-9994-91e3-afac]
└── [XMP XML Data]
```

### Processing Order
1. **Handler Detection**: Check hdlr atom for metadata type
2. **Key Table Loading**: Parse keys atom for mdta mappings
3. **Item Processing**: Extract EXIF from items using iloc
4. **UUID Processing**: Extract XMP from UUID boxes
5. **Apple Metadata**: Process ilst atom with key mappings

## Special Considerations

### HEIC-Specific Behavior

#### 1. Metadata Priority Inversion
Unlike other formats, HEIC gives EXIF priority over XMP to maintain compatibility with camera EXIF data.

#### 2. Item-Based EXIF
EXIF data stored as items rather than atoms requires iloc-based extraction and assembly.

#### 3. Deflate Compression
Some HEIC metadata may be deflate-compressed:
```perl
# Handle compressed metadata
if ($compress) {
    eval { $val = Compress::Zlib::uncompress($val) };
}
```

#### 4. Multiple Document Support
For multi-image HEIC files:
- **Document Numbers**: Each image gets separate document number
- **Metadata Inheritance**: Some metadata shared across images
- **Individual Processing**: Each item processed independently

### Performance Optimizations

#### Fast Scan Mode
**Location**: QuickTime.pm:10046
```perl
# Skip metadata processing in fast scan for large files
last if $fast > 1 and ($tag eq 'mdat' or ($tag eq 'idat' and $$et{FileType} ne 'HEIC'));
```

#### Selective Extraction
- **Group-based filtering**: Extract only requested metadata groups
- **Conditional processing**: Skip expensive operations when not needed
- **Memory efficiency**: Stream processing for large metadata blocks

## Implementation Guide for exif-oxide

### 1. EXIF Item Extraction
```rust
fn extract_exif_from_items(items: &[ItemInfo], locations: &[ItemLocation]) -> Result<ExifData> {
    for item in items {
        if item.content_type == "application/x-exif" {
            let data = extract_item_data(item.item_id, locations)?;
            return parse_exif_with_header_detection(data);
        }
    }
    Err("No EXIF item found")
}
```

### 2. XMP UUID Processing
```rust
const XMP_UUID: [u8; 16] = [0xbe, 0x7a, 0xcf, 0xcb, 0x97, 0xa9, 0x42, 0xe8, 
                           0x9c, 0x71, 0x99, 0x94, 0x91, 0xe3, 0xaf, 0xac];

fn extract_xmp_from_uuid(uuid_boxes: &[UUIDBox]) -> Result<XMPData> {
    for uuid_box in uuid_boxes {
        if uuid_box.uuid == XMP_UUID {
            let xmp_data = &uuid_box.data[16..]; // Skip UUID header
            return parse_xmp(xmp_data);
        }
    }
    Err("No XMP UUID found")
}
```

### 3. Apple Metadata Processing
```rust
fn extract_apple_metadata(keys: &[String], values: &[Vec<u8>]) -> Result<AppleMetadata> {
    let mut metadata = AppleMetadata::new();
    
    for (key, value) in keys.iter().zip(values.iter()) {
        let processed_key = key.strip_prefix("com.apple.quicktime.").unwrap_or(key);
        metadata.insert(processed_key.to_string(), process_apple_value(value)?);
    }
    
    Ok(metadata)
}
```

## Testing and Validation

### Test Cases

#### 1. Standard HEIC with EXIF
- **Structure**: Item-based EXIF data
- **Expected**: Complete EXIF extraction matching ExifTool
- **Validation**: Compare all EXIF tags

#### 2. HEIC with XMP
- **Structure**: UUID box with XMP
- **Expected**: XMP metadata extraction
- **Validation**: XML structure and namespace processing

#### 3. Live Photo HEIC
- **Structure**: Apple mdta metadata with Live Photo keys
- **Expected**: Live Photo detection and metadata
- **Validation**: Check for motion photo indicators

#### 4. Multi-Image HEIC
- **Structure**: Multiple EXIF items for different images
- **Expected**: Separate metadata for each image
- **Validation**: Document numbering and metadata isolation

## References

**Primary Functions**:
- `HandleItemInfo()` - QuickTime.pm:9226+ (main processing)
- `ConvertISO6709()` - GPS coordinate conversion
- `ProcessKeys()` - Apple metadata key processing

**Key Data Structures**:
- XMP UUID definition - QuickTime.pm:681-690
- Apple metadata keys - QuickTime.pm:6577-6776
- Location info structure - QuickTime.pm:1733-1793

**Standards References**:
- ISO 6709:2008 (Geographic coordinates)
- XMP Specification (Adobe)
- Apple QuickTime Metadata (developer.apple.com)

**Related Documentation**:
- `HEIC_HEIF_PROCESSING.md` - Overall processing architecture
- `HEIC_BINARY_EXTRACTION.md` - Item data extraction
- `HEIC_IMAGE_VIDEO_DETECTION.md` - File type detection