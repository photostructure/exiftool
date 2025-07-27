# ExifTool Tag Table Switching During Subdirectory Processing

**ExifTool Version:** 13.26  
**Document Version:** 1.0  
**Last Updated:** 2025-07-27

## Overview

This document explains how ExifTool switches tag table context when processing subdirectories, particularly focusing on how the same tag ID (e.g., 0x0001) resolves to different tags in different contexts (e.g., EXIF vs GPS).

## Core Mechanism

### 1. Subdirectory Definition with TagTable

When a tag is defined as a subdirectory with a different tag table, ExifTool switches the lookup context. Example from `Exif.pm:3019-3031`:

```perl
0x8825 => {
    Name => 'GPSInfo',
    Groups => { 1 => 'GPS' },
    WriteGroup => 'IFD0',
    Flags => 'SubIFD',
    SubDirectory => {
        DirName => 'GPS',
        TagTable => 'Image::ExifTool::GPS::Main',  # <-- Tag table switch
        Start => '$val',
        MaxSubdirs => 1,
    },
},
```

### 2. Tag Table Resolution in ProcessExif

When ProcessExif encounters a subdirectory tag, it resolves the new tag table (`Exif.pm:6832-6838`):

```perl
if ($$subdir{TagTable}) {
    $newTagTable = GetTagTable($$subdir{TagTable});
    $newTagTable or warn("Unknown tag table $$subdir{TagTable}"), next;
} else {
    $newTagTable = $tagTablePtr;    # use existing table
}
```

### 3. ProcessDirectory Call with New Table

The subdirectory is processed with the new tag table (`Exif.pm:6978`):

```perl
$ok = $et->ProcessDirectory(\%subdirInfo, $newTagTable, $$subdir{ProcessProc});
```

### 4. ProcessDirectory Function

The ProcessDirectory function (`ExifTool.pm:8944-8988`) passes the tag table to the processor:

```perl
sub ProcessDirectory($$$;$)
{
    my ($self, $dirInfo, $tagTablePtr, $proc) = @_;
    # ...
    # $tagTablePtr is the new table (e.g., GPS::Main)
    $proc or $proc = $$tagTablePtr{PROCESS_PROC} || \&Image::ExifTool::Exif::ProcessExif;
    # ...
    my $rtnVal = &$proc($self, $dirInfo, $tagTablePtr);
}
```

### 5. Tag Lookup in New Context

When ProcessExif is called again for the subdirectory, it uses the new tag table for lookups (`Exif.pm:6375`):

```perl
my $tagInfo = $et->GetTagInfo($tagTablePtr, $tagID);
```

## Example: GPS Tag Resolution

### In EXIF Context
- `$tagTablePtr = \%Image::ExifTool::Exif::Main`
- Tag 0x0001 = InteropIndex

### In GPS Context
- `$tagTablePtr = \%Image::ExifTool::GPS::Main`
- Tag 0x0001 = GPSLatitudeRef

The tag ID is identical (0x0001), but the lookup table is different, resulting in completely different tag resolutions.

## Key Points

1. **Tag Table Parameter**: The `$tagTablePtr` parameter is passed through the entire processing chain
2. **Context Isolation**: Each subdirectory processes with its own tag table, providing complete context isolation
3. **Recursive Architecture**: ProcessDirectory can call ProcessExif recursively with different tag tables
4. **Dynamic Resolution**: Tag lookups always use the current `$tagTablePtr`, not a global table

## Implementation Details

### GetTagInfo Function

The GetTagInfo function (`ExifTool.pm:3652`) performs the actual tag lookup:

```perl
sub GetTagInfo($$$;$$$)
{
    my ($self, $tagTablePtr, $tagID) = @_;
    my @infoArray = GetTagInfoList($tagTablePtr, $tagID);
    # ...
}
```

### Tag Table Structure

Each tag table (like GPS::Main) is a hash with:
- Numeric keys (tag IDs) mapping to tag information
- Special keys (GROUPS, WRITE_PROC, etc.) for table metadata

Example from `GPS.pm:51-76`:

```perl
%Image::ExifTool::GPS::Main = (
    GROUPS => { 0 => 'EXIF', 1 => 'GPS', 2 => 'Location' },
    WRITE_PROC => \&Image::ExifTool::Exif::WriteExif,
    0x0001 => {
        Name => 'GPSLatitudeRef',
        # ...
    },
    0x0002 => {
        Name => 'GPSLatitude',
        # ...
    },
);
```

## Implications for exif-oxide

When implementing subdirectory processing in exif-oxide:

1. **Pass Tag Table Context**: Ensure the tag table reference is passed through all processing functions
2. **Isolate Lookups**: Tag lookups must use the current context's tag table, not a global lookup
3. **Support Table Switching**: Implement the GetTagTable mechanism for resolving table names to table references
4. **Maintain State**: Track the current tag table context throughout recursive processing

This architecture allows ExifTool to reuse the same processing code (ProcessExif) while completely changing the tag interpretation context, enabling efficient handling of nested metadata structures with overlapping tag ID spaces.