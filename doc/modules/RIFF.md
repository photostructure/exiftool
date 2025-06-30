# RIFF.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 1.71
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

The RIFF.pm module handles Resource Interchange File Format (RIFF) files, which serve as containers for various media types including AVI videos, WAV audio files, WEBP images, and several lossless audio formats (LA, OFR, PAC, WV). This is a container format processing module that implements hierarchical chunk parsing with support for both reading and writing operations (limited writing support for WEBP).

## Module Structure

### Tag Tables (19 total)

**Core Processing Tables:**

- `Main` - Primary RIFF chunk processor with manufacturer-specific JUNK handling
- `ProcessChunks` - Generic chunk processing function used across multiple tables
- `audioEncoding` - Comprehensive audio codec lookup table (240+ entries)

**Audio-Specific Tables:**

- `AudioFormat` - WAV format chunk (fmt) processing
- `BroadcastExt` - EBU broadcast wave extension (bext)
- `DS64` - 64-bit extensions for RF64 format
- `Sampler` - Instrument sampling parameters
- `Instrument` - Musical instrument metadata

**Video-Specific Tables:**

- `Hdrl` - AVI header list processing
- `AVIHeader` - Main AVI header structure
- `Stream` - AVI stream headers and data
- `StreamHeader` - Individual stream header details
- `StreamData` - Stream-specific data processing
- `OpenDML` - OpenDML extensions for AVI

**WebP-Specific Tables:**

- `VP8` - Simple WebP (lossy) format
- `VP8L` - Lossless WebP format
- `VP8X` - Extended WebP with feature flags
- `ANIM` - WebP animation parameters
- `ANMF` - Individual animation frame data
- `ALPH` - Alpha channel information

**Metadata Tables:**

- `Info` - RIFF INFO chunk (177 tags including 3rd party extensions)
- `Exif` - EXIF 2.3 specification tags for WAV
- `Tdat` - Adobe CS3 Bridge metadata
- `UserText` - Momento M6 dashcam streaming data

**Utility Tables:**

- `Composite` - Duration calculations using frame rates
- `Junk` - Fallback for unrecognized content

### Key Data Structures

**Format Recognition Hashes:**

- `%riffType` - Maps 4-byte identifiers to file types (WAVE→WAV, AVI →AVI, etc.)
- `%riffMimeType` - MIME type associations for each RIFF variant
- `%isImageData` - Chunks containing image data for hash digest calculation

**Character Encoding Support:**

- `%code2charset` - Windows code page to character set mapping (18 encodings)

**Specialized Processing Arrays:**

- `%audioEncoding` - TwoCC audio format codes (240+ codec definitions)

## Processing Architecture

### 1. Main Processing Flow

**Entry Point:** `ProcessRIFF($$)` - Main file processor

- Validates RIFF signature (RIFF/RF64) or specialty formats (LA0x, OFR, LPAC, wvpk)
- Handles RF64 extensions for large files (>4GB)
- Sets file type and MIME type based on 4-byte identifier
- Processes chunks sequentially with padding alignment
- Supports concatenated AVI files as sub-documents

**Chunk Processing:** `ProcessChunks($$$)` - Generic chunk processor

- Handles LIST chunks with type-specific sub-processing
- Implements character set decoding via CharsetRIFF option
- Creates unknown tag definitions dynamically
- Manages base address shifting for nested structures

### 2. Special Processing Functions

**Media-Specific Processors:**

- `ProcessStreamData($$$)` - AVI stream content processor
- `ProcessSGLT($$$)` - BikeBro accelerometer data extraction (20-byte records)
- `ProcessSLLT($$$)` - BikeBro GPS data extraction (30-byte records)
- `ProcessLucas($$$)` - Lucas LK-7900 Ace GPS streaming data

**Utility Functions:**

- `CalcDuration($@)` - Multi-source duration calculation for concatenated files
- `ConvertTimecode($)` - SMPTE timecode formatting
- `ConvertRIFFDate($)` - Multi-format date parsing (handles Casio, Konica variations)
- `MakeTagInfo($$)` - Dynamic unknown tag creation with hex formatting

## Key Features

### 1. Multi-Format Container Support

- **AVI Video:** Full header parsing, stream information, OpenDML extensions
- **WAV Audio:** Format chunks, broadcast extensions, EXIF 2.3 compliance
- **WEBP Images:** VP8/VP8L/VP8X variants, animation support, alpha channels
- **Specialty Audio:** LA (lossless), OFR (OptimFROG), PAC (LPAC), WV (WavPack)

### 2. Manufacturer-Specific Extensions

**Camera Manufacturer JUNK Processing:**

- **Olympus:** `OLYMDigital Camera` signature detection
- **Casio:** Standard EXIF format in JUNK chunks (BigEndian, offset 10)
- **Ricoh:** Sub-chunk processing with ucmt signature
- **Pentax:** Multiple variants (IIII signature, PENTDigital Camera)

**Manufacturer-Specific LIST Processing:**

- **Nikon:** LIST_ncdt → Nikon::AVI tag table
- **Pentax:** LIST_hydt, LIST_pntx → Pentax::AVI tag table

### 3. Embedded Data Extraction

**Streaming GPS Data:**

- BikeBro SGLT: Accelerometer data (frame number + 3-axis acceleration)
- BikeBro SLLT: GPS coordinates, altitude, speed, timestamp
- Lucas LK-7900: NMEA-like GPS parsing (0GDA/0GPS records)

**Timed Metadata:**

- Requires ExtractEmbedded option for full extraction
- Sub-document processing with DOC_NUM tracking
- Stream-specific text processing (##tx chunks by codec)

### 4. WebP Advanced Features

- **Extended Format Detection:** VP8X chunk presence triggers "Extended WEBP" type
- **Animation Support:** Loop count, frame duration summation, background color
- **Feature Flag Parsing:** ICC, Alpha, EXIF, XMP, Animation capability detection
- **Lossless Detection:** Automatic FileType modification for VP8L chunks

## Special Patterns

### 1. Concatenated File Handling

- AVI files may be concatenated RIFF chunks
- Duration calculation spans multiple sub-documents
- Sub-document metadata extraction with Doc# grouping
- Complex frame rate reconciliation between header and stream data

### 2. Large File Support

- RF64 format handling for files >4GB
- LargeFileSupport option controls processing behavior
- Data size extensions via ds64 chunk
- Chunk length validation and corruption detection

### 3. Dynamic Tag Creation

- Unknown tags converted to hex format if largely binary
- Tag name sanitization for non-printable characters
- Automatic tag table population during processing

### 4. Character Set Handling

- Windows code page detection and conversion
- CharsetRIFF option overrides
- CodePage inheritance from other modules
- Multi-byte character support (UTF8, Cyrillic, Arabic, etc.)

## Usage Notes

### Reading Operations

- Supports all major RIFF variants (AVI, WAV, WEBP, specialty audio)
- ExtractEmbedded option required for streaming GPS/accelerometer data
- FastScan option stops at data/idx1/LIST_movi for performance
- Validate option enables strict chunk size checking

### Writing Operations

- **Limited Support:** Currently only WEBP images support writing
- **WriteRIFF Function:** Declared but not implemented in this module
- **Supported Metadata:** EXIF, XMP, ICC_Profile for WEBP
- **AUTOLOAD System:** Writer routines loaded dynamically when needed

### Performance Considerations

- NoBuffer mode available for FastScan operations
- Large file validation can be disabled for performance
- Chunk-based processing minimizes memory usage
- Hash digest calculation for image data chunks

## Debugging Tips

### Common Issues

1. **Corrupt LIST_movi:** Module attempts recovery by seeking to calculated end position
2. **Concatenated Files:** Duration may be incorrect if frame rate data is unreliable
3. **Character Encoding:** Check CharsetRIFF option if text appears garbled
4. **Large Files:** Enable LargeFileSupport for files >2GB, use option 2 for detailed warnings

### Diagnostic Features

- Verbose output shows chunk structure and processing flow
- Unknown chunk handling creates discoverable tag names
- Warning system for corrupt chunks and size mismatches
- Sub-document processing visible in verbose mode

### Validation Features

- Chunk size verification against file boundaries
- Padding validation for odd-sized chunks
- RIFF header size consistency checking
- Stream data integrity validation
