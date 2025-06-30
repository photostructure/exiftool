# TagLookup.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 1.20
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

The TagLookup module provides fast, case-insensitive lookup functionality for ExifTool tags, specifically optimized for writing operations. It serves as a performance-critical component that allows ExifTool to quickly find tag information without having to load and search through all format-specific modules.

Key purposes include fast tag name lookup for writing operations, case-insensitive tag name resolution, wildcard tag matching support, flattened tag support for structured data, and dynamic module loading for composite tags.

## Module Structure

### Tag Tables (Auto-Generated)

1. **%tagLookup** (~7,050 entries) - Main lookup hash mapping lowercase tag names to table numbers and tag IDs
2. **%tagExists** (~5,680 entries) - Hash of non-writable tag names for existence checking
3. **@tableList** (~590 tables) - Ordered list of all tag table names containing writable tags
4. **%compositeModules** - Maps composite tag names to their required modules for dynamic loading
5. **%specialStruct** - Defines special keys in structure definitions (NAMESPACE, STRUCT_NAME, etc.)

### Key Data Structures

- **%tableNumHash** - Runtime mapping of table names to their numeric indices
- Auto-generated lookup tables containing ~90% of the module's 13,644 lines
- Optimized for O(1) lookups for direct tag names

## Processing Architecture

### 1. Main Processing Flow

The module uses auto-generated lookup tables created by BuildTagLookup.pm. Primary workflow involves fast hash lookups with fallback to wildcard pattern matching and dynamic module loading when needed.

### 2. Special Processing Functions

- **FindTagInfo($tag)** - Primary lookup function supporting wildcards and dynamic loading
- **TagExists($tag)** - Fast boolean existence check for tag names
- **AddTags($tagTablePtr, $table)** - Adds user-defined tags to lookup structures
- **AddFields($tagTablePtr, $tagID, $lcTag, ...)** - Handles structured tag field flattening

## Key Features

- **Wildcard Pattern Matching** - Converts shell-style wildcards (`*`, `?`) to regex patterns
- **Dynamic Module Loading** - On-demand loading of specific modules for composite tags
- **Flattened Tag Support** - Complex handling of structured tags (primarily XMP)
- **Table Number Optimization** - Uses numeric indices instead of full table names for space efficiency

## Special Patterns

- **Auto-Generated Code Sections** - Large sections marked with build markers containing lookup tables
- **Lazy Evaluation** - Flattened tags and table mappings initialized on demand
- **Memory vs Speed Trade-off** - Large lookup tables for extremely fast tag resolution
- **Wildcard Search Fallback** - Full table scanning when exact matches fail

## Usage Notes

- Primarily used internally by ExifTool's writing functions
- Export functions: `FindTagInfo` and `TagExists`
- Supports both exact matches and wildcard patterns
- The lookup tables are rebuilt by `build_tag_lookup` script
- Adding new tags requires regenerating this module
- User-defined tags are added at runtime via `AddTags()`

## Debugging Tips

- Use `TagExists()` to verify tag names exist in the system
- Test wildcard patterns like `"gps*"` or `"canon?"` in `FindTagInfo()`
- Check `@tableList` to see which tables are included in lookups
- Monitor which modules get dynamically loaded during composite tag resolution
- Structure flattening happens lazily to improve startup performance
