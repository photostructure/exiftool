# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ExifTool is a comprehensive Perl library and command-line application for reading, writing, and editing meta information in image, audio, and video files. It supports hundreds of file formats and thousands of tags.

## Development Commands

```bash
# Generate Makefile
perl Makefile.PL

# Run all tests
make test

# Run a specific test
perl t/Canon.t

# Run ExifTool without installation (from project root)
./exiftool t/images/ExifTool.jpg

# Validate code before commits (checks for common issues)
./validate

# Build tag lookup tables (required after adding new tags)
./build_tag_lookup

# Build geolocation database
./build_geolocation
```

## Running ExifTool

ExifTool is a command-line application for reading, writing, and editing metadata in over 150 file formats. It requires Perl 5.004+ or can be run using the Windows executable.

### Basic Usage

```bash
exiftool [options] filename
```

### Key Command Options

- `-h`: Display help
- `-s`: Show tag names instead of descriptions
- `-G`: Display group names
- `-a`: Extract all tags (including duplicates)
- `-n`: Disable print conversion
- `-e`: Exclude composite tags
- `-v`: Verbose output (use multiple times for more detail)
- `-fast`: Improve speed by skipping certain metadata types

### Writing Metadata

- `TAG=value`: Set metadata value
- `TAG+=value`: Add to list-type tag
- `TAG-=value`: Remove from list-type tag
- `TAG<=file`: Set value from file contents
- `-overwrite_original`: Overwrite file without backup

### Common Examples

```bash
# Read all metadata
exiftool image.jpg

# Write artist name
exiftool -Artist="Phil Harvey" image.jpg

# Shift all date/time values back by 1 hour
exiftool "-AllDates-=1" directory

# Rename files based on creation date
exiftool "-FileName<CreateDate" -d "%Y%m%d_%H%M%S.%%e" directory

# Copy metadata from one file to another
exiftool -TagsFromFile source.jpg target.jpg

# Extract specific tags only
exiftool -Artist -CreateDate image.jpg

# Write tags in batch
exiftool -Artist="Name" -Copyright="2024" *.jpg
```

### Performance Tips

- Process multiple files in a single command for better performance
- Use `-fast` options to skip certain metadata types when speed is critical
- Specify only needed tags with tag names to reduce processing
- Disable print conversions (`-n`) and composite tags (`-e`) if not required

## Developer Documentation

For detailed tag development reference, see `lib/Image/ExifTool/README` - comprehensive documentation on tag table structure, special keys, format types, and conversion functions.

## Module Documentation

**Always check the module documentation first** -- ExifTool source files
are very large, and ReadTool doesn't do a good job at providing summarizing
context for these files.

For all tasks, first study the relevant documentation:

- **`doc/concepts/MODULE_OVERVIEW.md`** - a categorized overview of ExifTool library modules
- **`doc/concepts/PATTERNS.md`** - Common patterns across all modules
- **`doc/concepts/*.md`** - Additional high-level concepts, explaining binary tag parsing, character sets, file type extraction, ValueConf, PrintConf, PROCESS_PROC, and WRITE_PROC.
- **`doc/modules/*.md`** - Individual module outlines for major modules (when available)

**If you don't find a document that provides the context you need for your task, YOU MUST ask the user to produce that documentation for you before you continue.**

The prompt to generate that document in a separate claude code will be something like `

## Architecture and Code Organization

### Core Structure

- **`exiftool`**: Main command-line script that parses arguments and calls the API
- **`lib/Image/ExifTool.pm`**: Core API module that coordinates all operations
- **`lib/Image/ExifTool/*.pm`**: Format-specific modules (e.g., Canon.pm, Nikon.pm, XMP.pm)
- **`t/`**: Test suite with `.t` files and test images in `t/images/`

### Key Architectural Patterns

**Tag Table Structure**: Each format module defines tag tables as Perl hashes with special keys (see README for complete list of 28 special keys):

- `PROCESS_PROC`: Function to read the format
- `WRITE_PROC`: Function to write the format
- `CHECK_PROC`: Function to validate values
- `GROUPS`: Group hierarchy for tag organization
- `NOTES`: Documentation for the table
- `FORMAT`: Default tag format for BinaryData tables
- `WRITABLE`: Indicates writable tags in table

**Processing Flow**:

1. File type detection based on magic numbers/extensions
2. Dispatch to appropriate format module
3. Recursive processing of subdirectories/sub-IFDs
4. Value extraction with format-specific conversions
5. Writing back changes while preserving unknown data

### Important Data Structures

**ExifTool Object Members**:

- `$$self{VALUE}`: Hash of extracted tag values
- `$$self{FILE_TYPE}`: Detected file type
- `$$self{NEW_VALUE}`: Hash of new values to write
- `$$self{TAGS_FROM_FILE}`: Source file for SetNewValuesFromFile

**Directory Info Hash** (passed to process functions):

- `DataPt`: Reference to data block
- `DirStart`: Start offset of directory in data
- `DataPos`: File position of data block
- `Base`: Base offset for pointer calculations

## Testing

Tests use the `t::TestLib` module for utilities. Test pattern:

- Test script: `t/FormatName.t`
- Expected outputs: `t/FormatName_N.out`
- Test images: `t/images/`

When adding features, always add corresponding tests with expected output files.

## Code Style and Conventions

- Use strict mode and warnings in all modules
- Tag names use CamelCase starting with uppercase
- Only `[A-Za-z0-9_-]` allowed in tag names
- Binary data represented as SCALAR references
- Undefined conversion values indicate tag should be deleted
- Use `$$self` for object member access (not `$self->`)

## Common Development Tasks

### Adding a New Tag

1. Add entry to appropriate tag table in format module (see README for tag info hash structure)
2. Set format, groups, and conversion functions (RawConv/ValueConv/PrintConv)
3. Run `./build_tag_lookup` to update lookup tables
4. Add test case with expected output
5. Run `make test` to verify

### Adding a New File Format

1. Create module in `lib/Image/ExifTool/FormatName.pm`
2. Define tag tables with PROCESS_PROC
3. Add to `%Image::ExifTool::fileTypeLookup` in ExifTool.pm
4. Add to `@loadAllTables` array
5. Create comprehensive test coverage

### Debugging

- Use `-v` option for verbose output showing processing steps
- Use `-htmlDump` to visualize binary file structure
- Check `$$self{WARNINGS}` for non-fatal issues
- Use `Image::ExifTool::HexDump()` for debugging binary data
