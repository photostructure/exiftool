# MPEG.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 1.17
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

The MPEG.pm module extracts comprehensive metadata from MPEG-1 and MPEG-2 audio and video streams. It implements sophisticated frame header parsing with version-specific processing, supports variable bitrate (VBR) detection, and includes advanced encoder identification. The module handles both pure audio (MP3) files and combined audio/video (MPEG) streams with complex validation and error recovery mechanisms.

## Module Structure

### Tag Tables (5 total)

1. **Audio** - MPEG audio frame header metadata (12 fields)

   - Version-specific conditional processing (MPEG 1, 2, 2.5)
   - Layer-specific bitrate tables (Layer 1, 2, 3)
   - Sample rate, channel mode, and encoding parameters
   - Complex conditional bitrate mappings

2. **Video** - MPEG video sequence header data (5 fields)

   - Image dimensions (width/height)
   - Aspect ratio with PAL/NTSC format identification
   - Frame rate detection (23.976-60 fps)
   - Video bitrate calculation

3. **Xing** - Variable bitrate frame information (6 fields)

   - VBR frames, bytes, and scale values
   - Encoder identification and version
   - Lame-specific quality metrics
   - VBR vs. CBR detection

4. **Lame** - Lame encoder metadata (4 fields)

   - Encoding method (CBR, ABR, VBR variants)
   - Low-pass filter settings
   - Bitrate and stereo mode configuration
   - Quality parameters

5. **Composite** - Calculated fields (2 fields)
   - Duration estimation from bitrate and file size
   - Variable bitrate audio calculation from VBR parameters

### Key Data Structures

- **Version/Layer State Variables**: $$et{MPEG_Vers}, $$et{MPEG_Layer}, $$et{MPEG_Mode}
- **Conditional Bitrate Tables**: Version and layer-specific bitrate mappings
- **Frame Sync Detection**: Pattern-based \xff frame header identification
- **VBR Flag Processing**: Multi-flag VBR information extraction
- **Encoder Detection**: Pattern matching for various MP3 encoders

## Processing Architecture

### 1. Main Processing Flow

MPEG processing follows a multi-stage approach:

1. **Stream Identification**: Detects MPEG markers (\0\0\x01) in data stream
2. **Frame Sync Detection**: Locates valid audio frame headers (\xff patterns)
3. **Header Validation**: Extensive validation to prevent false positives
4. **Version/Layer Extraction**: Determines MPEG version and audio layer
5. **Conditional Processing**: Routes to appropriate bitrate/format tables
6. **VBR Analysis**: Searches for Xing/Info frames in audio streams
7. **Encoder Identification**: Attempts to identify encoding software

### 2. Special Processing Functions

- **ProcessFrameHeader**: Generic bit-range extraction from 32-bit frame headers
- **ParseMPEGAudio**: Complete audio frame processing with VBR support
- **ProcessMPEGVideo**: Video sequence header analysis
- **ParseMPEGAudioVideo**: Combined stream processing for audio/video files
- **ProcessMPEG**: Main entry point with initial stream validation

## Key Features

### 1. Multi-Version Audio Support

- **MPEG-1 Audio**: Full Layer 1, 2, 3 support with dedicated bitrate tables
- **MPEG-2 Audio**: Reduced bitrate ranges for efficient encoding
- **MPEG-2.5 Audio**: Extended low-bitrate support for very low quality
- **Conditional Processing**: Version-specific tag interpretation

### 2. Variable Bitrate (VBR) Processing

- **Xing Frame Detection**: Identifies VBR metadata frames
- **Info Frame Handling**: Distinguishes between VBR and CBR Info frames
- **TOC Table Support**: Table of contents for VBR seeking (parsed but not extracted)
- **Quality Metrics**: VBR scale and quality parameter extraction

### 3. Advanced Encoder Detection

- **Lame Identification**: Version-specific Lame encoder metadata
- **Alternative Encoders**: RCA mp3PRO, Thomson mp3PRO, Gogo detection
- **Version Comparison**: String-based encoder version analysis
- **Quality Extraction**: Encoder-specific quality parameter mapping

### 4. Sophisticated Frame Validation

- **Header Sync Validation**: Multi-stage \xff frame sync verification
- **Reserved Value Checking**: Validates against reserved bit patterns
- **Consistency Verification**: Cross-checks version, layer, and bitrate combinations
- **Error Recovery**: Continues searching after invalid headers

### 5. Video Stream Analysis

- **Sequence Header Parsing**: Complete MPEG video sequence analysis
- **Aspect Ratio Mapping**: Predefined PAL/NTSC format identification
- **Frame Rate Detection**: Standard frame rate recognition (23.976-60 fps)
- **Bitrate Calculation**: Video bitrate with variable bitrate detection

### 6. Composite Calculations

- **Duration Estimation**: File size and bitrate-based duration calculation
- **VBR Bitrate Computation**: Accurate bitrate for variable bitrate audio
- **Approximation Handling**: Clear indication of approximate values

## Special Patterns

### 1. Version/Layer Conditional Processing

```perl
Condition => '$self->{MPEG_Vers} == 3 and $self->{MPEG_Layer} == 3',
Condition => '$self->{MPEG_Vers} != 3 and $self->{MPEG_Layer} > 1',
```

### 2. Bit-Range Frame Processing

```perl
'Bit11-12' => { Name => 'MPEGAudioVersion' },
'Bit16-19' => [{ # Multiple conditional bitrate tables }],
```

### 3. VBR Flag-Based Processing

```perl
if ($flags & 0x01) { # VBR Frames present }
if ($flags & 0x02) { # VBR Bytes present }
```

### 4. Frame Sync Pattern Matching

```perl
$$buffPt =~ m{(\xff.{3})}sg;  # Audio frame sync
$$buffPt =~ /\0\0\x01(\xb3|\xc0)/g;  # Video/audio stream markers
```

### 5. State Variable Management

```perl
RawConv => '$self->{MPEG_Vers} = $val',  # Store for conditional processing
RawConv => '$self->{MPEG_Layer} = $val',
```

## Usage Notes

### 1. Frame Validation Requirements

- Audio frame headers must pass extensive validation
- Reserved bit patterns cause frame rejection
- Version/layer combinations must be valid
- MP3 files require Layer 3 encoding

### 2. VBR Processing Limitations

- VBR metadata only available in Xing/Info frames
- TOC (Table of Contents) data is parsed but not extracted
- Some encoders may not include complete VBR information
- Info frames indicate CBR despite similar structure to Xing

### 3. Version Dependencies

- Bitrate tables vary significantly between MPEG versions
- Sample rates are version-dependent (44.1/22.05/11.025 kHz families)
- Layer 3 (MP3) has different characteristics from Layer 1/2
- MPEG-2.5 is a non-standard extension

### 4. Encoder-Specific Features

- Lame metadata only available in Lame 3.90 or later
- Quality metrics interpretation varies by encoder
- Some encoders use proprietary metadata formats
- Version string comparison requires careful handling

## Debugging Tips

### 1. Frame Sync Issues

- Use verbose mode to see frame sync detection attempts
- Check for \xff patterns in hex dumps of problematic files
- Verify frame header validation logic with known good files
- Monitor pos() movement in frame sync searches

### 2. Version/Layer Detection Problems

- Verify MPEG_Vers and MPEG_Layer state variables
- Check conditional bitrate table selection
- Validate version ID and layer description bit patterns
- Monitor for reserved value detection

### 3. VBR Processing Issues

- Check for Xing/Info frame presence in verbose mode
- Verify VBR flag interpretation (0x01, 0x02, 0x04, 0x08)
- Monitor frame offset calculations for VBR data
- Validate VBR vs. Info frame distinction

### 4. Bitrate Calculation Problems

- Verify correct bitrate table selection for version/layer
- Check for "free" bitrate (0000) handling
- Monitor composite bitrate calculations for VBR files
- Validate bitrate conversion from table values

### 5. Video Stream Analysis

- Check for proper MPEG video sequence header detection
- Verify aspect ratio and frame rate table lookups
- Monitor video bitrate calculation (400 bps units)
- Validate image dimension extraction

### 6. File Type Detection

- Verify proper MP3 vs. MPEG file type identification
- Check extension-based processing (.MP3 files)
- Monitor SetFileType() calls and file type determination
- Validate stream marker detection (\0\0\x01)

### 7. Encoder Identification

- Check encoder string pattern matching
- Verify Lame version string comparison logic
- Monitor alternative encoder detection patterns
- Validate encoder metadata extraction

### 8. Performance Considerations

- Frame sync detection can be computationally intensive
- Large buffer searches may impact performance
- VBR processing adds overhead to audio analysis
- Consider limiting search depth for performance optimization
