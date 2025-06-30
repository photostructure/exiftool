# ExifTool Common Patterns

**ExifTool Version:** 13.26  
**Document Version:** 1.0  
**Last Updated:** 2025-06-29

This document describes recurring patterns across ExifTool modules to help developers understand and maintain the codebase.

## Tag Table Structure Pattern

All ExifTool modules define tag tables as Perl hashes following this pattern:

```perl
%Image::ExifTool::ModuleName::Main = (
    # Special keys (uppercase)
    PROCESS_PROC => \&ProcessModuleName,
    WRITE_PROC => \&WriteModuleName,
    GROUPS => { 0 => 'GroupName', 1 => 'SubGroup', 2 => 'Category' },
    NOTES => 'Documentation text',
    
    # Tag definitions (usually hex keys for binary formats)
    0x0001 => 'TagName',
    0x0002 => {
        Name => 'ComplexTag',
        Format => 'int16u',
        ValueConv => '$val * 2',
    },
);
```

## Module Organization Patterns

### 1. Basic Format Module Pattern
Simple formats (BMP, GIF, PCX) follow this structure:
- Single main tag table
- Process proc reads fixed header
- Minimal or no subdirectories
- Direct value extraction

### 2. IFD-Based Module Pattern  
EXIF-style modules use Image File Directory structures:
- Multiple linked IFD tables
- Subdirectory navigation
- Pointer-based offsets
- Base offset management

### 3. Maker Note Module Pattern
Camera manufacturer modules typically have:
```perl
# Main maker note table
%Image::ExifTool::Canon::Main = ( ... );

# Specialized subtables
%Image::ExifTool::Canon::CameraSettings = ( ... );
%Image::ExifTool::Canon::FocusInfo = ( ... );

# Custom function tables  
%Image::ExifTool::CanonCustom::Functions1D = ( ... );
```

### 4. Container Format Pattern
QuickTime, RIFF, MXF use hierarchical atoms/chunks:
- Recursive atom/chunk processing
- Variable-length structures
- Conditional processing based on type
- Track/stream handling

## Processing Procedure Patterns

### Standard Process Proc
```perl
sub ProcessModuleName($$$)
{
    my ($et, $dirInfo, $tagTablePtr) = @_;
    my $dataPt = $$dirInfo{DataPt};
    my $pos = $$dirInfo{DirStart} || 0;
    my $dirLen = $$dirInfo{DirLen};
    
    # Validate
    return 0 if $dirLen < $minSize;
    
    # Process tags
    my $tagID = Get16u($dataPt, $pos);
    $et->HandleTag($tagTablePtr, $tagID, $value);
    
    return 1;
}
```

### Binary Data Pattern
For fixed-layout binary structures:
```perl
%Image::ExifTool::Module::Table = (
    PROCESS_PROC => \&Image::ExifTool::ProcessBinaryData,
    FORMAT => 'int8u',  # default format
    FIRST_ENTRY => 0,
    0 => { Name => 'Version', Format => 'int16u' },
    2 => { Name => 'Width', Format => 'int32u' },
);
```

## Common Tag Patterns

### SubDirectory Tags
```perl
0x927c => {
    Name => 'MakerNotes',
    SubDirectory => {
        TagTable => 'Image::ExifTool::MakerNotes::Main',
        Start => '$valuePtr',
        Base => '$start',
        ByteOrder => 'Unknown',
    },
},
```

### Conditional Tags
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

### List Tags
```perl
Keywords => {
    Flags => 'List',
    Writable => 'string',
    List => 'Bag',  # for XMP
},
```

## Value Conversion Patterns

### Lookup Tables
```perl
ValueConv => {
    0 => 'Auto',
    1 => 'Manual',
    2 => 'Program',
},
```

### Mathematical Conversions
```perl
ValueConv => '$val / 32',        # division
ValueConv => '2 ** ($val / 32)', # exponential
ValueConv => 'int($val + 0.5)',  # rounding
```

### Complex Conversions
```perl
ValueConv => 'Image::ExifTool::Exif::ConvertFraction($val)',
PrintConv => 'Image::ExifTool::Exif::PrintExposureTime($val)',
```

## Offset Handling Patterns

### IsOffset Tags
```perl
{
    Name => 'PreviewImageStart',
    Flags => 'IsOffset',
    OffsetPair => 0x0202,  # PreviewImageLength
    DataTag => 'PreviewImage',
    Writable => 'int32u',
    Protected => 2,
},
```

### FixOffsets Expression
```perl
FixOffsets => '$valuePtr += $base if $format eq "int32u"',
```

## Writing Patterns

### WriteAlso Pattern
```perl
UserComment => {
    Writable => 'undef',
    WriteAlso => {
        ExifUserComment => '$val',
        XMP-exif:UserComment => '$val',
    },
},
```

### Protected/Permanent Tags
```perl
{
    Name => 'ImportantTag',
    Writable => 'string',
    Flags => 'Permanent',  # can't delete
    Protected => 1,        # requires explicit write
},
```

## Module Initialization Pattern

```perl
# Auto-load additional tables
{
    my $tableName;
    sub GetTableName() { $tableName }
    sub SetTableName($) { $tableName = shift }
}

# Load custom tables on demand
sub LoadCustomTable($)
{
    my $custom = shift;
    my $tableName = "Image::ExifTool::Module::$custom";
    no strict 'refs';
    return \%$tableName if %$tableName;
    # ... load table ...
}
```

## Error Handling Patterns

### Validation
```perl
return 0 unless $dirLen >= 8;
$et->Warn('Invalid data') if $val < 0;
```

### Format Errors
```perl
unless ($format eq 'int16u') {
    $et->Warn("Unexpected format ($format) for $tag");
    return undef;
}
```

## Special Keys Reference

Most commonly used special keys in tag tables:
- **PROCESS_PROC**: Function to read this format
- **WRITE_PROC**: Function to write this format  
- **GROUPS**: Group hierarchy
- **FORMAT**: Default binary format
- **FIRST_ENTRY**: First tag ID for scanning
- **NOTES**: Documentation
- **WRITABLE**: Makes all tags writable
- **VARS**: Additional parameters