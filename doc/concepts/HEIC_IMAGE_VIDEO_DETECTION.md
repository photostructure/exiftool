# HEIC Image vs Video Detection in ExifTool

**Status**: Research Complete  
**Last Updated**: July 2025  
**Critical Challenge**: HEIC files can contain single images, image sequences, or video content using the same file extension

## Overview

One of the most complex aspects of HEIC processing is correctly distinguishing between different content types within the same container format. ExifTool uses a sophisticated multi-layered detection system to identify whether a HEIC file contains a single still image, an image sequence, or video content (such as Live Photos).

## The Detection Challenge

### Dual Nature Problem
HEIC files use the same file extension (.heic) for fundamentally different content types:
- **Single still images** - Traditional photo files
- **Image sequences** - Burst photos, time-lapse sequences  
- **Motion photos** - Live Photos with embedded video
- **Video files** - Traditional video using HEVC codec

### Why Detection Matters
Incorrect classification leads to:
- **Metadata extraction errors** - Wrong processing pipeline
- **Binary extraction failures** - Looking for image data in video tracks
- **User experience issues** - Expecting image but getting video sequence
- **Performance problems** - Processing video as single image

## ExifTool's Detection Strategy

### Three-Tier Detection System

#### Tier 1: ftyp Brand Analysis (Primary)
**Location**: QuickTime.pm:119-126, 227-231
**Confidence**: High
**Speed**: Fast (first 20 bytes of file)

```perl
# MIME type mapping based on major brand
%mimeLookup = (
    HEIC => 'image/heic',           # Single image - HIGH CONFIDENCE
    HEVC => 'image/heic-sequence',  # Image sequence - HIGH CONFIDENCE  
    HEICS=> 'image/heic-sequence', # Image sequence - HIGH CONFIDENCE
);

# File type descriptions
%ftypLookup = (
    'heic' => 'High Efficiency Image Format HEVC still image (.HEIC)',
    'hevc' => 'High Efficiency Image Format HEVC sequence (.HEICS)', 
    'mif1' => 'High Efficiency Image Format still image (.HEIF)',
    'msf1' => 'High Efficiency Image Format sequence (.HEIFS)',
    'heix' => 'High Efficiency Image Format still image (.HEIF)', # Canon variant
);
```

**Algorithm**:
1. **Read ftyp atom** (bytes 4-7 = 'ftyp')
2. **Extract major brand** (bytes 8-11 after ftyp header)
3. **Classify by brand**:
   - `heic` → Single still image
   - `hevc` → Image sequence/video
   - `mif1` → HEIF still image
   - `msf1` → HEIF sequence

#### Tier 2: Handler Type Verification (Secondary)
**Location**: QuickTime.pm:8319-8323
**Confidence**: Medium-High
**Speed**: Medium (requires atom traversal)

```perl
# Handler types in hdlr atoms
%handlerType = (
    pict => 'Picture',      # HEIC images - CONFIRMS IMAGE CONTENT
    vide => 'Video Track',  # Video content - CONFIRMS VIDEO CONTENT
    soun => 'Audio Track',  # Audio content - INDICATES MOTION MEDIA
);
```

**Algorithm**:
1. **Locate hdlr atoms** in media or metadata containers
2. **Extract handler type** (4-byte identifier)
3. **Classify content**:
   - `pict` handler → Image content (still or sequence)
   - `vide` handler → Video content (motion)
   - `soun` handler → Audio content (indicates video file)

#### Tier 3: Structural Analysis (Tertiary)
**Location**: Various locations in QuickTime.pm
**Confidence**: Medium
**Speed**: Slow (requires deep parsing)

**Duration Analysis**:
```perl
# Movie/track duration indicates video content
# Zero or minimal duration = still image
# Substantial duration = video/motion content
```

**Track Structure**:
- **Single item with image handler** → Still image
- **Multiple items with sequence** → Image sequence  
- **Tracks with video handlers** → Video content
- **Mixed tracks (image + video)** → Live Photo

## Detection Scenarios

### 1. Single Still Image HEIC
**Detection Pattern**:
```
ftyp: major_brand = 'heic'
meta/hdlr: handler_type = 'pict'  
pitm: primary_item = [single_item_id]
Duration: 0 or undefined
```

**Confidence**: Very High
**Processing**: Standard HEIC image extraction

### 2. HEIC Image Sequence
**Detection Pattern**:
```
ftyp: major_brand = 'hevc' 
meta/hdlr: handler_type = 'pict'
iinf: multiple_items with image types
Duration: 0 or undefined (still images)
```

**Confidence**: High  
**Processing**: Multi-image HEIC extraction

### 3. Live Photo HEIC
**Detection Pattern**:
```
ftyp: major_brand = 'heic' OR 'hevc'
meta/hdlr: handler_type = 'pict' 
Additional: Apple Live Photo metadata present
Structure: Image item + video track/item
```

**Special Detection Logic**:
**Location**: QuickTime.pm:6650-6716
```perl
# Live Photo specific metadata detection
'live-photo.auto' => { Name => 'LivePhotoAuto' },
'live-photo-info' => { Name => 'LivePhotoInfo' },
'live-photo-paired-identifier' => { Name => 'LivePhotoPairedIdentifier' },
```

**Confidence**: High
**Processing**: Extract still image + detect motion component

### 4. Samsung Motion Photo HEIC
**Detection Pattern**:
```
ftyp: major_brand = 'heic'
Structure: Primary HEIC image + embedded MP4
Special: mpvd atom present
```

**Special Detection**:
**Location**: QuickTime.pm:950-956
```perl
mpvd => {
    Name => 'MotionPhotoVideo',
    Notes => 'MP4-format video saved in Samsung motion-photo HEIC images.',
}
```

### 5. Video HEIC (HEVC Video)
**Detection Pattern**:
```
ftyp: major_brand = 'hevc'
trak/mdia/hdlr: handler_type = 'vide'
Duration: > 0 (substantial duration)  
Structure: Traditional QuickTime video tracks
```

**Confidence**: Very High
**Processing**: Standard QuickTime video processing

## Detection Implementation

### Primary Detection Function
**Conceptual Algorithm**:
```rust
#[derive(Debug, Clone)]
enum HeicContentType {
    StillImage,
    ImageSequence, 
    LivePhoto,
    MotionPhoto,
    Video,
}

fn detect_heic_content_type(file: &mut File) -> Result<HeicContentType> {
    // Tier 1: Read ftyp atom
    let ftyp = read_ftyp_atom(file)?;
    let major_brand = ftyp.major_brand;
    
    match major_brand {
        b"heic" => {
            // Could be still image, Live Photo, or Motion Photo
            if has_live_photo_metadata(file)? {
                Ok(HeicContentType::LivePhoto)
            } else if has_samsung_motion_metadata(file)? {
                Ok(HeicContentType::MotionPhoto)  
            } else {
                Ok(HeicContentType::StillImage)
            }
        },
        b"hevc" | b"HEVC" => {
            // Could be image sequence or video
            let handlers = find_handler_types(file)?;
            if handlers.contains(&b"vide") {
                Ok(HeicContentType::Video)
            } else {
                Ok(HeicContentType::ImageSequence)
            }
        },
        b"mif1" => Ok(HeicContentType::StillImage),  // HEIF still
        b"msf1" => Ok(HeicContentType::ImageSequence), // HEIF sequence
        _ => Err("Unknown HEIC variant")
    }
}
```

### Handler Type Detection
```rust
fn find_handler_types(file: &mut File) -> Result<Vec<[u8; 4]>> {
    let mut handlers = Vec::new();
    
    // Search for hdlr atoms in various containers
    if let Some(meta_hdlr) = find_atom_in_meta(file, b"hdlr")? {
        handlers.push(meta_hdlr.handler_type);
    }
    
    // Search track-based handlers
    for track in find_tracks(file)? {
        if let Some(trak_hdlr) = find_atom_in_track(&track, b"hdlr")? {
            handlers.push(trak_hdlr.handler_type);
        }
    }
    
    Ok(handlers)
}
```

### Live Photo Detection
```rust
fn has_live_photo_metadata(file: &mut File) -> Result<bool> {
    // Look for Apple Live Photo metadata keys
    let apple_keys = [
        "com.apple.quicktime.live-photo.auto",
        "com.apple.quicktime.live-photo.info", 
        "com.apple.quicktime.live-photo.paired-identifier",
    ];
    
    for key in &apple_keys {
        if find_apple_metadata_key(file, key)?.is_some() {
            return Ok(true);
        }
    }
    
    Ok(false)
}
```

## Edge Cases and Special Handling

### 1. Ambiguous Files
**Scenario**: File has conflicting indicators
**Solution**: Priority system with fallback detection
```rust
// Priority order:
// 1. Explicit Apple/Samsung metadata (highest confidence)
// 2. Handler type analysis
// 3. ftyp brand classification  
// 4. Structural analysis (fallback)
```

### 2. Corrupted Headers
**Scenario**: Missing or invalid ftyp/hdlr atoms
**Solution**: Structural analysis with content sniffing
```rust
// Fallback detection:
// - Count items vs tracks
// - Analyze data containers (mdat vs idat)
// - Look for video-specific atoms (stts, stsc, etc.)
```

### 3. Future Apple Variants
**Scenario**: New Apple camera features with unknown metadata
**Solution**: Conservative classification with logging
```rust
// Default to still image classification
// Log unknown metadata for future analysis
// Provide extensible detection system
```

### 4. Third-Party HEIC Variants
**Scenario**: Other manufacturers using HEIC container
**Solution**: Manufacturer-specific detection extensions
```rust
// Canon: Check for 'heix' brand
// Samsung: Look for motion photo atoms
// Others: Extensible detection framework
```

## Performance Considerations

### Fast Detection Mode
**Goal**: Minimize file I/O for detection
**Strategy**: Read only essential atoms for classification

```rust
fn fast_detect_heic_type(file: &mut File) -> Result<HeicContentType> {
    // Read only first 64 bytes for ftyp
    let ftyp = read_ftyp_fast(file)?;
    
    // For ambiguous cases, read minimal additional data
    match ftyp.major_brand {
        b"heic" => {
            // Quick check for Live Photo indicators
            let has_live_photo = quick_scan_for_live_photo(file)?;
            if has_live_photo {
                Ok(HeicContentType::LivePhoto)
            } else {
                Ok(HeicContentType::StillImage) // Default assumption
            }
        },
        _ => full_detect_heic_type(file) // Fall back to full detection
    }
}
```

### Caching Strategy
- **Cache detection results** to avoid re-analysis
- **Store in metadata context** for processing pipeline
- **Invalidate on file changes** or version updates

## Testing Strategy

### Test File Categories

#### 1. Single Image HEIC
- **iPhone standard photo** (ftyp: heic, hdlr: pict)
- **iPad photo** (various configurations)
- **Third-party camera HEIC** (Canon, etc.)

#### 2. Image Sequences  
- **Burst photo HEIC** (ftyp: hevc, multiple items)
- **Time-lapse sequence** (HEICS extension)
- **Multi-exposure HDR** (related images)

#### 3. Motion Photos
- **iPhone Live Photo** (Apple metadata present)
- **Samsung Motion Photo** (mpvd atom present)
- **Google Motion Photo** (if using HEIC container)

#### 4. Video Files
- **HEVC video in HEIC container** (ftyp: hevc, hdlr: vide)
- **Mixed content** (image + video tracks)

### Validation Methods
```bash
# Compare detection with ExifTool
exiftool -FileType -MIMEType -HandlerType test.heic

# Verify Live Photo detection  
exiftool -LivePhoto* -MotionPhoto* test.heic

# Check video indicators
exiftool -Duration -TrackDuration -HandlerType test.heic
```

## Integration with exif-oxide

### Detection Pipeline Integration
```rust
// Integrate with format detection in src/formats/mod.rs
impl FormatDetector for HeicDetector {
    fn detect(&self, file: &mut File) -> Result<FormatInfo> {
        let content_type = detect_heic_content_type(file)?;
        
        Ok(FormatInfo {
            format: Format::Heic,
            content_type: content_type,
            processor: ProcessorType::QuickTime, // Route through QuickTime
            confidence: DetectionConfidence::High,
        })
    }
}
```

### Processor Routing
```rust
// Route different HEIC types appropriately  
match heic_info.content_type {
    HeicContentType::StillImage => process_heic_image(file),
    HeicContentType::ImageSequence => process_heic_sequence(file),
    HeicContentType::LivePhoto => process_live_photo(file),
    HeicContentType::Video => process_heic_video(file),
}
```

## References

**Primary Detection Logic**:
- ftyp brand mapping - QuickTime.pm:119-126, 227-231
- Handler type processing - QuickTime.pm:8319-8323  
- Live Photo detection - QuickTime.pm:6650-6716
- Motion Photo detection - QuickTime.pm:950-956

**Key Data Structures**:
- %mimeLookup - MIME type by major brand
- %ftypLookup - Description by file type
- %handlerType - Handler type mapping

**Standards References**:
- ISO/IEC 23008-12:2017 (HEIF specification)
- Apple Live Photos Technical Documentation
- Samsung Motion Photo Format Specification

**Related Documentation**:
- `HEIC_HEIF_PROCESSING.md` - Overall processing pipeline
- `HEIC_METADATA_EXTRACTION.md` - Metadata-based detection
- `HEIC_BINARY_EXTRACTION.md` - Content extraction based on type