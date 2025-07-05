# ExifTool Write System: Complete Metadata Modification Architecture

**ExifTool Version:** 13.26  
**Document Version:** 1.0  
**Last Updated:** 2025-07-04

## Overview

ExifTool's write system is a sophisticated framework that enables safe, reliable metadata modification while preserving file integrity and handling the complex constraints of real-world metadata formats. The system supports writing to hundreds of file formats while maintaining structural integrity, handling offset updates, and preserving unknown or proprietary data.

**Core Capability**: The write system can modify metadata in place, add new tags, delete existing tags, and restructure metadata directories while maintaining file validity and preserving data that ExifTool doesn't understand.

## Write System Architecture

### Three-Tier Processing Framework

**1. Validation Tier**: Check and validate new values before writing
**2. Processing Tier**: Transform and convert values for writing  
**3. Serialization Tier**: Generate binary output with proper formatting

### Core Write Components

**WRITE_PROC**: Format-specific write function (`Exif.pm:389`):

```perl
%Image::ExifTool::Exif::Main = (
    WRITE_PROC => \&WriteExif,    # Main write function
    CHECK_PROC => \&CheckExif,    # Validation function
    WRITE_GROUP => 'ExifIFD',     # Default write location
);
```

**CHECK_PROC**: Validation and constraint checking
**WRITE_GROUP**: Default directory for writing tags
**Writable Properties**: Individual tag write specifications

## Tag-Level Write Control

### Writable Property System

**Basic Writability** (`README:807-827`):

```perl
# Simple writable specification
Writable => 'string',      # Format for writing
Writable => 'int16u',      # 16-bit unsigned integer
Writable => 1,             # Use existing format from file
Writable => 0,             # Not writable (read-only)
```

**Binary Data Writability** (`Canon.pm:1175-1180`):

```perl
# Shared attributes for writable binary data
my %binaryDataAttrs = (
    PROCESS_PROC => \&Image::ExifTool::ProcessBinaryData,
    WRITE_PROC => \&Image::ExifTool::WriteBinaryData,
    CHECK_PROC => \&Image::ExifTool::CheckBinaryData,
    WRITABLE => 1,
);
```

### WriteGroup Specifications

**Group-Based Writing** (`Exif.pm:398,853,etc.`):

```perl
# Target-specific write groups
WriteGroup => 'InteropIFD',    # Write to Interoperability IFD
WriteGroup => 'IFD0',          # Write to main image IFD
WriteGroup => 'ExifIFD',       # Write to EXIF subdirectory
WriteGroup => 'All',           # Write to multiple locations
```

**Write Group Hierarchy**:

1. **Tag-Level WriteGroup**: Highest priority, overrides all defaults
2. **Table-Level WRITE_GROUP**: Default for all tags in table
3. **Group1 Location**: Fallback based on tag's Group1 assignment
4. **Format Default**: Last resort based on file format

## Value Conversion for Writing

### Inverse Conversion Chain

Writing applies conversions in reverse order from reading:
**User Input** → **PrintConvInv** → **ValueConvInv** → **RawConvInv** → **Binary Output**

### PrintConvInv: Human-Readable to Logical

**String to Value Conversion**:

```perl
PrintConvInv => '$val =~ s/ seconds$//; $val',     # Remove units
PrintConvInv => '$val =~ s/mm$//; $val',           # Remove suffix
PrintConvInv => {                                  # Lookup conversion
    'Auto' => 0,
    'Manual' => 1,
    'Program' => 2,
},
```

### ValueConvInv: Logical to Raw

**Mathematical Inverse Conversion**:

```perl
ValueConvInv => '$val * 32',                      # Reverse division
ValueConvInv => '32*log($val/100)/log(2)',        # Complex inverse
ValueConvInv => \&InvertExposureTime,             # Function reference
```

### RawConvInv: Dynamic Raw Processing

**Runtime Raw Value Generation** (`README:679-688`):

```perl
RawConvInv => '$val',  # Direct pass-through (most common)
# Rare usage for dynamic raw value calculation
```

**Applied During Writing**: Unlike other conversions which happen during SetNewValue, RawConvInv executes during WriteInfo() when file context is available.

## Write Validation System

### CHECK_PROC Framework

**Format-Specific Validation** (`Canon.pm:1188`):

```perl
CHECK_PROC => \&Image::ExifTool::Exif::CheckExif,  # Standard EXIF validation
CHECK_PROC => \&CheckCanon,                        # Canon-specific validation
```

**CHECK_PROC Signature**:

```perl
sub CheckProc($$$$) {
    my ($et, $tagInfo, $valuePtr, $convType) = @_;
    # Returns: undef on success, error message on failure
}
```

### Tag-Level Validation

**WriteCheck**: Pre-write validation (`README:848-855`):

```perl
WriteCheck => '$val > 0 or "Value must be positive"',
WriteCheck => '$self->CheckImage(\$val)',          # Image validation
WriteCheck => \&ValidateCustomFormat,              # Function reference
```

**WriteCondition**: Context-specific validation (`README:869-875`):

```perl
WriteCondition => '$$self{FILE_TYPE} eq "JPEG"',   # File type constraint
WriteCondition => '$format eq "int16u"',           # Format requirement
```

**DelCheck**: Deletion validation (`README:862-867`):

```perl
DelCheck => '$val = ""; return undef',             # Set empty instead of delete
DelCheck => '$self->CheckDeleteConstraints($tagInfo)',
```

## WriteAlso: Cascading Updates

### Synchronized Writing

**Automatic Tag Synchronization** (`Exif.pm:7853-7859`):

```perl
WriteAlso => {
    'EXIF:ISO' => '$val',              # Write same value to EXIF
    'XMP:ISO' => '$val',               # Write same value to XMP
    'IPTC:ISO' => '$val',              # Write same value to IPTC
},
```

**Conditional Writing**:

```perl
WriteAlso => {
    PreviewImageStart  => 'defined $val ? 0xfeedfeed : undef',
    PreviewImageLength => 'defined $val ? 0xfeedfeed : undef',
    PreviewImageValid  => 'defined $val and length $val ? 1 : 0',
},
```

**Available Variables in WriteAlso**:

- `$val`: New raw value of parent tag (may be undef for deletion)
- `%opts`: Hash for modifying SetNewValue options
- Default options: `Type => "ValueConv"`, `Protected => 0x02`

### WriteAlso Processing Context

**Evaluation Context** (`README:829-846`):

- Parent tag options (AddValue, DelValue, Shift, Replace) inherited
- Previous WriteAlso tag values removed if Replace option used
- Empty warning (`"\n"`) suppresses target tag write without error
- Expression failure propagates as write error

## Protected and Permanent Tags

### Protection Levels

**Protected Flag System** (`README:509-513`):

```perl
Protected => 1,     # Bit 0x01: Unsafe tag (requires explicit specification)
Protected => 2,     # Bit 0x02: Protected from direct user modification
Protected => 3,     # Both protection levels
```

**Protection Behavior**:

- **Unsafe Tags**: Not copied by SetNewValuesFromFile() unless explicitly named
- **Protected Tags**: Cannot be set directly by users, only via WriteAlso or system

### Permanent Tags

**Permanent Flag** (`README:473-476`):

```perl
Flags => 'Permanent',   # Cannot be deleted from file
```

**Permanent Tag Behavior**:

- All MakerNotes tags are permanent by default
- Cannot be deleted, but values can be modified
- DelValue property specifies value to use when "deleting"

## Binary Data Writing

### Binary Data Framework

**Shared Write Attributes** (`Canon.pm:1175-1180`):

```perl
my %binaryDataAttrs = (
    PROCESS_PROC => \&Image::ExifTool::ProcessBinaryData,
    WRITE_PROC => \&Image::ExifTool::WriteBinaryData,
    CHECK_PROC => \&Image::ExifTool::CheckBinaryData,
    WRITABLE => 1,
);
```

**Binary Write Requirements**:

- **DATAMEMBER**: Tags that must be extracted for offset calculations
- **IS_OFFSET**: Offset-type tags requiring adjustment
- **IS_SUBDIR**: Subdirectory tags requiring special handling

### Variable-Length Binary Data

**Dynamic Format Support**:

```perl
Format => 'string[$val{3}]',        # Length from previous tag
Format => 'var_int16u[10]',         # Variable-length with offset adjustment
```

**Offset Management**: Variable-length formats automatically adjust subsequent tag positions during writing.

## Offset and Pointer Management

### Automatic Offset Updates

**IsOffset Tag Handling**: Tags marked with IsOffset flag automatically updated during writing:

```perl
IsOffset => 1,          # Standard offset adjustment
OffsetPair => 0x0117,   # Associated byte count tag
DataTag => 'ImageData', # Data being referenced
```

**Offset Update Process**:

1. Calculate new positions for all data blocks
2. Update offset tags to point to new locations
3. Ensure offset/length pairs remain consistent
4. Handle subdirectory pointer updates

### Fixup System

**Offset Fixup Framework**: Handles complex offset interdependencies during writing:

- Deferred offset calculation for forward references
- Circular dependency resolution
- Multiple-pass offset resolution
- Base offset adjustments for subdirectories

## Write Process Flow

### SetNewValue Phase

**Value Setting Process**:

1. **Input Validation**: Check format and constraints
2. **Conversion Chain**: Apply PrintConvInv → ValueConvInv → RawConvInv
3. **Storage**: Store converted value for later writing
4. **WriteAlso Processing**: Handle cascading updates

### WriteInfo Phase

**File Modification Process**:

1. **File Reading**: Read existing metadata structure
2. **Value Integration**: Merge new values with existing structure
3. **Structure Update**: Add/remove/modify directory entries
4. **Offset Calculation**: Update all pointers and offsets
5. **Binary Generation**: Create new binary metadata structure
6. **File Writing**: Write updated file with preserved unknown data

## Format-Specific Writing

### EXIF/TIFF Writing

**IFD Structure Management**:

- Preserve directory chain integrity
- Handle subdirectory creation and deletion
- Manage offset calculations for variable-sized data
- Support BigTIFF for large files

### Maker Notes Writing

**Manufacturer-Specific Handling**:

- Preserve proprietary structure formats
- Handle encrypted sections appropriately
- Maintain offset base calculations
- Support format-specific quirks

### XMP Writing

**XML Structure Management**:

- Namespace handling and prefix assignment
- Structured data serialization
- Language variant support
- Flattened tag reconstruction

## Write Safety and Integrity

### Data Preservation

**Unknown Data Handling**: ExifTool preserves data it doesn't understand:

- Unknown directory entries copied unchanged
- Unrecognized binary data preserved
- Proprietary structures maintained
- File format compliance preserved

### Backup and Recovery

**File Safety Mechanisms**:

- Automatic backup creation (unless disabled)
- Original file preservation
- Atomic write operations where possible
- Error recovery with original file restoration

### Validation Integration

**Pre-Write Validation**:

- Format constraint checking
- Value range validation
- Cross-reference consistency
- File format compliance verification

## Advanced Write Features

### Conditional Writing

**Context-Aware Writing**:

```perl
Condition => '$format eq "int16u"',        # Format-based conditions
WriteCondition => '$$self{Make} eq "Canon"', # Camera-specific writing
```

### Batch Operations

**Multiple Tag Updates**: Efficient handling of multiple tag modifications in single operation
**Group Operations**: Write multiple tags to same directory efficiently
**Transaction Support**: All-or-nothing updates for complex modifications

### Custom Write Functions

**Format-Specific Writers**:

- Custom WRITE_PROC for proprietary formats
- Specialized handling for encrypted sections
- Format conversion during writing
- Structure reconstruction for complex formats

## Implementation Guidelines for exif-oxide

### Core Write Architecture

**Validation Framework**: Implement complete validation system:

- CHECK_PROC function support
- WriteCheck expression evaluation
- WriteCondition context validation
- DelCheck deletion constraints

**Conversion System**: Support full inverse conversion chain:

- PrintConvInv for human-readable input
- ValueConvInv for logical value conversion
- RawConvInv for dynamic raw generation
- Error handling for conversion failures

### Write Safety Implementation

**Data Preservation**: Maintain ExifTool's data preservation guarantees:

- Unknown data copying
- Structure integrity maintenance
- Offset calculation accuracy
- File format compliance

**Error Recovery**: Implement robust error handling:

- Pre-write validation
- Atomic operations where possible
- Rollback capability for failures
- Clear error reporting

### Performance Optimization

**Efficient Writing**: Optimize for real-world usage:

- Minimize file I/O operations
- Batch related updates
- Cache directory structures
- Defer expensive calculations

**Memory Management**: Handle large files efficiently:

- Streaming for large data blocks
- Memory limits for safety
- Efficient offset calculation
- Lazy loading of unchanged sections

## Conclusion

ExifTool's write system represents a sophisticated framework that enables safe, reliable metadata modification across hundreds of file formats. The system's power lies in its combination of flexibility and safety - supporting complex metadata modifications while preserving file integrity and unknown data.

**Key Architectural Strengths**:

- **Comprehensive Validation**: Multi-level validation prevents data corruption
- **Data Preservation**: Unknown and proprietary data maintained during modification
- **Format Flexibility**: Supports diverse metadata formats with unified interface
- **Write Safety**: Backup and recovery mechanisms prevent data loss

**Critical Implementation Requirements**:

- Complete validation framework preservation
- Exact conversion chain semantics
- Data preservation guarantees
- Performance optimization for real-world usage

The write system's complexity reflects the challenge of safely modifying metadata across the enormous diversity of digital media formats. Every validation rule, conversion mechanism, and safety feature represents solutions to actual problems encountered when modifying real-world files.

For exif-oxide implementation, preserving the exact write system architecture is essential for maintaining ExifTool's reputation for safe, reliable metadata modification. The system must handle not just writing new values, but doing so safely while preserving everything ExifTool doesn't understand - a critical requirement for working with proprietary and evolving metadata formats.
