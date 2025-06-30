# Sony.pm Module Outline

**ExifTool Version:** 13.26  
**Module Version:** 2.20  
**Document Version:** 1.0  
**Last Updated:** 2025-06-29

## Overview

The Sony module handles metadata from Sony's diverse camera lineup, including Alpha DSLRs, mirrorless cameras, and video cameras. It features sophisticated encryption, extensive E-mount lens support, and comprehensive video metadata extraction.

## Module Structure

### Tag Tables (74 total)

**Primary Tables:**
- **%Sony::Main** - Primary dispatcher with 200+ tags
- **%sonyLensTypes2** - E-mount lens database (1,000+ entries)
- **%sonyLensTypes** - A-mount/legacy lens database

**Camera-Specific Tables:**
- CameraInfo, CameraInfo2, CameraInfo3 - Generation-based
- CameraInfoUnknown - Fallback for new models
- Model-specific variants for A100, A200, etc.

**Feature Tables:**
- Panorama - Sweep panorama settings
- FaceInfo1-5 - Face detection data variants
- MoreInfo, MoreSettings - Extended camera data
- rtmd - Real-time metadata from videos

**Format Tables:**
- SR2, SR2Private - Sony Raw format
- ARW - Advanced Raw format  
- SRF - Sony Raw Format
- PMP - Picture Motion Picture

### Encryption System

**Cipher Functions:**
- `Decipher()` - Simple substitution cipher (cube formula)
- `Decrypt()` - Complex LFSR-based encryption
- Double-encryption bug handling (v9.04-9.10)

**Encrypted Sections:**
- 0x94xx tag directories
- Camera-specific info blocks
- Sensitive calibration data

## Processing Flow

### 1. Format Detection
```perl
SetARW() -> Determine ARW vs SR2
ProcessSony() -> Main entry point
  ├── Model identification
  ├── Encryption detection
  └── Dispatch to subtables
```

### 2. Encryption Handling
- Check for enciphered tags (0x94xx)
- Apply appropriate decryption
- Handle double-encryption cases
- Validate decrypted data

### 3. Special Processing Functions

**ProcessEnciphered($$$)**
- Decrypts protected 0x94xx directories
- Handles write operations with re-encryption
- Model-specific encryption keys

**ProcessMoreInfo($$$)**
- Sony's custom info structure
- Variable-length entries
- Validation and parsing

**Process_rtmd($$$)**
- Video real-time metadata
- Frame-by-frame camera settings
- Focus/zoom tracking data

## Key Features

### E-Mount Lens System

**Comprehensive Database:**
- 1,000+ E-mount lenses
- Extensive third-party support
- Adapter detection (LA-EA, Metabones)
- Teleconverter handling

**Lens Identification:**
- LensType2 for E-mount
- LensType for A-mount
- Specification parsing
- Mount adapter compensation

### Video Metadata (rtmd)

**Real-time Data:**
- Focus ring position
- Zoom ring position
- Camera orientation
- Exposure changes
- GPS tracking (if available)

**Video Groups:**
- Organized by data type
- Frame-accurate timing
- Synchronized with video stream

### Raw Format Support

**Multiple Formats:**
- ARW (primary raw format)
- SR2 (older format)
- SRF (Sony Raw Format)
- Format-specific metadata

**ARW Features:**
- IFD-based structure
- Embedded preview images
- Color profile data
- Model-specific quirks

## Unique Patterns

### Conditional Tag Processing
```perl
Condition => '$$self{Model} =~ /^ILCE-/',  # E-mount cameras
Condition => '$$valPt =~ /^\x01\x00/',    # Format detection
```

### Encryption Pattern
```perl
# Enciphered directory detection
if ($tagID >= 0x9400 and $tagID < 0x9500) {
    ProcessEnciphered($et, $dirInfo, $tagTablePtr);
}
```

### Lens Detection
```perl
# E-mount lens with adapter check
my $lensType2 = $$et{LensType2} || 0;
if ($lensType2 > 0x8000) {  # Adapted lens
    # Special handling
}
```

## Special Features

### Sweep Panorama
- Direction detection
- Image size calculation
- Stitching parameters
- Preview extraction

### Computational Photography
- HDR mode data
- Dynamic Range Optimizer
- Multi-frame noise reduction
- Handheld night shot

### Face Detection
- Multiple face info versions
- Face coordinates
- Recognition confidence
- Smile detection

## Comparison with Canon/Nikon

| Feature | Sony | Canon | Nikon |
|---------|------|-------|-------|
| Encryption | Yes (Simple) | No | Yes (Complex) |
| Video Metadata | Extensive | Limited | Moderate |
| Lens Database | 1,000+ | ~400 | 618 |
| Raw Formats | 3 | 2 | 1 |
| Face Detection | Advanced | Basic | Moderate |

## Usage Notes

- Encryption simpler than Nikon
- Extensive video support
- E-mount ecosystem focus
- Frequent firmware updates
- Good third-party lens support

## Debugging Tips

- Check encryption with `-v3`
- Lens ID conflicts common with adapters
- Video metadata requires MP4 format
- Face info version varies by model
- Double-encryption affects old files