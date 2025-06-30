# ICC_Profile.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 1.41
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

The ICC_Profile module is designed to read ICC (International Color Consortium) profile metadata from various file formats. ICC profiles are used for color management, translating color data between different devices' native color spaces.

The module supports reading ICC profile information from multiple file formats (JPEG, TIFF, PDF, PostScript, Photoshop, PNG, MIFF, PICT, QuickTime, XCF, and some RAW formats), processing embedded ICC profile data with type-specific formatting, extracting both header information and tag data, and supporting various ICC profile versions and specifications.

## Module Structure

### Tag Tables (8 total)

1. **%Main** - Primary ICC profile tag table with extensive manufacturer signatures
2. **%Header** - ICC profile header structure with version, class, and device information
3. **%ColorRep** - Color representation data and parameters
4. **%ViewingConditions** - Viewing conditions table for color management
5. **%Measurement** - Measurement parameters and geometry
6. **%Chromaticity** - Chromaticity data and color space information
7. **%ColorantTable** - Colorant information and specifications
8. **%Metadata** - Metadata dictionary for profile descriptions

### Key Data Structures

- **%illuminantType** - Standard illuminant types for color calculations
- **%profileClass** - ICC profile classes (input, display, output, etc.)
- **%manuSig** - Extensive manufacturer signatures lookup table (300+ entries)

## Processing Architecture

### 1. Main Processing Flow

The module processes ICC data in big-endian format throughout and uses an embedded type system where ICC profiles embed format type information within each tag. Processing involves sophisticated type-based formatting and validation of profile structure and data integrity.

### 2. Special Processing Functions

- **ProcessICC($$)** - Entry point for processing standalone ICC profile files
- **ProcessICC_Profile($$$)** - Main processor for ICC profile data within other file formats
- **WriteICC_Profile($$;$)** - Handles writing ICC profile data
- **ProcessMetadata($$$)** - Processes ICC metadata dictionary records
- **ValidateICC($)** - Validates ICC profile data structure
- **FormatICCTag($$$)** - Formats ICC tag values based on embedded type information

## Key Features

- **Embedded Type System** - ICC profiles embed format type information within each tag
- **Multi-Language Support** - Implements multiLocalizedUnicodeType (mluc) for internationalized text
- **Binary Data Handling** - Processes complex binary structures with variable-length arrays
- **Comprehensive Validation** - Profile structure validation with detailed error messages

## Special Patterns

- **Type-Based Formatting** - Uses sophisticated formatting system with ICC-specific data types
- **Big-Endian Processing** - Always processes ICC data in big-endian format
- **Multi-Language Text** - Supports language-specific tag variants and Unicode text
- **Binary Structure Processing** - Handles embedded subdirectories and complex data layouts

## Usage Notes

- Always processes ICC data in big-endian format
- Requires explicit extraction request for binary ICC_Profile data
- Supports both standalone ICC files and embedded profiles
- Handles multiple ICC directory instances with group numbering
- Large manufacturer signature lookup table affects memory usage

## Debugging Tips

- Verbose output shows processing steps and tag formatting details
- Detailed error messages provided for validation failures
- Support for unknown tag extraction when profile contains non-standard tags
- Binary data dumping capabilities for low-level analysis
- Complex type-based formatting may be slower than simple tag extraction