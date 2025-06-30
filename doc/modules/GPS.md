# GPS.pm Module Outline

**ExifTool Version:** 13.26  
**Module Version:** 1.58  
**Document Version:** 1.0  
**Last Updated:** 2025-06-30

## Overview

The GPS module handles EXIF GPS metadata for location information in images and videos. It provides comprehensive coordinate conversion capabilities, robust timestamp processing, and composite tag generation for complete GPS data integration. The module supports reading and writing GPS coordinates in multiple formats, handles timezone conversions for GPS timestamps, and implements MWG 2.0 compliance for camera and subject location tagging.

## Module Structure

### Tag Tables (2 total)

**Main Processing:**

- `Main` - Standard EXIF GPS IFD tags (0x0000-0x001f) with coordinate conversion
- `Composite` - Derived GPS tags combining multiple fields for complete location data

### Key Data Structures

**Coordinate Conversion:**

- `%coordConv` - Template for coordinate ValueConv/PrintConv/PrintConvInv
- `%printConvLatRef` - Latitude reference conversion (N/S with cardinal direction support)
- `%printConvLonRef` - Longitude reference conversion (E/W with cardinal direction support)

**Core GPS Tags:**

- Location data: GPSLatitude, GPSLongitude, GPSAltitude (camera position per MWG 2.0)
- Subject location: GPSDestLatitude, GPSDestLongitude, GPSDestBearing (subject position per MWG 2.0)
- Navigation: GPSTrack, GPSSpeed, GPSImgDirection, GPSDestDistance
- Technical: GPSDOP, GPSMeasureMode, GPSStatus, GPSSatellites, GPSDifferential
- Temporal: GPSTimeStamp, GPSDateStamp with timezone handling
- Reference: GPSMapDatum, GPSProcessingMethod, GPSAreaInformation

**Composite Integration:**

- GPSDateTime - Combined date/time with UTC timezone
- GPSLatitude/GPSLongitude - Signed decimal degrees from coordinate pairs
- GPSAltitude - Combined altitude with reference (above/below sea level)
- GPSDestLatitude/GPSDestLongitude - Subject location coordinates

## Processing Architecture

### 1. Main Processing Flow

**Standard EXIF Processing:**

- Uses standard ExifTool EXIF write/check procedures
- GPS IFD group classification with Location secondary group
- Writable tags with format validation and conversion
- MWG 2.0 compliance for location tag usage

**Coordinate System Support:**

- Degrees/Minutes/Seconds (DMS) format
- Decimal degrees format
- Signed coordinate values
- Cardinal direction references (N/S/E/W)
- Custom coordinate format strings

### 2. Special Processing Functions

**ConvertTimeStamp($)**

- Converts GPS timestamp from rational array to HH:MM:SS.sss format
- Handles fractional seconds with precision
- Normalizes hour/minute/second overflow
- Supports sub-second precision to microseconds

**PrintTimeStamp($)**

- Formats GPS timestamp for display with microsecond rounding
- Handles decimal seconds formatting
- Ensures proper zero-padding for consistency
- Optimizes display by removing unnecessary precision

**ToDMS($$;$$)**

- Comprehensive coordinate conversion to DMS format
- Supports multiple output formats via CoordFormat option
- Handles signed coordinates with cardinal directions
- XMP format support with configurable decimal places
- Custom format string processing with %d, %f specifiers
- Round-off error handling for coordinate boundaries

**ToDegrees($;$$)**

- Robust coordinate parsing from various input formats
- Extracts numeric values from formatted strings
- Handles cardinal direction detection (N/S/E/W)
- Supports combined coordinate string parsing
- GPSCoordinates field separation (lat/lon extraction)
- Scientific notation and decimal format support

## Key Features

### 1. Comprehensive Coordinate Support

**Multiple Input Formats:**

- DMS format: 40° 26' 46.32" N
- Decimal degrees: 40.4462 N
- Signed decimal: +40.4462 (positive = North/East)
- Combined coordinates: "40.4462 N, 73.9865 W"
- Scientific notation: 4.04462E+01

**Flexible Output Formatting:**

- CoordFormat option for custom display
- Default: "40 deg 26' 46.32" N"
- XMP format: "40,26.7720 N" (decimal minutes)
- Signed format: "+40.4462" or "-73.9865"
- Custom format strings with %d/%f specifiers

### 2. Advanced Timestamp Processing

**GPS Time Handling:**

- UTC time conversion and validation
- Timezone adjustment for input timestamps
- Date/time combination for complete temporal data
- Fractional second precision (microseconds)
- "now" keyword support for current time

**Format Flexibility:**

- HHMMSS format parsing
- YYYYmmddHHMMSS format parsing
- Timezone notation: +/-HH:MM, Z (UTC)
- DST (Daylight Saving Time) awareness
- Multiple separator support (: . space)

### 3. MWG 2.0 Compliance

**Camera Location (tags 0x0001-0x0006):**

- GPSLatitude/GPSLatitudeRef - Camera position
- GPSLongitude/GPSLongitudeRef - Camera position
- GPSAltitude/GPSAltitudeRef - Camera elevation
- Standardized usage for camera location

**Subject Location (tags 0x0013-0x001A):**

- GPSDestLatitude/GPSDestLatitudeRef - Subject position
- GPSDestLongitude/GPSDestLongitudeRef - Subject position
- GPSDestBearing/GPSDestBearingRef - Direction to subject
- GPSDestDistance/GPSDestDistanceRef - Distance to subject

## Special Patterns

### 1. Coordinate Conversion Architecture

**Bidirectional Processing:**

```perl
ValueConv    => 'Image::ExifTool::GPS::ToDegrees($val)'     # Read
ValueConvInv => 'Image::ExifTool::GPS::ToDMS($self, $val)'  # Write
PrintConv    => 'Image::ExifTool::GPS::ToDMS($self, $val, 1)' # Display
```

**Template System:**

- `%coordConv` provides consistent conversion across all coordinate tags
- Shared between GPSLatitude, GPSLongitude, GPSDestLatitude, GPSDestLongitude
- PrintConvInv uses ToDegrees for reverse conversion with coordinate type

### 2. Composite Tag Integration

**Multi-Source Processing:**

- GPSAltitude composite supports both GPS and XMP sources
- Desire mechanism for optional tag dependencies
- RawConv validation for required reference tags
- XMP integration with GPS data sources

**Automatic Reference Handling:**

- GPSLatitude composite automatically sets GPSLatitudeRef
- WriteAlso mechanism for coordinated tag writing
- Sign detection for automatic N/S/E/W assignment
- Avoid flag prevents conflicts with direct tag access

### 3. Reference Direction Processing

**Flexible Direction Input:**

- String parsing: "North", "South", "East", "West"
- Abbreviated: "N", "S", "E", "W"
- Signed numeric: +/- values for automatic direction
- Case-insensitive matching with word boundary detection

**Cardinal Direction Logic:**

- North/South from numeric sign or explicit direction
- East/West from numeric sign or explicit direction
- Regex pattern matching for embedded directions
- Fallback to sign-based direction assignment

## Usage Notes

### Coordinate Systems

**Standard GPS Coordinates:**

- WGS84 datum (default for GPS devices)
- Decimal degrees with 6-8 decimal places typical
- Cardinal directions for hemisphere specification
- Altitude relative to WGS84 ellipsoid or sea level

**Supported Reference Systems:**

- GPSMapDatum tag for coordinate system specification
- Processing method indication (GPS, CELLID, WLAN, MANUAL)
- Differential correction status
- Horizontal positioning error measurement

### Writing GPS Data

**Coordinate Writing:**

- Multiple input format acceptance
- Automatic coordinate validation
- Reference direction calculation
- Format conversion and normalization

**Timestamp Writing:**

- Date/time parsing from various formats
- Timezone conversion to UTC
- Separate date and time tag handling
- Current time support with "now" keyword

## Debugging Tips

### Common Issues

**Coordinate Conversion Problems:**

- Verify input format matches expected patterns
- Check for invalid values (inf, undef)
- Ensure proper cardinal direction specification
- Validate numeric ranges (latitude ±90°, longitude ±180°)

**Timestamp Issues:**

- GPS time is always UTC - verify timezone handling
- Check for proper date/time separation
- Validate fractional seconds processing
- Ensure proper format string matching

**Reference Direction Problems:**

- Case sensitivity in direction parsing
- Word boundary detection for embedded directions
- Sign precedence over explicit directions
- Coordinate/reference tag synchronization

### Advanced Debugging

**Composite Tag Issues:**

- Check Require/Desire tag availability
- Verify RawConv validation logic
- Monitor WriteAlso tag coordination
- Track Avoid flag priority conflicts

**Format String Processing:**

- CoordFormat option configuration
- Custom format specifier validation
- Round-off error in coordinate boundaries
- Precision handling in decimal formatting

**Conversion Function Debugging:**

- ToDMS format code validation (0-3)
- ToDegrees regex pattern matching
- Numeric extraction from formatted strings
- Scientific notation parsing

**MWG Compliance:**

- Camera vs subject location tag usage
- Proper tag pairing for coordinates
- Reference tag synchronization
- Group classification verification

The GPS module provides the foundation for location metadata handling across ExifTool, with robust coordinate conversion, flexible formatting, and comprehensive standard compliance for GPS data integration.
