# JSON.pm Module Outline

**ExifTool Version:** 13.26  
**Module Version:** 1.10  
**Document Version:** 1.0  
**Last Updated:** 2025-06-30

## Overview

The JSON module reads and extracts metadata from JSON files, providing flexible support for any JSON structure through dynamic tag creation. It handles nested objects, arrays, and complex data structures while maintaining compatibility with ExifTool's structured data features. The module includes specialized support for ON1 settings files and C2PA (Content Authenticity Initiative) metadata, demonstrating its adaptability to emerging metadata standards.

## Module Structure

### Tag Tables (1 total)

**Main Processing:**

- `Main` - Minimal predefined tags with dynamic tag creation capability

### Key Data Structures

**Predefined Tags:**

- `ON1_SettingsData` - Base64-encoded PLIST subdirectory
- `ON1_SettingsMetadata*` - ON1 application metadata (timestamps, names, usage)
- `adjustmentsSettingsStatisticsLightMap` - AAE file adjustment data

**Dynamic Tag System:**

- Temporary tag creation for unknown JSON keys
- Automatic tag name generation from JSON structure
- MakeTagName/MakeDescription conversion for consistent naming
- Special character handling (colons to underscores)

**Processing Features:**

- NO_ID flag for dynamic tag handling
- Temporary flag for runtime-created tags
- Structured data support via Struct option
- Array handling with List flag
- Hash flattening with Flat flag

## Processing Architecture

### 1. Main Processing Flow

**JSON File Recognition:**

- Uses Image::ExifTool::Import::ReadJSON for JSON parsing
- Supports direct file access or data block processing
- Block extraction capability for embedded JSON
- Charset option support for encoding handling

**Processing Sequence:**

1. Parse JSON file into database hash structure
2. Clean up temporary tags from previous processing
3. Iterate through JSON keys in sorted order
4. Process each tag through recursive structure handling
5. Generate dynamic tag definitions as needed
6. Store extracted values with appropriate flags

### 2. Special Processing Functions

**ProcessJSON($$)**

- Primary entry point for JSON file processing
- Handles both file and data block inputs
- Integrates with Image::ExifTool::Import::ReadJSON
- Manages block extraction and file type detection
- Coordinates database processing and tag extraction

**ProcessTag($$$$%)**

- Recursive processor for JSON structures
- Handles three data types: HASH, ARRAY, scalar values
- Implements structure flattening with tag name generation
- Manages Struct option for preserving object structure
- Applies appropriate flags (Struct, Flat, List) based on data type

**FoundTag($$$$%)**

- Dynamic tag creation and storage
- Handles special tag name transformations (ON1, C2PA)
- Prevents conflicts with ExifTool special tags
- Creates temporary tag definitions with name/description
- Integrates with ExifTool tag handling system

## Key Features

### 1. Dynamic Tag Creation System

**Automatic Tag Generation:**

- JSON keys converted to ExifTool tag names
- MakeTagName normalization for consistent naming
- Special character replacement (colons to underscores)
- Camel case conversion for hierarchical structures
- Temporary tag flagging for runtime definitions

**Tag Name Processing:**

- Hierarchical key flattening for nested objects
- Numeric key handling with underscore separation
- Capitalize first letter of sub-keys
- Non-alphabetic character capitalization
- Conflict resolution with special tags

### 2. Structured Data Support

**Structure Preservation:**

- Struct option enables object preservation
- Dual processing: structured + flattened tags
- Struct > 1 provides both representations
- Maintains JSON object hierarchy
- Supports complex nested structures

**Array Processing:**

- List flag for array elements
- Individual element processing
- Recursive array handling
- Maintains array order
- Supports mixed-type arrays

### 3. Specialized Format Support

**ON1 Settings Integration:**

- UUID pattern recognition for ON1 files
- Base64 decoding for settings data
- PLIST subdirectory processing
- File type override to ONP format
- Metadata timestamp handling

**C2PA Metadata:**

- Content Authenticity Initiative support
- Case normalization for C2PA tags
- Description formatting corrections
- Emerging standard compatibility

## Special Patterns

### 1. Recursive Structure Flattening

**Hierarchical Tag Names:**

```
JSON: {"camera": {"settings": {"iso": 100}}}
Result: CameraSettingsIso = 100
```

**Naming Convention:**

- Parent keys become tag prefixes
- Camel case for readability
- Underscores for numeric separation
- Capitalization after non-alphabetic characters

### 2. Option-Driven Processing

**Struct Option Levels:**

- Struct = 0: Flattened tags only
- Struct = 1: Structured tags only
- Struct > 1: Both structured and flattened
- Flexible metadata representation

**MissingTagValue Handling:**

- Set to "null" to ignore JSON nulls
- Configurable null value treatment
- Maintains JSON data integrity

### 3. Block Extraction Integration

**Embedded JSON Support:**

- BlockExtract option for JSON blocks
- BlockInfo integration for named blocks
- DirStart/DirLen for partial extraction
- REQ_TAG_LOOKUP and EXCL_TAG_LOOKUP filtering

## Usage Notes

### File Format Support

**JSON File Types:**

- Standard JSON files (.json)
- ON1 settings files (.on1, .onp)
- Adobe AAE files with JSON metadata
- Embedded JSON blocks in other formats

**Processing Modes:**

- Direct file processing via RAF
- Data block processing for embedded JSON
- Block extraction for selective processing
- Charset-aware text processing

### Metadata Extraction

**Data Type Handling:**

- Objects: Flattened to hierarchical tags
- Arrays: Individual elements with List flag
- Scalars: Direct value extraction
- Null values: Configurable handling

**Special Cases:**

- SourceFile tag filtering (auto-generated)
- ON1 UUID pattern recognition
- C2PA tag name normalization
- Temporary tag cleanup between files

## Debugging Tips

### Common Issues

**Missing Tags:**

- Check Struct option setting for object handling
- Verify MissingTagValue setting for null handling
- Ensure JSON file is properly formatted
- Check for character encoding issues

**Tag Name Problems:**

- Special characters converted to underscores
- Hierarchical names may be very long
- Case sensitivity in tag name generation
- Conflicts with ExifTool special tags

**Structure Processing:**

- Nested objects may create deep hierarchies
- Array processing creates multiple list entries
- Complex structures may need Struct > 1
- Temporary tags cleaned between files

### Advanced Debugging

**ON1 File Detection:**

- UUID pattern matching for settings files
- Base64 decoding for SettingsData
- File type override to ONP format
- PLIST subdirectory processing

**C2PA Integration:**

- Case normalization for tag names
- Description formatting corrections
- Emerging standard compatibility
- Special tag name handling

**Dynamic Tag Creation:**

- Temporary flag for runtime tags
- Tag table modification during processing
- Name/description generation consistency
- Conflict resolution with existing tags

**Block Processing:**

- DirStart/DirLen for partial extraction
- BlockInfo integration for named blocks
- REQ_TAG_LOOKUP filtering
- Block extraction option levels

The JSON module demonstrates ExifTool's flexibility in handling diverse metadata formats through dynamic tag creation, structured data support, and specialized processing for emerging standards like C2PA and application-specific formats like ON1 settings.
