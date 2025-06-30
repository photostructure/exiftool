# Canon.pm Module Outline

**ExifTool Version:** 13.26  
**Module Version:** 4.81  
**Document Version:** 1.0  
**Last Updated:** 2025-06-29

## Overview

The Canon module is one of ExifTool's largest and most complex modules, handling proprietary metadata from Canon cameras. It includes extensive model-specific processing, comprehensive lens databases, and sophisticated serial number decoding.

## Module Structure

### Tag Tables (107 total)

**Primary Tables:**

- **%Canon::Main** - Primary dispatcher, contains 200+ tags
- **%Canon::CameraSettings** - Core camera configuration
- **%Canon::ShotInfo** - Shot-specific information
- **%Canon::ProcessingInfo** - Image processing parameters

**Camera-Specific Tables (32):**

- CameraInfo tables for each model line (1D, 5D, 7D, R series, PowerShot, etc.)
- Model-specific parsing based on firmware version
- Size validation for each camera info block

**Color Data Tables (12):**

- ColorData1 through ColorData12 for different camera generations
- ColorCoefs, ColorCalib for calibration data
- Model-specific color processing parameters

**Specialized Tables:**

- AFInfo/AFInfo2 - Autofocus system configuration
- LensInfo - Lens identification and characteristics
- MovieInfo - Video recording parameters
- FaceDetect1-3 - Face detection data

### Key Data Structures

**Lens Database:**

- `%canonLensTypes` - 400+ Canon and third-party lenses
- Lens ID calculation with teleconverter support
- Focal length and aperture range mapping

**Serial Number Patterns:**

- Model-specific decoding algorithms
- Format validation for different camera series
- Manufacturing date extraction

## Processing Flow

### 1. Main Entry Processing

```perl
ProcessCanon() -> %Canon::Main
  ├── Serial number extraction
  ├── Model identification
  └── Dispatch to subtables
```

### 2. Model-Specific Routing

- Firmware version detection ("FirmwareVersionLookAhead")
- Camera info table selection based on model/version
- Size validation before processing

### 3. Special Processing Functions

**ProcessSerialData($$$)**

- Decodes encoded serial numbers
- Validates format based on camera model
- Extracts manufacturing information

**ProcessFilters($$$)**

- Parses creative filter parameters
- Handles variable-length filter data
- Decodes filter-specific settings

**ProcessCTMD($$$)**

- Canon Timed MetaData processing
- GPS and acceleration data extraction
- Timestamp synchronization

## Key Features

### Serial Number Handling

- Complex model-specific decoding
- Format validation (10-12 digits)
- Special handling for professional bodies (1D series)

### Lens Identification System

- Lens ID calculation from multiple parameters
- Teleconverter detection and compensation
- Third-party lens support (Sigma, Tamron, Tokina, Zeiss)
- Focal length range validation

### AF System Data

- AF point configuration and selection
- Microadjustment values
- Focus distance information
- Subject tracking data

### Color Processing

- White balance calibration
- Color temperature fine-tuning
- Picture style parameters
- Model-specific color matrices

## Common Patterns

### Binary Data Processing

```perl
PROCESS_PROC => \&Image::ExifTool::ProcessBinaryData,
FORMAT => 'int8u',
FIRST_ENTRY => 0,
DATAMEMBER => [ 'tag1', 'tag2' ],  # for dependencies
```

### Conditional Tags

```perl
Condition => '$$self{Model} =~ /EOS-1D/',
Condition => '$$self{FirmwareVersion} ge "2.0.0"',
```

### Validation Pattern

```perl
my $validate = { 0x00 => 254, 0x02 => 5 };  # offset => expected value
return 0 unless Image::ExifTool::Validate($self, $data, $validate);
```

### Conversion Functions

- `CanonEv()` - Exposure value conversion
- `PrintFocalRange()` - Lens focal length formatting
- `LensWithTC()` - Teleconverter adjustment

## Special Considerations

### Firmware Dependencies

- Tag locations vary by firmware version
- Version detection required before processing
- Fallback handling for unknown versions

### Model Evolution

- Newer models may reuse older camera info structures
- Progressive addition of features in subtables
- Backward compatibility maintained

### Write Support

- Limited to specific tags
- TIFF footer handling in WriteCanon()
- Validation of written values

## Usage Notes

- Most complex maker notes implementation
- Frequent updates for new camera models
- Extensive validation prevents misinterpretation
- Performance considerations due to size

## Debugging Tips

- Use `-v3` to see firmware version detection
- Check validation warnings for corrupt data
- Lens ID conflicts may need manual resolution
- Camera info size mismatches indicate new firmware
