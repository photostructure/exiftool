# Ogg.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 1.04
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

The Ogg.pm module extracts metadata from Ogg bitstream container files, serving as a routing layer that identifies and processes multiple codec types within the Ogg container format. It implements complete Ogg page parsing with stream multiplexing support, handling Vorbis audio, Theora video, Opus audio, FLAC audio, and ID3 tags while providing performance optimizations through limited packet scanning.

## Module Structure

### Tag Tables (1 total)

1. **Main** - Documentation table for supported formats (5 entries)
   - Vorbis: Audio codec routing to Vorbis module
   - Theora: Video codec routing to Theora module
   - Opus: Modern audio codec routing to Opus module
   - FLAC: Lossless audio routing to FLAC module
   - ID3: Legacy tag format routing to ID3 module

### Key Data Structures

- **Stream Tracking**: %streamPage hash for per-stream page numbering
- **Packet Assembly**: %val hash for packet reconstruction across pages
- **Page Headers**: 28-byte Ogg page header structure
- **Segment Tables**: Variable-length segment size arrays
- **Stream States**: Flag-based processing state management

## Processing Architecture

### 1. Main Processing Flow

Ogg processing follows the official specification:

1. **ID3 Pre-processing**: Checks for leading/trailing ID3 tags
2. **Page Header Parsing**: Extracts 28-byte Ogg page headers
3. **Stream Identification**: Tracks multiple streams by serial number
4. **Segment Processing**: Reads variable-length segment tables
5. **Packet Reconstruction**: Assembles packets across continuation pages
6. **Codec Routing**: Dispatches packets to appropriate codec modules
7. **Performance Limiting**: Stops after MAX_PACKETS (2) per stream

### 2. Special Processing Functions

- **ProcessOGG**: Main Ogg container parser with page-level processing
- **ProcessPacket**: Codec identification and routing dispatcher
- **Page Structure Parsing**: Complete Ogg page header interpretation
- **Stream Multiplexing**: Multi-stream management with continuation handling
- **FLAC Integration**: Special FLAC-in-Ogg format processing

## Key Features

### 1. Ogg Container Implementation

- **Page Structure Parsing**: Complete 28-byte header processing
- **Magic Number Validation**: 'OggS' signature verification
- **Flag Interpretation**: Beginning/continuation/end-of-stream flags
- **Segment Tables**: Variable-length data organization
- **CRC Validation**: (Header parsing only - CRC not verified)

### 2. Multi-Stream Support

- **Stream Multiplexing**: Simultaneous processing of multiple logical streams
- **Serial Number Tracking**: Per-stream identification and management
- **Page Sequencing**: Sequential page number validation per stream
- **Stream Synchronization**: Lost page detection and warning

### 3. Codec Identification and Routing

- **Vorbis Detection**: '\x01vorbis', '\x03vorbis', '\x05vorbis' headers
- **Theora Detection**: '\x80theora', '\x81theora', '\x82theora' headers
- **Opus Detection**: 'OpusHead', 'OpusTags' headers
- **FLAC Detection**: '\x7fFLAC' header with metadata block count
- **Dynamic Routing**: Automatic dispatch to appropriate codec modules

### 4. Packet Reconstruction

- **Continuation Handling**: Packets spanning multiple pages
- **Buffer Management**: Efficient string concatenation for large packets
- **Flag Processing**: Beginning/continuation/end-of-stream logic
- **Segment Assembly**: Variable-length segment reconstruction

### 5. Performance Optimization

- **Limited Scanning**: MAX_PACKETS (2) per stream for efficiency
- **Early Termination**: Stops when sufficient metadata extracted
- **Stream Filtering**: Only processes recognized codec types
- **Memory Management**: Cleans up processed packets immediately

### 6. File Type Management

- **Dynamic Override**: Changes file type based on content
- **OGV Detection**: Theora video triggers OGV file type
- **OPUS Detection**: Opus audio triggers OPUS file type
- **Format Precedence**: Video content takes precedence over audio

## Special Patterns

### 1. Multi-Stream Page Processing

```perl
$stream = Get32u(\$buff, 14);   # Extract stream serial number
$streamPage{$stream} = ++$page; # Track per-stream page numbers
```

### 2. Codec Header Detection

```perl
if ($$dataPt =~ /^(.)(vorbis|theora)/s or $$dataPt =~ /^(OpusHead|OpusTags)/) {
    # Route to appropriate codec module
}
```

### 3. Continuation Page Handling

```perl
if (defined $val{$stream}) {
    $val{$stream} .= $buff;     # Append continuation data
} elsif (not $flag & 0x01) {   # Start new packet if not continuation
```

### 4. FLAC Special Processing

```perl
if ($buff =~ /^\x7fFLAC..(..)/s) {
    $numFlac = unpack('n',$1);  # Extract metadata block count
    $val{$stream} = substr($buff, 9);  # Skip Ogg FLAC header
}
```

### 5. Performance Limiting

```perl
if ($packets > $MAX_PACKETS * $streams or not defined $raf) {
    last unless %val;   # Stop processing when limit reached
}
```

## Usage Notes

### 1. Container vs. Codec Processing

- Ogg.pm handles container structure only
- Actual metadata extraction performed by codec modules
- Codec modules: Vorbis.pm, Theora.pm, Opus.pm, FLAC.pm
- ID3 processing handled by ID3.pm module

### 2. Multi-Stream Limitations

- Limited to MAX_PACKETS (2) per stream for performance
- May not extract all metadata from complex multi-stream files
- Page sequence validation provides corruption detection
- Lost synchronization triggers warnings

### 3. Codec Dependencies

- Requires appropriate codec modules for complete processing
- Unknown codecs are silently ignored
- FLAC processing requires FLAC.pm module
- ID3 processing requires ID3.pm module

### 4. Performance Considerations

- Early termination optimizes processing of large files
- Packet limit prevents excessive scanning
- Stream filtering reduces unnecessary processing
- Memory cleanup prevents accumulation of large buffers

## Debugging Tips

### 1. Container Structure Issues

- Use verbose mode to see page headers and flags
- Check magic number validation ('OggS' signature)
- Monitor stream serial number tracking
- Verify page sequence numbering

### 2. Multi-Stream Problems

- Check stream count and serial number assignment
- Monitor page flag interpretation (0x01, 0x02, 0x04)
- Verify continuation page handling logic
- Watch for lost synchronization warnings

### 3. Codec Detection Issues

- Verify codec header pattern matching
- Check packet reconstruction across pages
- Monitor codec module routing
- Validate subdirectory processing calls

### 4. Performance Analysis

- Monitor packet counting and stream limits
- Check early termination conditions
- Verify buffer cleanup and memory usage
- Watch for excessive continuation pages

### 5. FLAC-in-Ogg Debugging

- Check FLAC header detection (\x7fFLAC)
- Verify metadata block count extraction
- Monitor FLAC module integration
- Validate header stripping logic

### 6. File Type Detection

- Check file type override logic
- Verify Theora → OGV conversion
- Monitor Opus → OPUS conversion
- Validate codec precedence handling

### 7. Page Structure Validation

- Use -htmlDump to examine page headers
- Check segment table processing
- Verify data length calculations
- Monitor page flag combinations

### 8. Stream Synchronization

- Check page sequence validation
- Monitor missing page detection
- Verify stream state management
- Watch for corruption warnings
