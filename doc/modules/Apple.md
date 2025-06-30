# Apple.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 1.15
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

The Apple.pm module handles EXIF maker notes tags from Apple devices, primarily iPhone and iPad cameras. It extracts comprehensive metadata including camera sensor information, autofocus performance, HDR processing, Live Photos data, burst mode details, and advanced computational photography features like semantic styles and depth sensing.

## Module Structure

### Tag Tables (3 total)

1. **Main** - Primary Apple maker notes tag table with 42+ tag definitions covering camera sensors, image processing, and device metadata
2. **RunTime** - PLIST-format CMTime structure for device runtime tracking since last boot
3. **Composite** - Derived tags like RunTimeSincePowerUp calculated from multiple source values

### Key Data Structures

- **Main tag table**: Uses standard EXIF structure with hex-indexed tags (0x0001-0x005a)
- **PLIST integration**: Binary PLIST parsing for complex structured data
- **CMTime structure**: Apple's Core Media time representation for precise timing
- **Composite tags**: Calculated values combining multiple raw measurements

## Processing Architecture

### 1. Main Processing Flow

- Inherits standard EXIF write/check procedures from Image::ExifTool::Exif
- Uses ConvertPLIST() function to process binary PLIST data embedded in tags
- Supports both reading and writing of most maker notes tags
- Integrates with Apple's structured data formats (PLIST, CMTime)

### 2. Special Processing Functions

- **ConvertPLIST()**: Converts binary PLIST format to usable tag values, handling both hash structures and serialized formats
- **ProcessBinaryPLIST**: Processes PLIST-formatted subdirectories (RunTime table)
- **Standard EXIF procedures**: Inherits WriteExif and CheckExif from parent module

## Key Features

### Camera Technology Support
- **Multi-camera systems**: Front, back wide-angle, back normal camera identification
- **Optical Image Stabilization (OIS)**: OIS mode tracking and performance metrics
- **Autofocus performance**: AF stability, confidence, measured depth, and performance metrics
- **Time-of-flight sensors**: Depth measurement and confidence data from ToF-assisted autofocus

### Computational Photography
- **HDR processing**: HDR image type detection, headroom, and gain mapping
- **Live Photos**: Content identifier linking, video index timing, and media group UUID
- **Burst mode**: UUID tracking for image sequences and burst identification
- **Semantic styles**: Apple's photographic style system with tone/warmth bias control
- **ProRAW support**: Raw format capture type identification

### Sensor and Hardware Data
- **Acceleration vector**: 3-axis accelerometer data in g-force units with detailed coordinate system
- **Device runtime**: Precise timing since last boot excluding standby time
- **Focus distance**: Range measurements and focus position tracking
- **Signal processing**: Noise amplitude, SNR measurements, and image processing flags

### Advanced Metadata
- **Content identification**: Unique photo and capture request identifiers
- **Scene analysis**: Person/pet detection flags and scene classification
- **Color processing**: Color correction matrices and temperature measurement
- **Quality metrics**: Processing hints and quality level indicators

## Special Patterns

### PLIST Data Handling
- Extensive use of binary PLIST format for complex structured data
- Automatic deserialization to hash structures or serialized strings
- Integration with XMP structure serialization for compatibility

### Apple Ecosystem Integration
- CMTime structure for precise media timing synchronization
- UUID-based content linking across Live Photos and burst sequences
- Integration with Photos app feature detection and processing

### Coordinate System Documentation
- Detailed acceleration vector coordinate system with physics-accurate descriptions
- Left-handed coordinate system for acceleration measurements
- Comprehensive notes on Apple's coordinate system conventions

## Usage Notes

### Camera Identification
- CameraType tag distinguishes between front camera and back camera variants
- Back cameras include wide-angle (0.5x) and normal focal length options
- Values: 0=Back Wide Angle, 1=Back Normal, 6=Front

### Live Photos Integration
- ContentIdentifier (formerly MediaGroupUUID) links still images to video components
- LivePhotoVideoIndex provides timing offset (divide by RunTimeScale for seconds)
- Only valid when ContentIdentifier exists in metadata

### Computational Photography Features
- ImageCaptureType identifies ProRAW, Portrait mode, and standard photo modes
- SemanticStyle tags contain Apple's photographic style processing parameters
- HDR-related tags track processing status and technical parameters

### Data Precision
- RunTime values require division by RunTimeScale (typically 1,000,000,000 for nanoseconds)
- Focus distances provided in meters with range calculations
- Acceleration vectors in standard g-force units with coordinate system documentation

## Debugging Tips

### PLIST Data Issues
- Use -v option to see PLIST parsing details and structure
- Check for truncated or malformed PLIST data in maker notes
- Verify byte order handling during PLIST conversion

### Missing or Incorrect Values
- Some tags only exist under specific conditions (e.g., Live Photos, flash usage)
- Unknown tags (0x004e, 0x004f, etc.) may contain undocumented features
- Tag availability varies by iOS version and device model

### Coordinate System Confusion
- Apple's documentation incorrectly describes acceleration coordinate system
- ExifTool provides correct physics-based coordinate system documentation
- Verify acceleration vector orientation against actual device positioning

### Version Compatibility
- Tag availability expanded significantly with newer iOS versions
- Some tags deprecated or renamed between versions (ContentIdentifier vs MediaGroupUUID)
- Check device iOS version for feature availability context
