# ExifTool WRITE_PROC Deep Dive

## Overview

The `WRITE_PROC` system handles writing metadata back to files. **For the formal specification, see `lib/Image/ExifTool/README` lines 61-65.** This document focuses on implementation details, edge cases, and tribal knowledge not covered in the README.

## Core Architecture

### WRITE_PROC Definition

**Per README:** Function reference returning new directory data or undefined on error. Takes same arguments as PROCESS_PROC except the directory information hash is optional for source copying.

### Function Signature

```perl
sub WriteFormatName($$$)
{
    my ($et, $dirInfo, $tagTablePtr) = @_;
    # $et         - ExifTool object reference
    # $dirInfo    - Directory information hash reference (optional for source)
    # $tagTablePtr - Tag table hash reference

    # Returns: new directory data or undef on error
}
```

### Directory Information Hash

**Complete specification:** See `lib/Image/ExifTool/README` lines 42-59 for all directory information hash entries.

**Key write-specific properties:**

- `NewDataPos` - File position for new data (write proc only)
- `Fixup` - Offset fixup hash (EXIF writing only)
- `LastIFD`, `ImageData` - EXIF-specific write coordination

## Common WRITE_PROC Implementations

### 1. WriteBinaryData (lib/Image/ExifTool.pm)

**Most Common Implementation**: Used for simple binary data structures.

```perl
# Example usage from multiple modules
WRITE_PROC => \&Image::ExifTool::WriteBinaryData,
```

**Behavior**:

- Handles fixed-format binary data blocks
- Supports `DATAMEMBER` entries for variable-length fields (**see README:263-266**)
- Validates tag formats and sizes
- Manages `var_*` format adjustments automatically (**see README:316-323**)

### 2. WriteExif (lib/Image/ExifTool/Exif.pm)

**EXIF/TIFF Format Handler**: Used for IFD-based formats.

```perl
# Example from multiple camera manufacturers
WRITE_PROC => \&Image::ExifTool::Exif::WriteExif,
```

**Complex Behaviors**:

- IFD (Image File Directory) construction
- Offset fixup management via `$$dirInfo{Fixup}`
- Multi-IFD linking and validation
- Maker note rebuilding and base adjustment
- Sub-IFD handling and validation

### 3. Format-Specific Procedures

Many formats implement custom write procedures:

```perl
# Photoshop IRB resources
WRITE_PROC => \&WritePhotoshop,

# XMP/RDF XML
WRITE_PROC => \&WriteXMP,

# IPTC record format
WRITE_PROC => \&WriteIPTC,

# QuickTime atom structure
WRITE_PROC => \&WriteQuickTime,
```

## Format-Specific Write Procedures

### WritePhotoshop (lib/Image/ExifTool/WritePhotoshop.pl)

**IRB Resource Block Format**:

```perl
sub WritePhotoshop($$$)
{
    my ($et, $dirInfo, $tagTablePtr) = @_;

    # Always big-endian format
    SetByteOrder('MM');

    # IRB format: Type(4) + TagID(2) + Name(pascal) + Size(4) + Data(N)
    # Each entry must align to even byte boundaries
    # Handles resource types: '8BIM', 'PHUT', 'DCSR', 'AgHg', 'MeSa'

    return $newData;  # or undef on error
}
```

**Key Features**:

- **Alignment Requirements**: Each resource block must start on even byte boundary
- **Multiple Resource Types**: Not just '8BIM' - legacy support for other types
- **Pascal String Handling**: Resource names use pascal format with padding
- **Size Validation**: Extensive validation of resource block sizes

### WriteXMP (lib/Image/ExifTool/WriteXMP.pl)

**RDF/XML Structure Writing**:

- **Namespace Management**: Dynamic namespace prefix allocation
- **Structure Flattening**: Converts hierarchical data to flat tag representation
- **List Processing**: Handles Bag, Seq, and Alt list types
- **Character Encoding**: UTF-8 validation and encoding

### WriteQuickTime (lib/Image/ExifTool/WriteQuickTime.pl)

**Hierarchical Atom Structure**:

- **Atom format**: Size(4) + Type(4) + Data(Size-8)
- **Container atoms**: Nested sub-atom structures
- **Key Dictionary**: iTunes-style keyed metadata
- **64-bit support**: Large atom handling

## Error Handling and Validation

### Return Value Protocol

**Success/Failure Indication**:

- **Success**: Return new directory data (scalar or scalar reference)
- **Failure**: Return `undef`
- **Empty Directory**: Return empty string `''`

### CHECK_PROC Integration

**CHECK_PROC Function Signature:** See `lib/Image/ExifTool/README` lines 67-72 for complete specification.

**Error Recovery Strategies**:

1. **Tag Skipping**: Skip invalid tags, continue with others
2. **Format Fallback**: Use alternative encodings when possible
3. **Warning Generation**: Issue warnings rather than fatal errors
4. **Partial Success**: Return partial data when some tags succeed

## Special Cases and Edge Conditions

### 1. DummyWriteProc

**MWG Module Pattern**:

```perl
# lib/Image/ExifTool/MWG.pm
WRITE_PROC => \&Image::ExifTool::DummyWriteProc,
```

**Purpose**: For pseudo-groups that don't actually write data but trigger other write operations through WriteAlso tags.

### 2. SubDirectory WriteProc Override

**SubDirectory-Specific Writers**:

```perl
SubDirectory => {
    TagTable => 'Image::ExifTool::Panasonic::DistortionInfo',
    WriteProc => \&WriteDistortionInfo,  # Override table default
}
```

**Use Cases**:

- Format-specific encoding within larger structures
- Custom validation or processing requirements
- Proprietary data formats within standard containers

### 3. MakerNote Write Handling

**Block-Writable Configuration**:

```perl
# All MakerNote subdirectories are marked as block-writable
foreach $tagInfo (@Image::ExifTool::MakerNotes::Main) {
    $$tagInfo{Writable} = 'undef';    # Block-writable format
    $$tagInfo{Binary} = 1;            # Binary data flag
    $$tagInfo{MakerNotes} = 1;        # Special MakerNote flag
}
```

**Write Limitations**:

- **Phase One**: Rebuilding not supported (`IsPhaseOne` flag)
- **GE Type 2**: "Maker notes could not be parsed" warning
- **Leica Trailers**: Special handling for JPEG trailer format

## Write Process Flow

### High-Level Write Sequence

1. **Tag Collection**: Gather new values from `$$et{NEW_VALUE}`
2. **Source Analysis**: Parse existing directory data (if provided)
3. **Tag Merging**: Combine new and existing tags according to write options
4. **Validation**: Apply CHECK_PROC validation to all new values
5. **Format Encoding**: Convert values to binary format
6. **Structure Building**: Construct new directory/container structure
7. **Offset Management**: Calculate and register offset fixups
8. **Size Calculation**: Determine final data size
9. **Data Assembly**: Build final binary output

### Integration with File Writers

**Writer Chain Coordination**:

- WRITE_PROC generates directory data
- File format writer (WriteJPEG, WriteTIFF, etc.) handles file-level structure
- Offset fixup system manages inter-directory references
- Trailer management handles appended data blocks

## Tribal Knowledge and Surprises

### 1. The "NewDataPos" Critical Detail

**From README**: `NewDataPos` is marked as "write proc only" - this is the file position where the new data will be written. This is crucial for:

- Calculating absolute offsets for fixup
- Managing base offset adjustments
- Coordinating with file-level writers

### 2. Optional Source Directory Parameter

**Surprise**: The second parameter (dirInfo) is optional. When provided, it contains source directory information for copying existing tags. When omitted, the write procedure should only process new values from `$$et{NEW_VALUE}`.

### 3. EXIF Fixup System Complexity

**Hidden Complexity**: The `$$dirInfo{Fixup}` hash is only used in EXIF writing but represents one of the most complex offset management systems in ExifTool:

- Tracks interdependent offset relationships
- Handles circular dependencies between IFDs
- Manages maker note base adjustments
- Coordinates with image data placement

### 4. Binary Flag Interaction

**Subtle Behavior**: The `Binary => 1` flag affects how WriteBinaryData processes values:

- Binary values are treated as raw data blocks
- Non-binary values undergo format conversion
- Mixed binary/non-binary tables require careful handling

### 5. MakerNote Write Protection Philosophy

**Design Decision**: MakerNotes are block-writable by design philosophy:

- Individual tag editing could break proprietary format relationships
- Offset interdependencies make selective editing dangerous
- Manufacturer-specific encodings are too complex for safe reconstruction
- Block replacement preserves internal consistency

### 6. Byte Order Independence

**Write-Time Byte Order**: Write procedures must handle byte order correctly:

- Some formats (Photoshop) force specific byte order
- Others inherit from file-level byte order
- MakerNotes preserve original endianness
- EXIF IFDs can have mixed byte ordering

### 7. Size Constraint Management

**Memory Efficiency**: Write procedures must consider:

- Large image data block handling (`$$dirInfo{ImageData}`)
- Memory usage for huge directories
- Streaming vs. buffering decisions
- Size validation before allocation

## Development Guidelines

### Adding New WRITE_PROC Functions

**Essential Patterns**:

1. **Signature Consistency**: Always use `($$$)` signature
2. **Return Value Protocol**: Return data or undef, never die/throw
3. **Byte Order Management**: Set and restore byte order appropriately
4. **Error Handling**: Use ExifTool error reporting methods
5. **Validation Integration**: Support CHECK_PROC validation
6. **Offset Awareness**: Handle offset calculations correctly

### Best Practices

**Format-Specific Considerations**:

```perl
sub WriteNewFormat($$$)
{
    my ($et, $dirInfo, $tagTablePtr) = @_;

    # 1. Validate arguments
    return undef unless $et and $tagTablePtr;

    # 2. Set byte order if format-specific
    my $oldOrder = GetByteOrder();
    SetByteOrder('MM') if $formatRequiresBigEndian;

    # 3. Get new tag values
    my $newTags = $et->GetNewTagInfoHash($tagTablePtr);

    # 4. Process source data if provided
    my %srcTags;
    if ($dirInfo) {
        # Parse existing directory
        # Extract current tags
    }

    # 5. Merge and validate tags
    # 6. Build new directory structure
    # 7. Calculate final size and offsets

    # 8. Restore byte order
    SetByteOrder($oldOrder);

    # 9. Return result
    return $newData;  # or undef on error
}
```

### Testing Requirements

**Comprehensive Test Coverage**:

- **Round-trip Testing**: Write then read to verify data integrity
- **Error Condition Testing**: Invalid values, corrupted data, size limits
- **Edge Case Testing**: Empty directories, maximum sizes, unusual formats
- **Integration Testing**: Coordination with file-level writers
- **Performance Testing**: Large directory handling, memory usage

### Documentation Requirements

**For complete tag table structure, format specifications, and validation rules, see `lib/Image/ExifTool/README`.**

**Additional WRITE_PROC Documentation**:

1. **Format Specification**: Document the binary format being written
2. **Limitation Notes**: Document any write limitations or restrictions
3. **Example Usage**: Provide working examples of tag definitions
4. **Error Conditions**: Document all possible error conditions and causes

## Conclusion

The WRITE_PROC system represents the sophisticated heart of ExifTool's metadata writing capabilities. It demonstrates several key design principles:

1. **Format Flexibility**: Each format can implement custom encoding logic while sharing common infrastructure
2. **Error Resilience**: Graceful handling of edge cases and invalid data
3. **Performance Awareness**: Memory-efficient handling of large data structures
4. **Backward Compatibility**: Support for legacy formats and deprecated features
5. **Validation Integration**: Comprehensive validation at multiple levels

Understanding WRITE_PROC is essential for:

- **Adding New Format Support**: Implementing write capabilities for new metadata formats
- **Debugging Write Issues**: Understanding why certain write operations fail
- **Performance Optimization**: Optimizing write operations for specific use cases
- **Integration Development**: Building applications that leverage ExifTool's write capabilities

The complexity of the WRITE_PROC system reflects the real-world complexity of metadata formats - each with unique quirks, requirements, and edge cases that must be handled correctly to ensure data integrity and format compliance.
