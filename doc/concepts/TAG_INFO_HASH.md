# ExifTool Tag Information Hash: Complete Tag Definition System

**ExifTool Version:** 13.26  
**Document Version:** 1.0  
**Last Updated:** 2025-07-04

## Overview

ExifTool's tag information hash system is the comprehensive framework that defines every metadata tag's behavior, validation, conversion, and processing characteristics. This sophisticated system encompasses over 50 different properties that control every aspect of tag handling, from basic naming and grouping to complex conversion expressions and writing constraints.

**Core Concept**: Every tag in ExifTool is defined by a tag information hash containing properties that control reading, conversion, validation, writing, and documentation behavior. This system enables ExifTool to handle the enormous complexity and diversity of metadata formats while maintaining consistency and extensibility.

## Tag Information Hash Structure

### Basic Definition Forms

Tag information can be specified in three forms (`README:271-278`):

**1. Simple Scalar** - Tag name only:

```perl
0x010f => 'Make',
```

**2. Hash Reference** - Complete tag definition:

```perl
0x010f => {
    Name => 'Make',
    Writable => 'string',
    Groups => { 2 => 'Camera' },
    PrintConv => '$val',
},
```

**3. Array Reference** - Conditional tag definitions:

```perl
0x0001 => [
    {
        Name => 'TagVersion1',
        Condition => '$format eq "int16u"',
        ValueConv => '$val / 100',
    },
    {
        Name => 'TagVersion2',
        Condition => '$format eq "int32u"',
        ValueConv => '$val / 1000',
    },
],
```

## Core Tag Properties

### Essential Properties

**Name**: Tag identifier (`README:280-290`):

```perl
Name => 'ExposureTime',
```

- Must start with uppercase letter
- May contain only `[A-Za-z0-9_-]` characters
- Auto-generated from TagID if not specified
- Need not be unique (duplicates handled by priority system)

**Description**: Human-readable tag description (`README:292-294`):

```perl
Description => 'Exposure Time',
```

- Used in verbose output and documentation
- Defaults to Name with spaces inserted between words
- Plain text only

**Groups**: Group classification override (`README:298-299`):

```perl
Groups => { 0 => 'EXIF', 1 => 'ExifIFD', 2 => 'Camera' },
```

- Overrides table-level GROUPS definition
- See GROUP_SYSTEM.md for complete details

### Format and Data Properties

**Format**: Binary data format specification (`README:301-323`):

```perl
Format => 'int16u',              # Basic format
Format => 'string[4]',           # Fixed-length string
Format => 'string[$val{3}]',     # Dynamic size expression
Format => 'var_int16u[10]',      # Variable-length with offset adjustment
```

**Supported Formats**:

- **Integer**: `int8u/s`, `int16u/s`, `int32u/s`, `int64u/s`
- **Rational**: `rational32u/s`, `rational64u/s`
- **Floating Point**: `float`, `double`
- **String**: `string`, `undef`
- **Special**: `var_*` (variable-length), `*Rev` (reversed byte order)

**Count**: Value count specification (`README:325-332`):

```perl
Count => 4,        # Fixed count
Count => -1,       # Variable count
Count => '$val{1}' # Dynamic count expression
```

**FixCount**: Correct count from IFD offsets (`README:334-336`):

```perl
FixCount => 1,  # Fix incorrect Kodak tag counts
```

## Conversion System

### Value Conversion Chain

ExifTool applies conversions in this order:
**Raw Value** → **RawConv** → **ValueConv** → **PrintConv** → **Display String**

### RawConv: Early Raw Processing (`README:579-608`)

**Purpose**: Convert raw value during extraction (while file is open)
**Usage**: Only when necessary for validation or warning generation

```perl
RawConv => '$val ? $val : undef',                    # Validate non-zero
RawConv => '$$self{MAKER_NOTE_ADDR} = $val',        # Store in ExifTool object
RawConv => \&ConvertRawFunction,                     # Code reference
```

**Available Variables**: `$val`, `$self`, `$tag`, `$tagInfo`, `$priority`, `@grps`
**Return Values**: Scalar, scalar reference (binary), hash reference, array reference, or `undef`

### ValueConv: Logical Value Conversion (`README:610-663`)

**Purpose**: Convert raw value to logical, programmatically useful form

**Hash Lookup**:

```perl
ValueConv => {
    0 => 'Auto',
    1 => 'Manual',
    2 => 'Program',
},
```

**Expression Evaluation**:

```perl
ValueConv => '$val / 32',                    # Mathematical conversion
ValueConv => 'exp($val/32*log(2))*100',     # Complex expression
ValueConv => '$val =~ s/\0+$//; $val',      # String processing
```

**Code Reference**:

```perl
ValueConv => \&ConvertExposureTime,
```

**Array Processing**:

```perl
ValueConv => ['Off', 'On', 'Auto'],         # List conversion
```

**Special Hash Keys**:

- **BITMASK**: Reference to hash for bit decoding
- **OTHER**: Subroutine for unknown values
- **Notes**: Documentation for SeparateTable flag

### PrintConv: Human-Readable Formatting (`README:665-677`)

**Purpose**: Convert logical value to human-readable display string

```perl
PrintConv => {
    0 => 'No Flash',
    1 => 'Fired',
    # ... flash mode combinations
},

PrintConv => 'sprintf("%.1f mm",$val)',      # Formatted output
PrintConv => '"$val seconds"',               # String interpolation
```

**List Processing**: For array values, converted values joined by `'; '`

### Inverse Conversions

**ValueConvInv**: Inverse of ValueConv (`README:690-703`):

```perl
ValueConvInv => '$val * 32',
ValueConvInv => \&InverseConvertFunction,
```

**PrintConvInv**: Inverse of PrintConv (`README:705-708`):

```perl
PrintConvInv => '$val =~ s/ seconds$//; $val',
```

**RawConvInv**: Inverse of RawConv (`README:679-688`):

```perl
RawConvInv => '$val',  # Rare usage for dynamic raw values
```

## Flag System

ExifTool uses an extensive flag system to control tag behavior (`README:338-578`). Flags can be specified as:

```perl
Flags => 'Binary',                    # Single flag
Flags => ['Binary', 'Protected'],     # Array of flags
Flags => { Binary => 1, Priority => 0 }, # Hash with values
```

### Data Handling Flags

**Binary**: Mark as binary data (`README:355-359`):

```perl
Flags => 'Binary',  # Sets ValueConv to '\$val'
```

**List**: Enable list-type behavior (`README:435-444`):

```perl
Flags => 'List',     # Accumulate duplicate entries
Flags => 'Bag',      # XMP Bag type
Flags => 'Seq',      # XMP Sequence type
Flags => 'Alt',      # XMP Alternative type
```

**AutoSplit**: Automatic value splitting (`README:346-348`):

```perl
Flags => { AutoSplit => ',?\\s+' },  # Split pattern
```

### Protection and Priority Flags

**Protected**: Write protection levels (`README:509-513`):

```perl
Protected => 1,  # Bit 0x01: Unsafe tag
Protected => 2,  # Bit 0x02: Protected from direct user modification
Protected => 3,  # Both protection levels
```

**Priority**: Tag precedence control (`README:498-507`):

```perl
Priority => 0,   # Won't override previous tags
Priority => 1,   # Default priority
Priority => 5,   # High priority (overrides lower)
```

**Permanent**: Prevent tag deletion (`README:473-476`):

```perl
Flags => 'Permanent',  # Cannot be deleted (all MakerNotes default)
```

### Offset and Reference Flags

**IsOffset**: Offset value adjustment (`README:423-430`):

```perl
IsOffset => 1,          # Adjust to absolute file offset
IsOffset => 2,          # Use parent directory base
IsOffset => 3,          # Already absolute offsets
IsOffset => '$val > 0', # Conditional expression
```

**OffsetPair**: Offset/length pair specification (`README:469-471`):

```perl
OffsetPair => 0x0202,  # Paired tag ID
DataTag => 'PreviewImage',  # Associated data tag
```

**EntryBased**: Entry-relative offsets (`README:392-395`):

```perl
Flags => 'EntryBased',  # Offset relative to IFD entry
```

### Processing Control Flags

**Unknown**: Unknown tag handling (`README:562-564`):

```perl
Unknown => 1,  # Extract when Unknown option set
Unknown => 2,  # Extract when Unknown >= 2 (binary tables)
```

**Hidden**: Hide from output (`README:412-415`):

```perl
Flags => 'Hidden',  # Suppress from documentation and verbose output
```

**Avoid**: Minimize creation (`README:350-353`):

```perl
Flags => 'Avoid',  # Avoid creating when writing if alternatives exist
```

### Specialized Processing Flags

**SubIFD**: Subdirectory marker (`README:555-560`):

```perl
SubIFD => 1,  # Standard subdirectory
SubIFD => 2,  # High-priority subdirectory (ExifIFD, GPS)
```

**MakerNotes**: Maker note identification (`README:450`):

```perl
Flags => 'MakerNotes',  # Mark as maker note data
```

**DataMember**: ExifTool data storage (`README:372-378`):

```perl
DataMember => 'CompressionType',  # Store in $$self{CompressionType}
```

## Writing System Properties

### Basic Writing Control

**Writable**: Write capability specification (`README:807-827`):

```perl
Writable => 'string',      # Format for writing
Writable => 1,             # Use existing format
Writable => 0,             # Not writable
Writable => 2,             # Show as writable in docs only
```

**WriteGroup**: Target write location (`README:877-886`):

```perl
WriteGroup => 'ExifIFD',   # Specific IFD
WriteGroup => 'All',       # All applicable locations
```

**WriteAlso**: Cascade writing (`README:829-846`):

```perl
WriteAlso => {
    'EXIF:ISO' => '$val',
    'XMP:ISO' => '$val',
},
```

### Writing Validation

**WriteCheck**: Value validation (`README:848-855`):

```perl
WriteCheck => '$val > 0 or "Value must be positive"',
WriteCheck => \&ValidateFunction,
```

**WriteCondition**: File-specific validation (`README:869-875`):

```perl
WriteCondition => '$$self{FILE_TYPE} eq "JPEG"',
```

**DelCheck**: Deletion validation (`README:862-867`):

```perl
DelCheck => '$val = ""; return undef',  # Set to empty instead of delete
```

### Advanced Writing Features

**WriteHook**: Custom write processing (`README:857-858`):

```perl
WriteHook => \&ProcessWriteHook,  # QuickTime only
```

**IsOverwriting**: Overwrite control (`README:888-891`):

```perl
IsOverwriting => \&CheckOverwrite,
```

**Shift**: Time shifting support (`README:802-805`):

```perl
Shift => 'Time',  # Enable time shifting
Shift => '0',     # Prevent shifting
```

## Validation and Testing

### Condition System

**Condition**: Conditional tag validity (`README:744-763`):

```perl
Condition => '$format eq "int16u"',           # Format-based
Condition => '$$self{Make} eq "Canon"',       # Context-based
Condition => '$count == 4',                   # Count-based
```

**Available Variables**: `$self`, `$valPt`, `$format`, `$count`

**WriteCondition**: Write-time validation:

```perl
WriteCondition => '$$self{FILE_TYPE} ne "TIFF"',
```

### Validation Framework

**Validate**: Value validation expression (`README:718-724`):

```perl
Validate => '$val >= 0 and $val <= 100 or "Value out of range"',
```

**Relist**: Value reorganization (`README:726-731`):

```perl
Relist => [0, 2, 1],          # Reorder values
Relist => [[0,1], 2],         # Join values 0,1
```

## Binary Data Properties

### Bit-Level Processing

**Mask**: Bit mask extraction (`README:733-737`):

```perl
Mask => 0x0f,     # Extract lower 4 bits
BitShift => 4,    # Shift after masking
```

**BitsPerWord**: Bitmask word size (`README:928-929`):

```perl
BitsPerWord => 16,  # 16-bit words for BITMASK tags
```

### Dynamic Processing

**Hook**: Dynamic format control (`README:965-975`):

```perl
Hook => '$format = $val == 1 ? "int16u" : "int32u"',
```

**Available Variables**: `$self`, `$size`, `$dataPt`, `$pos`, `$format`, `$varSize`

**LargeTag**: Memory optimization (`README:977-979`):

```perl
Flags => 'LargeTag',  # Don't store large data in %val hash
```

## Composite Tag Properties

### Source Dependencies

**Require**: Required source tags (`README:765-784`):

```perl
Require => {
    0 => 'EXIF:FNumber',
    1 => 'EXIF:ExposureTime',
},

Require => 'EXIF:ISO',  # Scalar shorthand
```

**Desire**: Optional source tags (`README:786-794`):

```perl
Desire => {
    2 => 'EXIF:Flash',
    3 => 'EXIF:LensModel',
},
```

**Inhibit**: Inhibiting tags (`README:796-800`):

```perl
Inhibit => {
    4 => 'EXIF:ColorSpace',  # Don't build if this exists
},
```

## XMP-Specific Properties

### Structure Support

**Struct**: Structure definition (`README:940-948`):

```perl
Struct => {
    STRUCT_NAME => 'ContactInfo',
    NAMESPACE => 'stDim',
    City => { },
    Country => { },
},
```

**Namespace**: XMP namespace (`README:953-954`):

```perl
Namespace => 'exif',  # Override table namespace
```

**FlatName**: Flattened tag naming (`README:956-960`):

```perl
FlatName => 'Dimension',  # Name for flattened tags
```

## Advanced Properties

### Specialized Format Properties

**Units**: Valid units specification (`README:962-963`):

```perl
Units => ['pixels', 'inches', 'cm'],  # MIE tags only
```

**ByteOrder**: Tag-specific byte order (`README:985-988`):

```perl
ByteOrder => 'II',  # Little-endian for this tag only
```

**SetBase**: Base offset control (`README:981-983`):

```perl
SetBase => '$val',  # Set ExifTool BASE for ReEntry
```

### Documentation Properties

**Notes**: Tag documentation (`README:296`):

```perl
Notes => 'calculated from multiple exposure readings',
```

**PrintConvColumns**: Documentation formatting (`README:710-712`):

```perl
PrintConvColumns => 3,  # Number of columns in docs
```

**SeparateTable**: Separate documentation table (`README:529-533`):

```perl
Flags => 'SeparateTable',  # List PrintConv values separately
```

### Internal Properties

**TagID**: Internal tag identifier (`README:995-998`):

```perl
# Reserved for internal use
TagID => '0x010f',  # Saved automatically
```

## Implementation Guidelines for exif-oxide

### Critical Architecture Elements

**Complete Property Support**: Implement all tag information hash properties:

- Core properties (Name, Description, Groups, Format)
- Conversion system (RawConv, ValueConv, PrintConv + inverses)
- Flag system (50+ flags controlling behavior)
- Writing system (Writable, WriteGroup, WriteAlso, validation)
- Validation framework (Condition, WriteCondition, Validate)

**Expression Evaluation**: Support Perl expression evaluation for:

- ValueConv/PrintConv expressions with `$val`, `$self`, `$tag` variables
- Condition expressions with format and context access
- Dynamic format expressions with data access variables
- Hook expressions for runtime format modification

### Conversion System Implementation

**Conversion Chain**: Preserve exact conversion order and semantics:

1. RawConv (early, while file open, may return undef)
2. ValueConv (logical conversion, programmatic values)
3. PrintConv (human-readable formatting)

**Hash Lookups**: Support hash-based conversions with special keys:

- BITMASK for bit-level decoding
- OTHER for unknown value handling
- Notes for documentation integration

**Expression Context**: Provide complete expression evaluation context:

- Tag value (`$val`) and ExifTool object (`$self`)
- Format information (`$format`, `$count`)
- Data access (`$valPt` for first 128 bytes)
- Group information and metadata path

### Flag System Implementation

**Flag Processing**: Expand flags into hash members for runtime efficiency:

```rust
// Pseudo-code for flag expansion
if tag_info.flags.contains("Binary") {
    tag_info.value_conv = Some("\\$val".to_string());
}
if tag_info.flags.contains("Protected") {
    tag_info.protected = Some(1);
}
```

**Priority System**: Implement tag priority resolution:

- Priority 0: Won't override previous tags
- Priority 1+: Override lower priority tags
- Special IFD handling for priority adjustment

**Write Protection**: Support multiple protection levels:

- Unsafe tags (bit 0x01): Require explicit specification
- Protected tags (bit 0x02): Prevent direct user modification
- Permanent tags: Cannot be deleted

### Memory and Performance Optimization

**Lazy Evaluation**: ValueConv and PrintConv only when requested
**Large Data Handling**: LargeTag flag for memory optimization
**Binary Data Optimization**: Direct binary handling without conversion
**Expression Caching**: Cache compiled expressions for performance

## Conclusion

ExifTool's tag information hash system represents a sophisticated metadata definition framework that enables consistent handling of enormous format diversity while maintaining extensibility and performance. The system's power lies in its comprehensive property set that covers every aspect of tag behavior from basic identification to complex conversion expressions and writing constraints.

**Key Architectural Strengths**:

- **Comprehensive Property Coverage**: 50+ properties handle all aspects of tag behavior
- **Flexible Conversion System**: Three-stage conversion chain with expressions and lookups
- **Sophisticated Flag System**: Fine-grained control over tag characteristics
- **Advanced Writing Support**: Complete write system with validation and cascading
- **Format Adaptability**: Dynamic format handling for complex manufacturer-specific data

**Critical Implementation Requirements**:

- Complete property system preservation for format compatibility
- Full expression evaluation support for dynamic behavior
- Exact conversion chain semantics for value accuracy
- Comprehensive flag system for behavioral control
- Performance optimization for real-world usage

The tag information hash system enables ExifTool to handle the incredible complexity of metadata formats across hundreds of manufacturers and software implementations. Understanding this system is essential for implementing metadata processing that works reliably with the diverse and often non-standard implementations found in digital media files.

For exif-oxide implementation, preserving the exact tag information hash architecture is critical for maintaining compatibility with ExifTool's 25+ years of format-specific adaptations and the thousands of tag definitions that enable comprehensive metadata support.
