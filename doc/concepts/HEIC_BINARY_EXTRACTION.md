# HEIC Binary Data Extraction in ExifTool

**Status**: Research Complete  
**Last Updated**: July 2025  
**Critical for**: PhotoStructure binary image extraction requirements

## Overview

This document details ExifTool's approach to extracting binary image data from HEIC files. Unlike traditional image formats where image data is stored sequentially, HEIC uses an item-based system with complex location mapping through the iloc (Item Location) box.

## Key Challenges

### 1. Item-Based Storage
Unlike JPEG (single image stream) or traditional QuickTime (track-based media), HEIC stores images as discrete "items" referenced by unique IDs with potentially fragmented data across multiple locations.

### 2. Multiple Storage Methods
HEIC supports three construction methods for item data:
- **Method 0**: Direct file offset (traditional approach)
- **Method 1**: Data stored in `idat` container (HEIC-specific)
- **Method 2**: Item references (derived/computed data)

### 3. Fragmented Data
Image data can be split across multiple "extents" (data fragments), requiring assembly from multiple file locations.

## Core Data Structures

### Item Location (iloc) Box Structure
**Parsing Function**: `ParseItemLocation()` at QuickTime.pm:9014

```perl
# iloc box provides mapping from item ID to data location
{
    ItemID => 1001,              # Unique identifier for this item
    ConstructionMethod => 0,     # How data is stored (0/1/2)
    DataReferenceIndex => 0,     # Usually 0 for local data
    BaseOffset => 12345,         # Base address for calculations
    Extents => [                 # Array of data fragments
        [extent_index, offset, length, size_bytes, position],
        # Multiple extents possible for fragmented data
    ]
}
```

### Item Information (iinf/infe) Structure
```perl
# Maps item IDs to content types and properties
{
    ItemID => 1001,
    ItemType => 'hvc1',          # HEVC codec type
    ItemName => 'Image',         # Human-readable name
    ContentType => 'image/hevc', # MIME type
    ItemProtectionIndex => 0,    # Usually 0 (no protection)
}
```

## Binary Extraction Process

### 1. Primary Image Identification
**Location**: QuickTime.pm:2818-2827
```perl
pitm => [{
    Name => 'PrimaryItemReference',
    RawConv => '$$self{PrimaryItem} = unpack("x4N",$val)',
}]
```

The `pitm` (Primary Item) box identifies which item contains the main displayable image.

### 2. Data Location Resolution
**Function**: `HandleItemInfo()` at QuickTime.pm:9226+

**Algorithm**:
1. **Iterate through all items** in the file
2. **Check if item is primary** using `$$et{PrimaryItem}`
3. **Resolve data location** using iloc information
4. **Calculate absolute offsets** considering base offsets and construction methods
5. **Handle multiple extents** for fragmented data

### 3. Binary Data Assembly
**Core Logic**: QuickTime.pm:9319-9323
```perl
# Assemble data from multiple extents
foreach $extent (@{$$item{Extents}}) {
    $val .= $buff if defined $buff;
    $raf->Seek($$extent[1] + $base, 0) or last;
    $raf->Read($buff, $$extent[2]) or last;
}
```

**Steps**:
1. **Calculate base offset** based on construction method
2. **Iterate through extents** in the item
3. **Seek to each fragment location**
4. **Read fragment data** into buffer
5. **Concatenate fragments** to form complete image

## Construction Method Handling

### Method 0: File Offset (Traditional)
- **Base Calculation**: Use iloc BaseOffset directly
- **Data Location**: Absolute file positions
- **Usage**: Standard approach for most items

### Method 1: idat Container (HEIC-Specific)
**Critical Code**: QuickTime.pm:9266
```perl
# Note: In HEIC's, these seem to indicate data in 'idat' instead of 'mdat'
my $constMeth = $$item{ConstructionMethod} || 0;
```

- **Base Calculation**: BaseOffset + MediaDataInfo[0] (idat location)
- **Data Location**: Relative to idat container start
- **Usage**: Common in HEIC files for image data

### Method 2: Item Reference (Derived)
- **Base Calculation**: Complex reference resolution
- **Data Location**: Computed from other items
- **Usage**: Thumbnail images, derived content

## Multi-Image Handling

### Burst Photos and Sequences
**Processing**: QuickTime.pm:9399-9405

**Algorithm**:
1. **Process all items** in ItemInfo array
2. **Create sub-documents** for non-primary items
3. **Track relationships** via iref (Item Reference) box
4. **Maintain document numbers** (`$$et{DOC_NUM}`)

```perl
# Multi-document processing for burst/sequence images
if (@{$$et{ItemInfo}} > 1) {
    # Process additional items as separate documents
    $$et{DOC_NUM} = ++$docNum;
}
```

### Live Photos (Motion Photos)
**Special Handling**: Lines 6650-6716, 950-956

Live Photos contain both still image and video components:
- **Image Item**: Primary still frame (typically item 1)
- **Video Item**: 3-second video clip (separate item)
- **Metadata Links**: Apple-specific metadata connects them

```perl
# Live Photo detection and processing
'live-photo.auto' => { Name => 'LivePhotoAuto' },
'live-photo-info' => { Name => 'LivePhotoInfo' },
```

## Data Container Handling

### mdat (Media Data) Box
- **Traditional QuickTime**: Primary media storage
- **HEIC Context**: May contain image data for Method 0 items
- **Processing**: Standard QuickTime mdat handling

### idat (Item Data) Box - HEIC Specific
- **Purpose**: HEIC-specific item data storage
- **Usage**: Preferred container for Method 1 construction
- **Processing**: Requires HEIC-aware offset calculations

**Key Difference**: QuickTime.pm:10046
```perl
# Fast scan handling differentiates idat processing
last if $fast > 1 and ($tag eq 'mdat' or ($tag eq 'idat' and $$et{FileType} ne 'HEIC'));
```

## Error Handling and Edge Cases

### Fragmented Data
- **Multiple Extents**: Items can span multiple file locations
- **Gap Handling**: Missing or corrupted extents gracefully handled
- **Size Validation**: Extent sizes verified against item information

### Large File Support
- **64-bit Sizes**: Extended atom sizes for large containers
- **Memory Management**: Streaming reads to avoid loading entire file
- **Progress Tracking**: Extent-by-extent processing for large images

### Corrupted Files
- **Missing iloc**: Graceful fallback when location info unavailable
- **Invalid Offsets**: Bounds checking for file access
- **Partial Data**: Continue processing with available fragments

## Performance Considerations

### Fast Scan Optimization
**Location**: QuickTime.pm:10046
```perl
# Skip large data processing in fast mode
last if $fast > 1 and ($tag eq 'mdat' or ($tag eq 'idat' and $$et{FileType} ne 'HEIC'));
```

For metadata-only extraction, ExifTool can skip processing large image data blocks.

### Memory Efficiency
- **Streaming Reads**: Process data in chunks rather than loading entire items
- **Selective Extraction**: Only extract items when specifically requested
- **Buffer Management**: Reuse buffers for multiple extent processing

## Implementation Guide for exif-oxide

### 1. iloc Box Parser
Implement `ParseItemLocation()` equivalent:
```rust
struct ItemLocation {
    item_id: u32,
    construction_method: u8,
    data_reference_index: u16,
    base_offset: u64,
    extents: Vec<ItemExtent>,
}

struct ItemExtent {
    extent_index: u64,
    extent_offset: u64,
    extent_length: u64,
}
```

### 2. Binary Extraction Engine
```rust
fn extract_item_data(item_id: u32, iloc: &ItemLocation, file: &mut File) -> Result<Vec<u8>> {
    let mut data = Vec::new();
    let base_offset = calculate_base_offset(iloc.construction_method, iloc.base_offset);
    
    for extent in &iloc.extents {
        file.seek(SeekFrom::Start(base_offset + extent.extent_offset))?;
        let mut buffer = vec![0u8; extent.extent_length as usize];
        file.read_exact(&mut buffer)?;
        data.extend_from_slice(&buffer);
    }
    
    Ok(data)
}
```

### 3. Primary Image Selection
```rust
fn get_primary_image_data(pitm_item_id: u32, items: &[ItemInfo], locations: &[ItemLocation]) -> Result<Vec<u8>> {
    let primary_item = items.iter()
        .find(|item| item.item_id == pitm_item_id)
        .ok_or("Primary item not found")?;
    
    let item_location = locations.iter()
        .find(|loc| loc.item_id == pitm_item_id)
        .ok_or("Primary item location not found")?;
    
    extract_item_data(pitm_item_id, item_location, file)
}
```

## Testing Scenarios

### Single Image HEIC
- **Structure**: One primary item with image data
- **Expected**: Single image extraction
- **Validation**: Compare with ExifTool -b -PreviewImage

### Live Photo HEIC
- **Structure**: Primary image item + video item
- **Expected**: Extract still frame, identify video component
- **Validation**: Check for motion photo metadata

### Burst Sequence HEIC
- **Structure**: Multiple image items with relationships
- **Expected**: Extract all images, maintain sequence order
- **Validation**: Document numbers should increment

### ProRAW HEIC
- **Structure**: DNG data wrapped in HEIC container
- **Expected**: Extract DNG data, not decoded image
- **Validation**: Extracted data should be valid DNG

## References

**Primary Functions**:
- `ParseItemLocation()` - QuickTime.pm:9014
- `HandleItemInfo()` - QuickTime.pm:9226+
- `GetVarInt()` - QuickTime.pm:8925 (variable-length integer parsing)

**Key Data Structures**:
- iloc table definition - QuickTime.pm:2770
- iref table definition - QuickTime.pm:2836
- Image type detection - QuickTime.pm:536

**Standards References**:
- ISO/IEC 23008-12:2017 (HEIF specification) - Section 6.4 (Item Location)
- ISO/IEC 14496-12 (ISOBMFF) - Section 8.11 (Item Location Box)

**Related Documentation**:
- `HEIC_HEIF_PROCESSING.md` - Overall processing architecture
- `HEIC_METADATA_EXTRACTION.md` - Metadata from items
- `HEIC_IMAGE_VIDEO_DETECTION.md` - Content type detection