# Ricoh.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 1.38
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

The Ricoh.pm module handles maker notes for Ricoh cameras and related brands, including specialized support for 360-degree cameras (Theta series), GXR interchangeable lens units, and unique text-based maker note formats. It also processes custom metadata formats like RMETA and real-time sensor data.

## Module Structure

### Tag Tables (12 total)

**Primary Tables:**

- **Main** (line 46) - Primary Ricoh maker notes with camera settings
- **Type2** (line 447) - Alternative format for HZ15 and Pentax XG-1 models
- **ImageInfo** (line 482) - Binary image information structure

**Specialized Tables:**

- **Subdir** (line 608) - Ricoh subdirectory with manufacturing and firmware data
- **ThetaSubdir** (line 660) - 360-degree camera orientation and GPS data
- **FaceInfo** (line 692) - Face detection positioning information
- **Text** (line 807) - Text-based maker notes for DC/RDC models
- **RMETA** (line 830) - Custom APP5 segment for Pro G3 metadata

**Video/Sensor Tables:**

- **AVI** (line 885) - AVI video file metadata
- **RDTA/RDTB/RDTC/RDTG/RDTL** (lines 915-971) - Real-time sensor data streams
- **Composite** (line 974) - Calculated lens and orientation tags

### Key Data Structures

**Lens Database:**

- **ricohLensIDs** hash (line 31) - GXR lens unit identification
- Maps lens firmware codes to specific lens models
- Includes focal lengths pre-scaled for 35mm equivalent

**Real-Time Data Support:**

- Five specialized RDT tables for different sensor types
- Accelerometer, gyroscope, magnetic field, GPS, and frame timing data
- Timestamp synchronization across multiple data streams

## Processing Architecture

### 1. Main Processing Flow

**Standard EXIF Processing:**

- Uses WriteExif/CheckExif procedures for primary tag tables
- Supports full read/write operations with standard validation
- Maintains EXIF compliance for camera metadata

**Binary Data Processing:**

- 8 tag tables use ProcessBinaryData for complex binary structures
- Handles packed sensor data with precise timing information
- Supports variable-length records and conditional processing

### 2. Special Processing Functions

**ProcessRicohText** (line 1049):

- Parses text-based maker notes from DC/RDC models
- Uses regular expression parsing for key-value pairs
- Handles both binary and ASCII value representations
- Creates dynamic tags for unknown entries

**ProcessRicohRMETA** (line 1095):

- Processes custom APP5 RMETA segments from Pro G3
- Handles multi-segment audio data assembly
- Processes barcode recognition data
- Manages directory structure with multiple data types

**ProcessRicohRDT** (line 1001):

- Processes real-time metadata streams from video files
- Handles multiple data records with frame-by-frame timing
- Supports embedded data extraction with user control
- Manages timestamp synchronization across sensor types

### 3. Data Format Specializations

**Text Format Parser:**

- Regular expression: `([A-Z][a-z]{1,2})([0-9A-F]+);`
- Handles firmware version encoding and color gain values
- Supports both confirmed and unknown tag discovery

**RMETA Multi-Segment Processing:**

- Byte order detection and segment numbering
- Audio data assembly from multiple segments
- Barcode data parsing with length validation

## Key Features

### Multi-Format Support

- **EXIF Maker Notes**: Traditional binary maker note processing
- **Text-Based Notes**: Unique text format for early models
- **Video Metadata**: AVI chunk processing with Unicode handling
- **Real-Time Data**: Frame-synchronized sensor data streams

### 360-Degree Camera Support

- **Orientation Data**: Accelerometer and compass readings
- **GPS Integration**: Location data with timestamp correlation
- **Multi-Sensor Fusion**: Combined sensor data for spatial awareness

### Interchangeable Lens System

- **GXR Unit Support**: Dedicated lens unit identification
- **Focal Length Scaling**: Pre-calculated 35mm equivalent values
- **Mount Detection**: A12 mount adapter recognition

### Face Detection System

- **Position Tracking**: Up to 8 face positions per frame
- **Coordinate System**: Relative positioning to frame dimensions
- **Detection Validation**: Face count verification and conditional processing

### Custom Metadata Formats

- **RMETA Processing**: Proprietary metadata with embedded audio
- **Barcode Recognition**: Automated barcode data extraction
- **Real-Time Streams**: High-frequency sensor data logging

## Special Patterns

### Text-Based Maker Notes

```perl
# Pattern matching for text format
while ($data =~ m/([A-Z][a-z]{1,2})([0-9A-F]+);/sg) {
    my $tag = $1;    # Tag identifier
    my $val = $2;    # Hexadecimal value
}
```

### Real-Time Data Processing

```perl
# Frame-by-frame processing with timestamp correlation
for ($i=0; $i<$count; ++$i) {
    $$et{DOC_NUM} = ++$$et{DOC_COUNT};
    $et->ProcessBinaryData($dirInfo, $tagTablePtr);
    $$dirInfo{DirStart} += $len;
}
```

### Multi-Segment RMETA Assembly

```perl
# Audio data concatenation across segments
if ($$et{VALUE}{SoundFile}) {
    ${$$et{VALUE}{SoundFile}} .= $buff;
}
```

## Usage Notes

### Model-Specific Behavior

- **GXR System**: Lens-specific metadata varies by attached unit
- **Theta Series**: 360-degree specific orientation and GPS data
- **Early Models**: Text-based format requires special processing

### Real-Time Data Extraction

- **ExtractEmbedded Option**: Required for accessing sensor streams
- **Performance Impact**: High-frequency data can be resource intensive
- **Timestamp Correlation**: Multiple streams share timing references

### Video File Support

- **AVI Metadata**: Maker notes embedded in video container
- **MP4 Support**: Through QuickTime framework integration
- **Audio Streams**: Embedded audio data in RMETA segments

### Writing Capabilities

- **Standard Tags**: Full write support for EXIF-compatible tags
- **Binary Structures**: Limited write support for complex binary data
- **Text Format**: Read-only for text-based maker notes

## Debugging Tips

### Text Format Issues

- **Format Validation**: Check for proper text format header (Rev/Rv)
- **Encoding Problems**: Verify hexadecimal value parsing
- **Unknown Tags**: Enable unknown tag extraction for discovery

### Real-Time Data Problems

- **Extraction Settings**: Verify ExtractEmbedded option configuration
- **Data Integrity**: Check for truncated or corrupted streams
- **Timestamp Accuracy**: Validate timestamp conversion calculations

### RMETA Processing Issues

- **Segment Assembly**: Verify proper multi-segment data combination
- **Byte Order**: Check endianness detection for data structures
- **Audio Format**: Validate RIFF header for audio data

### Face Detection Anomalies

- **Coordinate Validation**: Verify face position coordinate ranges
- **Detection Count**: Check FacesDetected vs actual position data
- **Frame Scaling**: Ensure proper coordinate system scaling

### Lens Identification Problems

- **Firmware Parsing**: Verify lens firmware string extraction
- **Database Updates**: Check for new lens models requiring database additions
- **Mount Detection**: Validate A12 mount adapter recognition
