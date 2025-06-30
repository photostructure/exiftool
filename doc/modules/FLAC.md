# FLAC.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 1.09
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

The FLAC.pm module handles Free Lossless Audio Codec (FLAC) audio files, extracting comprehensive metadata including stream properties, embedded images, and comments. It features sophisticated bit-level data processing and seamless integration with ID3 tags and Vorbis comments, making it a complete solution for FLAC metadata extraction.

## Module Structure

### Tag Tables (4 total)

1. **Main** - FLAC metadata block types (8 block types)
   - StreamInfo (0): Core audio stream properties
   - Padding (1): Unused space blocks
   - Application (2): Vendor-specific data with RIFF support
   - SeekTable (3): Audio seeking information
   - VorbisComment (4): Text metadata using Vorbis format
   - CueSheet (5): CD cue sheet information
   - Picture (6): Embedded image data
   - Reserved blocks (7-126) and Invalid (127)

2. **StreamInfo** - Bit-level audio stream properties (8 fields)
   - Block sizes, frame sizes, sample rate, channels
   - Bits per sample, total samples, MD5 signature
   - Uses custom bit-range processing (ProcessBitStream)

3. **Picture** - Embedded image metadata (9 fields)
   - Picture type classification (21 predefined types)
   - MIME type and description with UTF-8 support
   - Dimensions, color depth, and raw image data
   - Variable-length string format (var_pstr32)

4. **Composite** - Calculated fields (1 field)
   - Duration calculation from sample rate and total samples

### Key Data Structures

- **Bit Stream Processing**: Custom bit-range extraction supporting arbitrary bit widths
- **Variable Pascal Strings**: var_pstr32 format for length-prefixed strings
- **Metadata Block Headers**: 4-byte headers with last-block flag and size
- **Picture Type Mapping**: Standardized picture type classifications
- **RIFF Integration**: Special handling for RIFF-formatted application blocks

## Processing Architecture

### 1. Main Processing Flow

The module follows FLAC specification exactly:

1. **ID3 Pre-processing**: Checks for leading/trailing ID3 tags before FLAC processing
2. **Signature Verification**: Validates 'fLaC' magic signature
3. **Sequential Block Processing**: Reads metadata blocks until last-block flag encountered
4. **Block Type Dispatch**: Routes blocks to appropriate tag tables
5. **Error Recovery**: Handles format corruption gracefully

### 2. Special Processing Functions

- **ProcessBitStream**: Advanced bit-level data extraction with arbitrary bit ranges
- **ProcessFLAC**: Main entry point with ID3 integration and sequential block processing
- **RIFF Application Processing**: Special subdirectory processing for RIFF-formatted blocks
- **UTF-8 Decoding**: Automatic character encoding for picture descriptions
- **Composite Tag Calculation**: Duration computation from audio parameters

## Key Features

### 1. Bit-Level Stream Processing
- **Arbitrary Bit Ranges**: Extract data from any bit position (e.g., Bit080-099 for sample rate)
- **Byte Order Support**: Handles both big-endian ('MM') and little-endian ('II') bit streams
- **Wide Integer Support**: Can process values larger than 64 bits
- **Mask Visualization**: Provides detailed bit mask information in verbose mode

### 2. Picture Block Processing
- **21 Picture Types**: Complete classification including icons, covers, artist photos
- **Variable String Format**: Efficient length-prefixed string storage
- **UTF-8 Support**: Proper character encoding for international descriptions
- **Binary Data Handling**: Direct extraction of embedded image data

### 3. Application Block Integration
- **RIFF Detection**: Special processing for "riff" application blocks (excluding headers)
- **Subdirectory Processing**: Routes RIFF blocks to RIFF tag table with little-endian order
- **Unknown Block Handling**: Graceful handling of unrecognized application blocks

### 4. Composite Calculations
- **Duration Computation**: Accurate duration from sample rate and total samples
- **Format Conversion**: Human-readable duration display (ConvertDuration)

### 5. ID3 Integration
- **Pre-processing Check**: Automatically handles ID3v1/ID3v2 tags before FLAC processing
- **DoneID3 Flag**: Prevents duplicate ID3 processing
- **Seamless Integration**: Returns control to ID3 processor when ID3 detected

## Special Patterns

### 1. Bit-Range Tag Naming
```perl
'Bit000-015' => 'BlockSizeMin',      # 16-bit value
'Bit080-099' => 'SampleRate',        # 20-bit value  
'Bit144-271' => 'MD5Signature',      # 128-bit value
```

### 2. Conditional Application Block Processing
```perl
Condition => '$$valPt =~ /^riff(?!RIFF)/',  # Special RIFF detection
```

### 3. Variable Format Strings
```perl
Format => 'var_pstr32',  # Length-prefixed string with 32-bit length
```

### 4. Binary Data with Length References
```perl
Format => 'undef[$val{7}]',  # Use PictureLength for image data size
```

### 5. Value Conversion Chains
```perl
ValueConv => '$val + 1',           # Channels: stored as n-1
ValueConv => 'unpack("H*",$val)',  # MD5: binary to hex
```

## Usage Notes

### 1. Processing Order
- Always processes ID3 tags first if present
- Metadata blocks must be processed sequentially
- Last-metadata-block flag terminates processing

### 2. Bit Stream Limitations
- Bit positions must align with FLAC specification
- Custom bit ranges require careful mask calculation
- Little-endian support exists but FLAC always uses big-endian

### 3. Picture Block Handling
- Picture descriptions are automatically UTF-8 decoded
- Picture data is extracted as binary blob
- Picture types follow ID3v2 APIC specification

### 4. Application Block Processing
- Only "riff" application blocks receive special processing
- RIFF blocks exclude the initial RIFF header
- Unknown application blocks are stored as binary data

## Debugging Tips

### 1. Bit Stream Issues
- Use verbose mode to see bit masks and extracted values
- Verify bit positions match FLAC specification exactly
- Check byte order setting (FLAC requires 'MM')

### 2. Block Processing Problems
- Verify metadata block headers for correct size calculation
- Check last-metadata-block flag handling
- Monitor for format corruption warnings

### 3. Picture Block Debugging
- Validate var_pstr32 string length calculations
- Check UTF-8 decoding for international characters
- Verify picture data length matches header specification

### 4. Application Block Analysis
- Use `-htmlDump` to examine raw application block data
- Check RIFF detection regex for edge cases
- Monitor subdirectory processing for RIFF blocks

### 5. Format Validation
- Check for proper 'fLaC' signature
- Validate metadata block sequence
- Monitor for premature EOF conditions

### 6. ID3 Integration Issues
- Verify ID3 processing completes before FLAC
- Check DoneID3 flag state
- Monitor for conflicting metadata sources