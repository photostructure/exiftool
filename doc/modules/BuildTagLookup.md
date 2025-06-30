# BuildTagLookup.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 3.62
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

The BuildTagLookup module is a utility module that serves as the build system for ExifTool's tag lookup infrastructure. This is not a runtime module - it's a build-time utility that processes all ExifTool tag tables to create optimized lookup structures and comprehensive documentation.

Key purposes include generating tag lookup tables for TagLookup.pm, creating documentation in POD and HTML formats, building cross-reference indexes for all supported metadata tags, validating tag table consistency across modules, and generating the complete TagNames documentation.

## Module Structure

### Core Object Members

- **TAG_NAME_INFO** - Complete tag information for all tables
- **TAG_LOOKUP** - Optimized lookup hash for tag resolution  
- **TAG_EXISTS** - Quick existence check for tags
- **TABLE_WRITABLE** - Tracks which tables contain writable tags
- **FLATTENED** - Flattened tag information for complex structures
- **STRUCTURES** - XMP and other structural metadata definitions
- **COUNT** - Statistics (total tags, unique names, etc.)

### Key Data Structures

- **%tweakOrder** - Controls table ordering in documentation (68 entries)
- **%formatOK** - Validates supported data formats (30+ formats)
- **%docs** - Contains all documentation templates and descriptions
- **%shortcutNotes** - Documentation for shortcut tags

## Processing Architecture

### 1. Main Processing Flow

The build process loads ALL ExifTool modules, iterates through every tag table in dependency order, extracts tag metadata and validates formats, handles special cases (shortcuts, plugins, composite tags), and generates optimized lookup structures.

### 2. Special Processing Functions

- **new()** - Constructor that processes all ExifTool tables and builds lookup structures
- **WriteTagLookup()** - Generates the optimized TagLookup.pm file
- **WriteTagNames()** - Creates complete POD and HTML documentation
- **NumbersFirst()** - Custom sorting for tag IDs (numbers vs. strings)
- **GetTableOrder()** - Determines documentation table sequence

## Key Features

- **Dual-Format Documentation** - POD for man pages, HTML for web
- **Template-Based System** - Variable substitution and automatic cross-referencing
- **Plugin Module Support** - Handles non-core modules
- **XMP Structure Processing** - Flattening and validation of complex structures
- **Statistical Reporting** - Comprehensive analysis of tag coverage

## Special Patterns

- **Table Processing Pipeline** - Multi-stage processing of all ExifTool modules
- **Advanced Documentation Generation** - Custom sorting, grouping, and linking logic
- **Comprehensive Validation** - Format checking, DICOM UID validation, consistency checks
- **Memory-Intensive Processing** - Loads entire ExifTool codebase for introspection

## Usage Notes

- **Build-time only** - Not loaded during normal ExifTool operation
- **Invoked by** - `./build_tag_lookup` script in ExifTool root directory
- **Output files** - `lib/Image/ExifTool/TagLookup.pm`, POD docs, HTML docs
- **Very memory-intensive** - Loads entire ExifTool codebase for processing
- **Should only be run when tag tables change**

## Debugging Tips

- Module sets `$Image::ExifTool::debug = 1` automatically for verbose output
- Use `%count` statistics for validation of processing results
- HTML output provides visual verification of table structures  
- Temporary files created during generation for safety
- Requires ALL ExifTool modules to be loadable for proper operation
- Processing time proportional to number of supported formats