# MWG.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 1.24
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

The MWG (Metadata Working Group) module provides Composite tag definitions designed to simplify implementation of MWG 2.0 guidelines. It creates a unified interface for accessing metadata across EXIF, IPTC, and XMP formats while ensuring compliance with MWG recommendations for metadata consistency and synchronization.

Key capabilities include unified metadata access across formats, automatic IPTC digest reconciliation, strict MWG conformance mode, IPTC character length recovery, EXIF Artist list conversion, and hierarchical keyword/region support.

## Module Structure

### Tag Tables (4 total)

1. **%Composite** - Main MWG composite tags (15 tags: Keywords, Description, DateTimeOriginal, CreateDate, ModifyDate, Orientation, Rating, Copyright, Creator, Country, State, City, Location)
2. **%Regions** - MWG 2.0 XMP region metadata (mwg-rs namespace)
3. **%Keywords** - MWG 2.0 XMP hierarchical keywords (mwg-kw namespace) with 6 levels of nesting
4. **%Collections** - MWG 2.0 XMP collections metadata (mwg-coll namespace)

### Key Data Structures

- **%sExtensions** - Structure for MWG Extensions (variable namespace)
- **%sRegionStruct** - MWG RegionStruct with Area, Type, Name, Description, FocusUsage, BarCodeValue, Extensions, Rotation
- **%sKeywordStruct** - MWG KeywordStruct with Keyword, Applied, Children (recursive structure)
- **$mwgLoaded** - Global flag tracking if MWG tags have been loaded

## Processing Architecture

### 1. Main Processing Flow

The module uses a composite tag approach where each MWG tag derives its value from multiple underlying EXIF, IPTC, and XMP sources based on MWG priority rules. When writing, the module updates all associated tags simultaneously while maintaining synchronization.

### 2. Special Processing Functions

- **Load()** - Loads MWG Composite tags and enables strict mode
- **ListToString($)** - Converts value lists to MWG-compliant strings with quote handling
- **StringToList($$)** - Parses MWG-compliant strings back to value lists
- **OverwriteStringList($$$$)** - Handles list-type tag overwriting logic
- **ReconcileIPTCDigest($)** - Updates IPTC digest after MWG tag changes
- **RecoverTruncatedIPTC($$$)** - Recovers IPTC data truncated by length limits

## Key Features

- **Unified Metadata Access** - Single tags accessing EXIF, IPTC, and XMP equivalents
- **Automatic Synchronization** - IPTC digest reconciliation ensures metadata consistency
- **Strict Conformance Mode** - Ignores non-standard metadata locations per MWG guidelines
- **IPTC Recovery** - Recovers truncated IPTC data using XMP equivalents when available
- **List Handling** - Sophisticated string-list conversion with quote escaping
- **Hierarchical Structures** - Support for complex keyword hierarchies and image regions

## Special Patterns

- **Priority-Based Value Selection** - Each tag uses specific priority order (EXIF > XMP > IPTC recovery)
- **IPTC Digest Synchronization** - Automatic digest updates when IPTC-related tags change
- **String-List Conversion** - MWG-compliant semicolon-space separation with quote handling
- **Character Limit Recovery** - Uses XMP data to recover IPTC data truncated by format limits
- **Conditional Writing** - Tags only write to IPTC if original file contained IPTC data
- **Artist List Modification** - Converts EXIF:Artist from string to list-type tag

## Usage Notes

- Must be explicitly loaded via `Image::ExifTool::MWG::Load()` or automatic loading when MWG group specified
- Loading enables strict MWG conformance mode by default (`$Image::ExifTool::MWG::strict = 1`)
- Strict mode can be disabled by setting `$Image::ExifTool::MWG::strict = 0`
- Automatically sets EXIF string encoding to UTF-8 in exiftool application
- Only writes IPTC if original file contained IPTC data
- IPTC digest automatically updated when IPTC-synchronized tags change

## Debugging Tips

- Check `$mwgLoaded` flag to verify MWG tags are loaded
- Monitor strict mode with `$Image::ExifTool::MWG::strict` value
- Use `-v` verbose mode to see MWG tag derivation and writing operations
- IPTC digest warnings indicate synchronization issues
- Quote parsing errors generate warnings for malformed string-list values
- WriteAlso operations show in verbose output for all associated tag updates
