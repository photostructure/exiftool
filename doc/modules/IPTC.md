# IPTC.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 1.59
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

The IPTC module handles reading and writing IPTC (International Press Telecommunications Council) metadata from image files. IPTC is a standardized format for embedding descriptive information in digital images, commonly used by news organizations, photographers, and media companies for cataloging and distributing images.

Key capabilities include reading IPTC metadata from various file formats, supporting all IPTC record types, handling character set encoding/decoding including UTF-8, validating IPTC data structure, and supporting both reading and writing operations.

## Module Structure

### Tag Tables (10 total)

1. **%Main** - Top-level dispatch table routing to specific record handlers
2. **%EnvelopeRecord** (Record 1) - File envelope information like format, dates, character sets
3. **%ApplicationRecord** (Record 2) - Core metadata like keywords, caption, byline, location
4. **%NewsPhoto** (Record 3) - Technical image parameters for news photos
5. **%PreObjectData** (Record 7) - Pre-object data information
6. **%ObjectData** (Record 8) - Object data handling
7. **%PostObjectData** (Record 9) - Post-object data information
8. **%FotoStation** (Record 240) - Proprietary FotoStation data
9. **%Composite** - Composite tags combining date/time values

### Key Data Structures

- **%iptcCharset** - Maps IPTC character set escape sequences to encoding names
- **%isStandardIPTC** - Defines standard IPTC locations per MWG specification
- **%fileFormat** - Maps file format codes to descriptive names

## Processing Architecture

### 1. Main Processing Flow

The module uses a record-based architecture with numeric record identifiers (1, 2, 3, 7, 8, 9, 240) to organize different types of metadata. Each record has its own tag table with specific formatting rules and hierarchical processing through subdirectory structures.

### 2. Special Processing Functions

- **ProcessIPTC($$$)** - Main reading function parsing IPTC data streams with validation
- **WriteIPTC($$$)** - Writing function (autoloaded when needed)
- **CheckIPTC($$$)** - Validation function (autoloaded when needed)
- **PrintCodedCharset($)** - Converts IPTC character set escape sequences to readable format
- **HandleCodedCharset($$)** - Determines if character set translation is needed
- **TranslateCodedString($$$$)** - Performs character encoding/decoding operations

## Key Features

- **Character Set Handling** - Sophisticated encoding system supporting ISO 2022 escape sequences
- **Standard vs Non-Standard IPTC** - MWG compliance checking with integrity validation
- **Data Validation** - Comprehensive format validation with automatic error correction
- **Multi-format Support** - Works with JPEG, TIFF, PSD, and other file types

## Special Patterns

- **ISO 2022 Character Encoding** - Advanced character set handling with escape sequences
- **MWG Compliance Checking** - Tracks standard locations and warns about non-standard placements
- **Automatic Error Correction** - Detects and corrects improperly byte-swapped IPTC data
- **Group-based Organization** - Non-standard IPTC gets numbered group names (IPTC1, IPTC2, etc.)

## Usage Notes

- UTF-8 should be set via CodedCharacterSet tag for special characters
- Module automatically detects and corrects byte-swapped data
- Standard compliance warnings appear automatically for non-standard locations
- Module handles improperly terminated strings with warnings
- Character encoding issues visible with `-v` verbose mode

## Debugging Tips

- Use `-v` verbose mode to see character set translation details
- Enable validation mode to catch malformed IPTC data
- Test with `t/IPTC.t` script and expected outputs in `t/IPTC_*.out`
- Module warns about non-standard IPTC locations automatically
- Check for multiple instances when IPTC appears in non-standard locations
