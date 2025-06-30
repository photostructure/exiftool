# GoPro.pm Module Outline

**ExifTool Version:** 13.26  
**Module Version:** 1.12  
**Document Version:** 1.0  
**Last Updated:** 2025-06-30

## Overview

The GoPro module extracts metadata from GoPro cameras using the GPMF (GoPro Metadata Format), a sophisticated binary format for storing sensor data, GPS information, and camera settings. It supports metadata extraction from MP4 videos (GPMF box and gpmd timed metadata), JPEG files (APP6 "GoPro" segment), and specialized data from GoPro Karma drones. The module handles complex data structures including accelerometer readings, GPS coordinates, and system time synchronization.

## Module Structure

### Tag Tables (7 total)

**Main Processing:**

- `GPMF` - Primary tag table for GoPro Metadata Format with 100+ tags
- `fdsc` - MP4 "fdsc" timed metadata (Hero5/Hero6 specific)

**GPS Data Tables:**

- `GPS5` - Standard GPS data (Hero5, Hero6, Hero9) - 5 fields
- `GPS9` - Enhanced GPS data (Hero13) - 9 fields including date/time
- `GPRI` - Raw GPS data (Karma drone) - 10 fields
- `GLPI` - GPS position data (Karma drone) - 9 fields

**Battery/Status:**

- `KBAT` - Battery status information (Karma drone) - 15 fields

### Key Data Structures

**Format System:**

- `%goProFmt` - Maps GPMF format codes to ExifTool types (15 formats)
- `%goProSize` - Custom size definitions for complex formats
- Support for containers, integers, floats, strings, and complex structures

**Processing Utilities:**

- `%addUnits` - Template for adding units to PrintConv values
- `%noYes` - Simple boolean conversion (N/Y to No/Yes)

**Complex Structures:**

- GPMF uses 4-byte tag IDs with format codes and length/count
- Support for nested containers (DEVC, STRM subdirectories)
- Dynamic structure parsing using TYPE field definitions

## Processing Architecture

### 1. Main Processing Flow

**File Recognition:**

- Processes GPMF box in MP4 videos
- Handles APP6 "GoPro" segments in JPEG files
- Extracts gpmd timed metadata with ExtractEmbedded option
- Supports "GP\x06\0" records in MP4 mdat atoms

**Processing Sequence:**

1. Identify data source (MP4/JPEG/timed metadata)
2. Parse GPMF binary structure (tag/format/length/count)
3. Apply format-specific data extraction
4. Handle nested containers recursively
5. Apply scaling factors and unit conversions
6. Synchronize timestamps for temporal data

### 2. Special Processing Functions

**ProcessGoPro($$$)**

- Primary GPMF format processor
- Handles binary structure parsing with format codes
- Processes nested containers and complex structures
- Implements dynamic structure parsing using TYPE definitions
- Applies scaling factors and handles unknown tags

**ProcessGP6($$)**

- Processes "GP\x06\0" records in MP4 mdat atoms
- Handles multiple DEVC containers in stream
- Manages document numbering for embedded data

**ProcessString($$$)**

- Extracts individual values from space-separated strings
- Used for GPS coordinate arrays and sensor data
- Handles array-based data structures

**ConvertSystemTime($$)**

- Sophisticated time synchronization system
- Uses SYST calibration data for temporal alignment
- Performs binary search and interpolation
- Converts system time to UTC with millisecond precision

**ScaleValues($$)**

- Applies SCAL scaling factors to numeric data
- Handles both single values and arrays
- Enables unit conversion and normalization

**AddUnits($$$)**

- Adds measurement units to values for display
- Uses stored Units information from UNIT/SIUN tags
- Provides human-readable output formatting

## Key Features

### 1. Advanced GPS Processing

**Multi-Format GPS Support:**

- GPS5: Basic 5-field GPS (lat/lon/alt/2D speed/3D speed)
- GPS9: Enhanced 9-field GPS with date/time and DOP
- GPRI: Raw GPS with full precision and status
- GLPI: Position with velocity vectors

**GPS Features:**

- Automatic coordinate conversion (degrees to DMS)
- Speed unit conversion (m/s to km/h)
- GPS time to local time conversion
- Horizontal positioning error calculation
- 2D/3D measurement mode detection

### 2. Sensor Data Integration

**Inertial Measurement:**

- 3-axis accelerometer data (m/s²)
- 3-axis gyroscope data (rad/s)
- 3-axis magnetometer data (µT)
- Camera orientation quaternions
- Attitude and attitude target data

**Environmental Sensors:**

- Camera temperature monitoring
- Battery status (current, voltage, capacity, temperature)
- Barometric pressure (Karma drone)
- Audio level monitoring (dBFS)

### 3. Professional Video Features

**Camera Settings:**

- Protune settings (ISO, sharpness, color, white balance)
- Video resolution and frame rate
- Field of view settings (Wide, Super View, Linear)
- Electronic image stabilization status
- HDR video mode settings

**Production Metadata:**

- Media unique identifiers
- Firmware version tracking
- Device and model identification
- Serial number extraction
- Creation timestamps with time zones

## Special Patterns

### 1. GPMF Binary Structure

GPMF uses a sophisticated binary format:

```
Structure: [Tag][Fmt][Len][Count][Data...]
- Tag: 4-byte identifier
- Fmt: 1-byte format code (0x00 = container, others = data types)
- Len: 1-byte size per element
- Count: 2-byte element count
- Data: Variable length payload
```

**Container Processing:**

- Format 0x00 indicates nested structure
- Recursive processing with subdirectory tables
- Automatic alignment to 4-byte boundaries

### 2. Dynamic Structure Parsing

The module supports complex structure parsing:

- TYPE field defines structure layout using format codes
- Dynamic parsing based on runtime type information
- Support for mixed-format structures (format 0x3f)
- Automatic field extraction and joining

### 3. Temporal Data Synchronization

**System Time Calibration:**

- SYST tags provide time calibration points
- Binary search for temporal alignment
- Linear interpolation between calibration points
- Microsecond precision time conversion

**Embedded Data Processing:**

- ExtractEmbedded option enables timed metadata
- Document numbering for temporal sequences
- GPS and sensor data correlation

## Usage Notes

### Camera Support

**GoPro Models:**

- Hero5, Hero6, Hero7, Hero8, Hero9, Hero10, Hero11, Hero12, Hero13
- GoPro Max (360-degree cameras)
- GoPro Karma (drone platform)
- Various action camera models

**Data Sources:**

- MP4 video files (GPMF box and gpmd streams)
- JPEG photos (APP6 GoPro segment)
- Timed metadata streams (requires -ee option)

### Metadata Types

**Core Camera Data:**

- Camera settings and mode information
- Image quality and processing parameters
- Lens characteristics and field of view
- Serial numbers and firmware versions

**Sensor Streams:**

- High-frequency GPS data (multiple Hz)
- Accelerometer/gyroscope readings
- Orientation and stabilization data
- Audio and environmental sensors

## Debugging Tips

### Common Issues

**Missing Metadata:**

- Enable ExtractEmbedded (-ee) for timed metadata streams
- Check for GPMF box presence in MP4 files
- Verify APP6 segment in JPEG files

**GPS Data Problems:**

- GPS may be disabled or unavailable
- Check GPS acquisition status and DOP values
- Verify coordinate scaling and unit conversion

**Time Synchronization:**

- System time calibration may be missing
- Check for SYST tags in metadata stream
- Temporal data requires calibration for accuracy

### Advanced Debugging

**Format Code Issues:**

- Unknown format codes indicate new camera firmware
- Check upper nibble format detection logic
- New cameras may introduce additional data types

**Structure Parsing:**

- TYPE field defines complex structure layout
- Verify format code mapping for new structures
- Complex structures may require firmware-specific handling

**Scaling Problems:**

- SCAL factors must match data structure
- Check scaling application timing
- Units may require custom conversion logic

**Performance Optimization:**

- Large video files may have extensive timed metadata
- Consider selective tag extraction for performance
- Embedded data can be memory intensive

The GoPro module demonstrates sophisticated handling of modern action camera metadata with emphasis on sensor fusion, temporal synchronization, and professional video production workflows.
