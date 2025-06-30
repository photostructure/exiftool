# Nikon.pm Module Outline

**ExifTool Version:** 13.26  
**Module Version:** 3.89  
**Document Version:** 1.0  
**Last Updated:** 2025-06-29

## Overview

The Nikon module is ExifTool's largest and most sophisticated maker notes implementation, featuring advanced encryption, comprehensive lens databases, and extensive model-specific processing. It handles metadata from Nikon's entire camera lineup including DSLRs and mirrorless Z-series.

## Module Structure

### Tag Tables (135 total)

**Primary Tables:**
- **%Nikon::Main** - Primary dispatcher (1,500+ lines)
- **%nikonLensIDs** - 618 lens entries (largest lens database)
- **%nikonTextEncoding** - Character encoding mappings

**Model-Specific Tables (30+):**
- ShotInfo tables for each camera generation
- Z-series specific: ShotInfoZ6III, Z7II, Z8, Z9
- MenuSettings variants for advanced cameras
- Size-validated camera info blocks

**Specialized Tables:**
- FlashInfo (versions 0100, 0102, 0103, 0106, 0107, 0300)
- PictureControl (3 versions)
- ColorBalance (versions A, B, C, 1-4)
- AFInfo with multiple structure variants

### Encryption System

**Key Components:**
- Serial number and shutter count as encryption keys
- Pre-scan for key extraction (tags 0x001d, 0x00a7)
- Multiple encrypted sections per camera
- Key validation and management

**ProcessNikonEncrypted($$$)**
- Handles decryption/encryption of protected data
- Model-specific decryption algorithms
- Maintains keys for write operations

## Processing Flow

### 1. Key Extraction Phase
```perl
ProcessNikon() 
  ├── Pre-scan for SerialNumber (0x001d)
  ├── Pre-scan for ShutterCount (0x00a7)
  └── Store keys in ExifTool object
```

### 2. Data Processing
- Decrypt encrypted sections using stored keys
- Process standard EXIF-format maker notes
- Handle model-specific subdirectories
- Validate data integrity

### 3. Special Processing Functions

**ProcessNikonCaptureEditVersions($$$)**
- Nikon Capture NX edit history
- Version tracking for edits
- Parameter extraction

**ProcessNikonAVI/MOV($$$)**
- Custom video metadata extraction
- Proprietary Nikon video tags
- Frame rate and codec information

## Key Features

### Advanced Lens System

**LensIDConv() Function:**
- 8-parameter composite lens ID
- Pattern matching for variants
- Third-party lens detection
- Firmware variation handling

**Lens Database:**
- 618 lens entries (55% more than Canon)
- Comprehensive third-party support
- Historical lens compatibility
- Teleconverter detection

### AF System Processing

**AF Point Functions:**
- `PrintAFPoints()` - Bitmask to readable names
- `PrintAFPointsGrid()` - Modern grid systems
- `GetAFPointGrid()` - Coordinate conversion
- Model-specific AF configurations

**AF Features:**
- Phase detection mapping
- Subject tracking data
- 3D tracking information
- Face detection integration

### Picture Control System

**Multiple Versions:**
- PictureControl (original)
- PictureControl2 (enhanced)
- PictureControl3 (latest)
- Custom picture control data

### Flash System Data

**FlashInfo Versions:**
- Progressive feature addition
- External flash communication
- TTL pre-flash data
- Multi-flash coordination

## Unique Patterns

### Encryption Pattern
```perl
# Pre-scan for decryption keys
if ($tagID == 0x001d) {  # SerialNumber
    $serialKey = $value;
} elsif ($tagID == 0x00a7) {  # ShutterCount
    $countKey = $value;
}
```

### Lens ID Pattern
```perl
# 8-byte lens data interpretation
my $id = sprintf("%.2X %.2X %.2X %.2X %.2X %.2X %.2X %.2X",
    unpack("C*", $val));
```

### Model Detection
```perl
Condition => '$$self{Model} =~ /^NIKON Z 9/i',
```

## Special Considerations

### Z-Series Evolution
- Mirrorless-specific metadata
- Electronic viewfinder data
- In-body stabilization info
- Advanced video features

### Encryption Challenges
- Key extraction must succeed for data access
- Some cameras use alternative key locations
- Firmware updates may change encryption

### Write Support
- Limited due to encryption
- Key management for re-encryption
- Validation of encrypted data

## Comparison with Canon

| Feature | Nikon | Canon |
|---------|-------|-------|
| Module Size | 14,191 lines | 10,639 lines |
| Tag Tables | 135 | 107 |
| Lens Database | 618 entries | ~400 entries |
| Encryption | Advanced | None |
| Video Support | Native | Limited |
| AF Complexity | Grid systems | Point-based |

## Usage Notes

- Most complex ExifTool module
- Encryption requires valid keys
- Frequent updates for new models
- Performance impact from decryption

## Debugging Tips

- Use `-v3` to see encryption key extraction
- Check for "Decryption error" warnings
- Missing keys prevent data access
- Validate serial number format for camera model