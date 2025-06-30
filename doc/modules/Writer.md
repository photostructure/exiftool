# Writer.pl Module Outline

**ExifTool Version:** 13.26
**Module Version:** No explicit version (core module)
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

Writer.pl is the core metadata writing engine for ExifTool. It contains all write routines for modifying metadata in image, audio, and video files, less frequently used ExifTool utility functions, directory mapping and write coordination logic, value validation and conversion functions, and file format detection and routing for write operations.

The module serves as the central dispatcher that coordinates writing operations across all supported file formats.

## Module Structure

### Directory Mapping Tables (6 total)

1. **%tiffMap** - Defines where to write directories in TIFF-based files
2. **%exifMap** - Maps EXIF directory locations for EXIF-only files
3. **%jpegMap** - Complete JPEG directory mapping (extends exifMap)
4. **%dirMap** - Master mapping by file type (JPEG, TIFF, ORF, RAW, EXIF, EXV)
5. **%writableType** - Maps file extensions to modules and write functions (24 writable formats)
6. **%rawType** - Identifies RAW file types and maker note deletion capabilities

### Key Data Structures

- **%blockExifTypes** - File types supporting block-writable EXIF
- **@delGroups** - 47 deletable metadata groups
- **%delMore** - Additional groups to delete when deleting parent groups
- **%excludeGroups** - Dependency relationships between groups
- **%intRange** - Min/max values for integer formats
- **%translateWriteGroup** - Group name translations for writing

## Processing Architecture

### 1. Main Processing Flow

The module uses hierarchical directory mapping with nested hash structures defining parent-child relationships between metadata directories across different file formats, enabling format-agnostic write operations.

### 2. Special Processing Functions

- **WriteInfo($$;$$)** - Primary write function coordinating entire write operation
- **SetNewValue($;$$%)** - Sets new metadata values with validation and options
- **SetNewValuesFromFile($$;@)** - Copies metadata from source to destination files
- **WriteDirectory($$$;$)** - Writes directory structures for various formats
- **WriteValue($$;$$$$)** - Writes individual tag values with format conversion
- **WriteBinaryData($$$)** - Handles binary data structures

## Key Features

- **Multi-Format Support** - Handles 24+ different file formats with format-specific logic
- **Group-Based Operations** - Sophisticated group dependency tracking for metadata consistency
- **Inverse Conversion Chain** - Complete inverse conversion pipeline (PrintConv → ValueConv → RawConv → binary)
- **Write Optimization** - In-place file operations when only pseudo-tags are being modified
- **Value Validation** - Comprehensive validation and sanitization of input values

## Special Patterns

- **Hierarchical Directory Mapping** - Nested structures defining metadata directory relationships
- **Group Dependency Management** - Ensures metadata consistency when deleting related groups
- **Multi-Value Handling** - Supports both scalar and array references with automatic conversion
- **Format-Agnostic Operations** - Common interface for diverse file format requirements

## Usage Notes

- Always use `SetNewValue()` to set metadata - direct hash manipulation breaks validation
- Check `$$self{NEW_VALUE}` hash structure to understand pending changes
- Use `CountNewValues()` to verify operations before writing
- Test with `IsWriting()` flag to distinguish read vs. write operations
- File format detection occurs early to route to appropriate write handlers

## Debugging Tips

- Enable verbose mode (`-v` option) to trace write operations
- Check `$$self{CHANGED}` counter to verify modifications occurred
- Use `$$self{DEL_GROUP}` hash to track group deletions
- Monitor `$$self{NEW_VALUE}` structure for pending value changes
- Be careful with binary data handling and byte ordering
- Account for group dependencies when deleting metadata
