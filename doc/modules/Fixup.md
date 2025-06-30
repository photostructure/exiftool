# Fixup.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 1.06
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

The Fixup module is a specialized utility class designed to handle pointer fixups in binary data structures. Its primary purpose is to track memory pointers that need to be adjusted when data blocks are moved or resized, maintain pointer relationships across different byte orders, support hierarchical pointer structures, and handle both 16-bit and 32-bit pointers.

This module is essential for ExifTool's ability to safely modify binary metadata structures (like EXIF IFDs) without breaking internal pointer references.

## Module Structure

### Core Object Members

- **Start** - Position in data where a zero pointer points to (base offset)
- **Shift** - Amount to shift pointer values (relative to Start)
- **Fixups** - Array reference of sub-Fixup objects (hierarchical structure)
- **Pointers** - Hash of pointer arrays, keyed by byte order string with format modifiers

### Key Data Structures

- Supports `'int16u'` and `'int32u'` pointer formats (default is 32-bit unsigned)
- Byte order strings: `"II"` (little-endian) or `"MM"` (big-endian)
- 16-bit modifier: `"2"` suffix (e.g., `"II2"`, `"MM2"`)
- Marker suffix: `"_$marker"` (e.g., `"II_thumbnail"`)

## Processing Architecture

### 1. Main Processing Flow

The module uses hierarchical fixup management with nested fixup structures where sub-blocks have their own pointer requirements. It collapses hierarchy into flat lists during `ApplyFixup()` for efficiency while maintaining parent-child relationships until fixup application.

### 2. Special Processing Functions

- **new()** - Constructor that initializes Start=0, Shift=0
- **Clone()** - Deep copy including all pointers and sub-fixups
- **AddFixup($pointer, $marker, $format)** - Adds pointer offset or sub-fixup object
- **ApplyFixup($dataPt)** - Recursively applies all fixups to binary data
- **IsEmpty()** - Returns true if no pointers need fixing
- **HasMarker($marker)** - Checks if marker exists in fixup hierarchy
- **SetMarkerPointers()** - Sets all tagged pointers to specific value
- **GetMarkerPointers()** - Retrieves values of tagged pointers

## Key Features

- **Byte Order Awareness** - Stores byte order with each pointer group to handle mixed-endian data
- **Marker System** - Allows tagging pointer groups with string markers for selective operations
- **Two-Phase Adjustment** - Start adjustment (data movement) and Shift adjustment (data insertion/deletion)
- **Hierarchical Support** - Manages nested fixup structures for complex binary formats

## Special Patterns

- **Hierarchical Fixup Management** - Supports nested structures with recursive processing
- **Byte Order Context Switching** - Automatically handles different endianness requirements
- **Marker-Based Operations** - Enables selective operations on pointer subsets
- **State Management** - Maintains state across multiple operations until fixup application

## Usage Notes

- Typical usage: Create → Add pointers/sub-fixups → Adjust for changes → Apply fixups
- Byte order must be set correctly when adding pointers (`GetByteOrder()`)
- Markers are case-sensitive string identifiers
- Fixup objects are consumed during `ApplyFixup()` (hierarchy is flattened)
- Supports negative offsets for flexible pointer calculations
- Thread-safe as long as byte order context is managed properly

## Debugging Tips

- Use `Dump()` method for hierarchical view of all fixups and pointers
- Check `IsEmpty()` for quick verification if fixup contains any work
- Verbose output shows byte order, pointer locations, and hierarchy
- Common use cases: EXIF IFD pointer management, TIFF structure maintenance, thumbnail pointer adjustments, MakerNote structure preservation
