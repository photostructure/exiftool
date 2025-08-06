# ExifTool Composite Tag System Architecture

## Overview

Composite tags in ExifTool are calculated fields that derive their values from one or more source tags. They provide convenient computed values like GPS coordinates in decimal degrees, combined date/time fields, or derived camera settings.

## Architecture

### 1. Organization Structure

**Centralized Main Table**: The primary composite table is `%Image::ExifTool::Composite` in `lib/Image/ExifTool.pm` (around line 6800-7500). This contains most composite definitions.

**Module-Specific Tables**: Individual modules can define their own composite tables using the pattern `%Image::ExifTool::ModuleName::Composite`. Examples:
- `%Image::ExifTool::GPS::Composite` in `/lib/Image/ExifTool/GPS.pm` (lines 353-490)
- `%Image::ExifTool::Exif::Composite` in `/lib/Image/ExifTool/Exif.pm` (lines 4639-4720)
- `%Image::ExifTool::Canon::Composite` in `/lib/Image/ExifTool/Canon.pm` (lines 7800-8200)

### 2. Registration Process

**AddCompositeTags Function**: Located in `lib/Image/ExifTool.pm` at lines 5662-5720, this function:
1. Merges all composite tables into the main %Composite table
2. Processes dependency relationships (Require/Desire/Inhibit)
3. Sets up the composite tag infrastructure

**Module Registration**: Each module with composite tags calls:
```perl
Image::ExifTool::AddCompositeTags('Image::ExifTool::ModuleName');
```

### 3. Field Extractor Symbol Detection

The field extractor correctly identifies composite tables as `composite_hash` types and extracts them with `IsComposite:1` metadata. The composite data is stored directly in the symbol's data field, not nested under a "data" key.

### 4. Composite Tag Structure

Each composite tag definition includes:

- **Require**: Array of mandatory source tags
- **Desire**: Array of optional source tags that enhance calculation
- **Inhibit**: Array of tags that prevent composite calculation if present
- **ValueConv**: Perl expression for raw value calculation
- **PrintConv**: Perl expression for human-readable formatting
- **Groups**: Categorization info (typically Composite group)

## Implementation Details

### GPS Module Example

The GPS composite table (`lines 353-490 in GPS.pm`) defines 6 composite tags:

1. **GPSAltitude**: Combines altitude value and reference
   - Desire: GPS:GPSAltitude, GPS:GPSAltitudeRef, XMP:GPSAltitude, XMP:GPSAltitudeRef
   - Handles both GPS and XMP sources

2. **GPSDateTime**: Combines date and time stamps
   - Require: GPS:GPSDateStamp, GPS:GPSTimeStamp
   - Creates ISO format: "YYYY:mm:dd HH:MM:SSZ"

3. **GPSLatitude/GPSLongitude**: Convert DMS to decimal degrees
   - Require: coordinate + direction reference
   - Apply sign based on N/S or E/W reference

4. **GPSDestLatitude/GPSDestLongitude**: Similar to above for destination

### Canon Module Example

The Canon composite table (`lines 7800-8200 in Canon.pm`) includes advanced composites:

- **DriveMode**: Derives from ContinuousDrive + SelfTimer settings
- **Lens**: Focal length range calculation
- **ShootingMode**: Complex logic combining exposure mode and easy mode
- **ISO**: Calculated from BaseISO × AutoISO when CameraISO isn't numeric

## Key Patterns

### 1. Dependency Types
- **Require**: Must have ALL listed tags
- **Desire**: Uses any available tags from list
- **Inhibit**: Skips calculation if ANY listed tags exist

### 2. Value Processing
- **ValueConv**: Raw calculation (numbers, internal values)  
- **PrintConv**: Human-readable formatting
- **Writable**: Some composites can be written back to source tags

### 3. Priority System
- `Priority => 0`: Let other tags take precedence
- Higher numbers = higher priority

## Codegen Integration

### Field Extraction
The field extractor (`field_extractor.pl`) correctly identifies composite symbols:
- Type: `composite_hash`
- Data: Direct object containing tag definitions
- Metadata: Includes `IsComposite: 1` flag

### Strategy Processing  
The CompositeTagStrategy (`composite_tag.rs`) processes these symbols by:
1. Extracting tag definitions from the symbol data
2. Converting Perl expressions to documentation
3. Generating Rust composite tag definitions
4. Creating a global registry for runtime lookup

The key fix was correcting the data access pattern from `symbol.data.get("data")` to `symbol.data.as_object()` since the field extractor stores composite data directly, not nested.

## Files Referenced

- `/home/mrm/src/exif-oxide/third-party/exiftool/lib/Image/ExifTool.pm` (lines 5662-5720, 6800-7500)
- `/home/mrm/src/exif-oxide/third-party/exiftool/lib/Image/ExifTool/GPS.pm` (lines 353-490) 
- `/home/mrm/src/exif-oxide/third-party/exiftool/lib/Image/ExifTool/Exif.pm` (lines 4639-4720)
- `/home/mrm/src/exif-oxide/third-party/exiftool/lib/Image/ExifTool/Canon.pm` (lines 7800-8200)
- `/home/mrm/src/exif-oxide/codegen/src/strategies/composite_tag.rs` (lines 98-106)
- `/home/mrm/src/exif-oxide/codegen/scripts/field_extractor.pl` (lines 75-85)

## Summary

ExifTool's composite system uses a distributed architecture with module-specific tables merged into a central registry via `AddCompositeTags()`. The field extractor correctly identifies these as `composite_hash` symbols, and the codegen system successfully processes them once the data access pattern is corrected to match the actual field extractor output format.