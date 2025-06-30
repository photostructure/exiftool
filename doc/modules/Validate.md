# Validate.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 1.24
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

The Validate module provides additional metadata validation functionality for ExifTool. It's designed as a retro-fit feature that adds comprehensive validation checks while minimizing performance impact when not in use. The module focuses primarily on EXIF/TIFF metadata validation and ensures compliance with various standards.

Key validation areas include EXIF specification compliance, GPS metadata version compatibility, file format-specific tag validation, date/time format validation, tag ordering verification, and image data offset/size validation.

## Module Structure

### Tag Tables (5 total)

1. **%exifSpec** - Maps EXIF tag IDs to ExifVersion numbers (EXIF 2.32 spec compliance)
2. **%gpsVer** - Maps GPS tag IDs to GPSVersionID numbers for version compatibility  
3. **%verCheck** - Links directory types to their version checking mechanisms
4. **%otherSpec** - Defines standard tags for various RAW file formats (CR2, NEF, DNG, ARW, etc.)
5. **%stdFormat** - Expected data formats for tags in different IFD types

### Key Data Structures

- **%validValue** - File-type specific validation rules for JPEG and TIFF files
- **@validDateField** - Defines valid ranges for date/time components (Month: 1-12, Day: 1-31, etc.)
- **%validateInfo** - Configuration for the "Validate" tag that reports validation results

## Processing Architecture

### 1. Main Processing Flow

The module uses a retro-fit architecture that integrates into existing ExifTool processing without breaking changes. Validation is conditionally activated only when needed to avoid performance penalties.

### 2. Special Processing Functions

- **ValidateRaw($$$)** - Validates individual tag raw values, checks PrintConv lookups
- **ValidateExifDate($)** - Validates EXIF-format date/time strings (YYYY:MM:DD HH:MM:SS)
- **ValidateXMPDate($)** - Validates XMP-format date/time strings with timezone support
- **ValidateExif($$$$$$$$)** - Comprehensive EXIF tag validation (ordering, placement, format, count)
- **ValidateOffsetInfo($$$;$)** - Validates image data offsets and sizes against file boundaries
- **FinishValidate($$)** - Final validation pass that generates summary reports

## Key Features

- **Multi-level Validation**: Specification compliance, format-specific rules, version compatibility, physical validation
- **Warning System Integration**: Uses ExifTool's existing warning infrastructure with detailed error context
- **Retro-fit Design**: Integrates seamlessly without breaking existing functionality
- **Performance Optimized**: Minimal impact when validation is not requested

## Special Patterns

- **Conditional Activation**: Only runs when "Validate" tag is requested or API option is set
- **Version-aware Validation**: Checks tags against declared specification versions (EXIF 2.32, 3.0)
- **File-type Specific Rules**: Different validation logic for JPEG vs TIFF vs RAW formats
- **Cross-tag Validation**: Verifies relationships between related tags (offsets vs sizes)

## Usage Notes

- Automatically enabled when the "Validate" tag is requested
- Can be enabled via API `Validate` option
- Primary focus on EXIF/TIFF metadata
- Some file types excluded from validation (RWZ, 3FR, RWL, RW2)
- Accommodates some manufacturer-specific quirks (e.g., Minolta A200 byte order issues)

## Debugging Tips

- Detailed warning messages include hex tag IDs and names
- Context-aware error reporting includes IFD/group information
- Validation result summary available via "Validate" tag (reports errors/warnings count)
- Use with `-v` option for detailed validation process information
- Incomplete validation occurs when FastScan option is used