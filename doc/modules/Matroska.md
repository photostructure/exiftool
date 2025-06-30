# Matroska.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 1.18
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

The Matroska.pm module provides comprehensive metadata extraction from Matroska multimedia container files (MKV, MKA, MKS, WEBM). It implements a complete EBML (Extensible Binary Meta Language) parser with advanced features including hierarchical structure processing, seek optimization, multi-language tag support, spherical video handling, and sophisticated performance optimizations for large multimedia files.

## Module Structure

### Tag Tables (3 total)

1. **Main** - Complete Matroska element definitions (400+ elements)

   - EBML Header: Version and document type information
   - Segment Info: Duration, timestamps, application info
   - Tracks: Video/audio stream metadata with codec information
   - Cluster: Media data organization (typically skipped for performance)
   - Cues: Seek tables for random access
   - Attachments: Embedded files and fonts
   - Chapters: Navigation and chapter information
   - Tags: Standardized metadata tags
   - Spherical Video: VR content projection data

2. **Projection** - Spherical Video V2 metadata (5 fields)

   - Projection type (rectangular, equirectangular, cubemap, mesh)
   - Pose information (yaw, pitch, roll)
   - Projection-specific private data

3. **StdTag** - Standardized Matroska tags (100+ fields)
   - Media metadata (title, artist, genre, etc.)
   - Technical information (BPS, FPS, encoding settings)
   - Rights management (copyright, license, terms)
   - Multi-language support with BCP47 codes

### Key Data Structures

- **VInt Processing**: Variable-length integer encoding/decoding
- **EBML Hierarchy**: Nested element structure with boundary tracking
- **Seek Information**: Random access tables for efficient navigation
- **Track Grouping**: Dynamic group assignment by track/chapter numbers
- **SimpleTag Structures**: Nested tag processing with language support
- **Timecode Scaling**: Global time base for duration calculations

## Processing Architecture

### 1. Main Processing Flow

Matroska processing follows EBML specification precisely:

1. **Header Validation**: Verifies EBML signature and header structure
2. **Element Parsing**: Processes variable-length element headers
3. **Hierarchical Navigation**: Maintains element boundary stack
4. **Seek Head Processing**: Extracts seek tables for optimization
5. **Track Analysis**: Identifies media types for file type determination
6. **Tag Structure Handling**: Processes nested SimpleTag elements
7. **Performance Optimization**: Skips Cluster data unless explicitly requested

### 2. Special Processing Functions

- **ProcessMKV**: Main EBML parser with complete element handling
- **GetVInt**: Variable-length integer decoder implementing EBML VInt specification
- **HandleStruct**: Complex nested structure processor for SimpleTag elements
- **Extensive SubDirectory Processing**: Hierarchical element navigation
- **Buffer Management**: 64kB chunk processing with dynamic expansion

## Key Features

### 1. Advanced EBML Implementation

- **Variable-Length Integers**: Complete VInt encoding/decoding support
- **Hierarchical Structure**: Nested element processing with boundary tracking
- **Unknown Element Handling**: Graceful processing of undefined elements
- **Format Validation**: Comprehensive header and structure validation

### 2. Performance Optimization System

- **Cluster Skipping**: Stops at first Cluster unless verbose/unknown/embedded options used
- **Seek Head Utilization**: Jumps directly to Tags section when available
- **Large Block Handling**: Intelligent skipping of unknown or oversized elements
- **Buffer Management**: Dynamic buffer expansion with 64kB increments
- **LargeFileSupport**: Configurable handling of very large multimedia files

### 3. Multi-Language Tag Support

- **BCP47 Language Codes**: Full language and country code support
- **Nested Tag Structures**: Complex hierarchical tag processing
- **Language Inheritance**: Parent-child language code propagation
- **Dynamic Tag Creation**: Automatic tag table expansion for unknown tags

### 4. Track and Media Analysis

- **Track Type Detection**: Video, audio, subtitle, and complex track identification
- **File Type Override**: Automatic MKA/MKS detection based on track content
- **Codec Information**: Comprehensive codec ID and name extraction
- **Track Grouping**: Dynamic GROUP1 assignment for organization

### 5. Spherical Video Support

- **Projection Types**: Rectangular, equirectangular, cubemap, mesh support
- **Pose Information**: Yaw, pitch, roll orientation data
- **QuickTime Integration**: Reuses QuickTime projection processing
- **Spatial Media RFC**: Implements Google's Spherical Video V2 specification

### 6. Advanced Navigation Features

- **Chapter Processing**: Time-based chapter navigation with numbering
- **Seek Tables**: Cue point processing for random access
- **Timecode Scaling**: Global time base conversion for accurate timing
- **Duration Calculation**: Proper duration handling with timecode scaling

## Special Patterns

### 1. Variable-Length Integer Processing

```perl
sub GetVInt($$) {
    # Custom VInt decoder implementing EBML specification
    # Returns value and updates position pointer
}
```

### 2. Hierarchical Structure Management

```perl
push @dirEnd, [ $pos + $dataPos + $size, $dirName, $struct ];
# Tracks nested element boundaries with size and structure info
```

### 3. Track-Dependent Processing

```perl
Condition => '$$self{TrackType} and $$self{TrackType} == 0x01',
# Video-specific processing based on track type
```

### 4. Seek Head Optimization

```perl
if ($seek{Tags} and $seek{Tags} > $pos + $dataPos) {
    $raf->Seek($seek{Tags},0);  # Jump directly to Tags section
}
```

### 5. SimpleTag Structure Handling

```perl
sub HandleStruct($$;$$$$) {
    # Complex nested tag processing with language support
    # Recursive structure flattening
}
```

## Usage Notes

### 1. Performance Considerations

- Cluster data processing disabled by default for speed
- Use -v, -U, or -ee options to enable full processing
- Large files may require LargeFileSupport option
- Seek heads optimize access to distant metadata

### 2. Multi-Language Support

- Language codes follow BCP47 specification
- Country codes combined with language for regional variants
- Default language is 'eng' when not specified
- Tag inheritance follows parent-child relationships

### 3. Track Type Dependencies

- Some tags only available for specific track types
- Video/audio tracks have different codec tag sets
- Track grouping affects metadata organization
- File type may be overridden based on track content

### 4. Spherical Video Requirements

- Projection tags require proper ProjectionType identification
- Spherical video XML processing integrated with XMP module
- Both upper and lowercase spherical-video tags supported
- Pose information requires floating-point processing

## Debugging Tips

### 1. EBML Structure Issues

- Use verbose mode to see element hierarchy and sizes
- Check VInt decoding for proper variable-length integer handling
- Monitor @dirEnd stack for boundary tracking
- Verify element size calculations and buffer management

### 2. Performance Problems

- Check Cluster processing settings (verbose/unknown/embedded)
- Monitor seek head utilization for optimization
- Verify large block skipping behavior
- Check buffer expansion and memory usage

### 3. Tag Processing Issues

- Verify SimpleTag structure handling in verbose mode
- Check language code processing and inheritance
- Monitor nested structure flattening
- Validate tag name normalization and creation

### 4. Track Analysis Problems

- Check TrackType detection and storage
- Verify conditional tag processing based on track type
- Monitor file type override logic
- Validate track grouping and GROUP1 assignment

### 5. Seek and Navigation Issues

- Check seek head processing and table creation
- Verify timecode scaling application
- Monitor chapter processing and numbering
- Validate duration calculations with timecode scale

### 6. Spherical Video Debugging

- Verify ProjectionType detection and storage
- Check projection-specific subdirectory processing
- Monitor QuickTime module integration
- Validate pose information extraction

### 7. Multi-Language Debugging

- Check BCP47 language code parsing
- Verify language/country code combination
- Monitor tag inheritance in nested structures
- Validate GetLangInfo integration

### 8. Large File Handling

- Monitor LargeFileSupport option behavior
- Check large block skipping logic
- Verify buffer management and expansion
- Watch for memory usage with very large files

### 9. Structure Validation

- Use -htmlDump to visualize EBML structure
- Check element nesting and boundary calculations
- Verify unknown element handling
- Monitor structure completion and cleanup
