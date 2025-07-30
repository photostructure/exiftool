# COMPOSITE_TAGS.md

This document explains ExifTool's Composite tag infrastructure - a powerful system for deriving new tags from existing tag values.

## Overview

Composite tags are convenience tags calculated after all other information is extracted. They derive their values from one or more source tags using custom logic. This system enables:

- Consolidating information from multiple tags into a single value
- Format conversion and unit calculations
- Fallback logic when preferred tags are missing
- Cross-format data synthesis (e.g., combining EXIF and XMP data)

**Important:** Some composite tags exhibit nondeterministic behavior due to Perl's hash randomization. See [NONDETERMINISM.md](NONDETERMINISM.md) for details on affected tags and workarounds.

## Architecture

### Main Composite Table

The central composite table is defined in `lib/Image/ExifTool.pm`:

```perl
%Image::ExifTool::Composite = (
    GROUPS => { 0 => 'Composite', 1 => 'Composite' },
    TABLE_NAME => 'Image::ExifTool::Composite',
    SHORT_NAME => 'Composite',
    VARS => { NO_ID => 1 }, # want empty tagID's for Composite tags
    WRITE_PROC => \&DummyWriteProc,
);
```

### Module-Specific Composite Tables

Each format module can define its own composite tags:

```perl
%Image::ExifTool::Exif::Composite = (
    GROUPS => { 2 => 'Image' },
    ImageSize => { ... },
    # more tags...
);

# Register with main composite table
Image::ExifTool::AddCompositeTags('Image::ExifTool::Exif');
```

## Tag Definition Structure

### Key Components

Each composite tag definition contains:

- **Require**: Hash of tags that must exist for composite to be calculated
- **Desire**: Hash of optional tags that enhance the calculation
- **Inhibit**: Hash of tags that prevent calculation if present
- **ValueConv**: Perl code to calculate the composite value
- **PrintConv**: Optional formatting for display

### Example Definition

```perl
ImageSize => {
    Require => {
        0 => 'ImageWidth',
        1 => 'ImageHeight',
    },
    Desire => {
        2 => 'ExifImageWidth',
        3 => 'ExifImageHeight',
    },
    ValueConv => q{
        return "$val[2] $val[3]" if $val[2] and $val[3];
        return "$val[0] $val[1]" if IsFloat($val[0]) and IsFloat($val[1]);
        return undef;
    },
    PrintConv => '$val =~ tr/ /x/; $val',
}
```

## Processing Flow

### 1. Registration (AddCompositeTags)

The `AddCompositeTags` function (ExifTool.pm:5662):

- Adds module-specific composite tags to the main Composite table
- Handles naming conflicts by appending module prefix
- Sets up tag metadata (groups, override behavior)
- Converts scalar Require/Desire/Inhibit to hash format

### 2. Building (BuildCompositeTags)

The `BuildCompositeTags` function (ExifTool.pm:3904) implements a sophisticated multi-pass algorithm:

#### Algorithm Overview

1. **Initialization**: Gets composite table, sets `BuildingComposite` flag, creates tag list
2. **Multi-pass loop**: Continues until all possible tags are built or circular dependency detected
3. **Dependency resolution**: Defers tags requiring unbuilt composites
4. **SubDoc support**: Handles multi-document files with special caching

#### Detailed Processing Steps

**Pass Setup**:

```perl
for (;;) {  # Multi-pass loop
    my (%notBuilt, @deferredTags);
    # Track which composite tags haven't been built yet
    foreach (@tagList) {
        $notBuilt{$$compTable{$_}{Name}} = 1 unless $specialTags{$_};
    }
```

**Tag Processing Loop**:
For each composite tag:

1. **Dependency Collection**:

   - Iterates through Require, Desire, and Inhibit indices
   - Resolves group-prefixed tag names (e.g., `GPS:GPSLatitude`)
   - Handles alternate file references (e.g., `File1:TagName`)
   - Defers tags requiring unbuilt composites

2. **Group Resolution**:

   ```perl
   if ($reqTag =~ /^(.*):(.+)/) {
       my ($reqGroup, $name) = ($1, $2);
       # Build list of matching tag keys
       # Use GroupMatches() to filter by group
   }
   ```

3. **SubDoc Processing**:

   - For multi-document files ($$self{DOC_COUNT} > 0)
   - Caches tag lookups for performance: `$cache{$reqTag} = []`
   - Processes each document individually: `$docNum = 0..DOC_COUNT`
   - Maps tags to document numbers using G3 group

4. **Dependency Validation**:

   ```perl
   if (defined $$rawValue{$reqTag}) {
       if ($$inhibit{$index}) {
           $found = 0; last;  # Inhibit tag exists, abort
       } else {
           $found = 1;        # Required/Desired tag found
       }
   } elsif ($$require{$index}) {
       $found = 0; last;      # Required tag missing, abort
   }
   ```

5. **Tag Creation**:
   - If all dependencies satisfied: `$self->FoundTag($tagInfo, \%tagKey)`
   - Updates `$$self{COMP_KEYS}` for tracking dependencies
   - Removes from `%notBuilt` tracking

**Deferred Processing**:

- Tags requiring unbuilt composites added to `@deferredTags`
- Next pass processes only deferred tags
- Continues until no progress made

**Circular Dependency Detection**:

```perl
if (@deferredTags == @tagList) {
    if ($allBuilt) {
        warn "Circular dependency in Composite tags\n";
        last;
    }
    $allBuilt = 1;  # Final pass, ignore Inhibit composites
}
```

#### Special Features

**SubDoc Caching**:

- Builds `$cache{$tagName}` arrays indexed by document number
- Optimizes processing when many sub-documents exist
- Uses `$$self{TAG_EXTRA}{$_}{G3}` for document mapping

**Alternate File Support**:

- Detects `File1:`, `File2:` etc. prefixes
- Retrieves tags from `$$self{ALT_EXIFTOOL}{$fileN}`
- Copies metadata with `CopyAltInfo()`

**Dependency Tracking**:

- `$$self{COMP_KEYS}` maps tag keys to composite dependencies
- Enables updating composites when source tags change
- Format: `$compKeys{tagKey} = [ [\%tagKey, $index], ... ]`

#### Performance Optimizations

- Sorts tag list for consistent processing order
- Caches group lookups in SubDoc mode
- Short-circuits when required tags don't exist
- Batches deferred tags for fewer passes

### 3. Dependency Resolution

The builder uses sophisticated dependency tracking:

- Tags requiring other composites are deferred
- Multiple passes ensure all possible tags are built
- Circular dependencies are detected and prevented
- Alt file references (File1:TagName) are supported

## Common Patterns

### 1. Fallback Values

```perl
Aperture => {
    Require => { 0 => 'FNumber' },
    Desire => { 1 => 'ApertureValue' },
    ValueConv => '$val[0] || $val[1]',
}
```

### 2. Unit Conversion

```perl
FocalLength35efl => {
    Require => {
        0 => 'FocalLength',
        1 => 'ScaleFactor35efl',
    },
    ValueConv => '$val[0] * $val[1]',
}
```

### 3. Complex Calculations

```perl
DOF => {
    Require => {
        0 => 'FocalLength',
        1 => 'Aperture',
        2 => 'CircleOfConfusion',
    },
    Desire => {
        3 => 'FocusDistance',
        4 => 'SubjectDistance',
    },
    ValueConv => q{
        # Complex depth of field calculation
        my $d = $val[3] || $val[4] || return undef;
        # ... calculation logic ...
    },
}
```

### 4. Cross-Format Synthesis

```perl
GPSDateTime => {
    Require => {
        0 => 'GPS:GPSDateStamp',
        1 => 'GPS:GPSTimeStamp',
    },
    ValueConv => '"$val[0] $val[1]Z"',
}
```

## Special Features

### IsComposite Flag

All composite tags have `$$tagInfo{IsComposite} = 1` for identification.

### Module Tracking

Writable composite tags store their source module in `$$tagInfo{Module}`.

### Override Behavior

Tags can specify `Override => 1` to replace existing composites with the same name.

### Groups Inheritance

Composite tags inherit default groups from their module, with family 0 and 1 always set to 'Composite'.

## Writing Composite Tags

Most composite tags are read-only, deriving from other tags. A few are writable:

- Writing splits the value and updates source tags
- The WRITE_PROC handles this translation
- Example: `Flash` tag facilitates EXIF<->XMP conversion

## User-Defined Composites

Users can create custom composite tags via the configuration file:

```perl
%Image::ExifTool::UserDefined::Composite = (
    MyCustomTag => {
        Require => { 0 => 'Make' },
        ValueConv => '"Camera: $val[0]"',
    },
);
```

## Group Name Conflicts and Precedence

### Tag Resolution Order

When requesting a tag like "GPSLatitude", ExifTool follows this precedence:

1. **Without group prefix**: Returns first matching tag in priority order:

   - Higher numbered duplicates have priority (e.g., "GPSLatitude (1)" over "GPSLatitude")
   - Most recently extracted tag wins if same priority
   - Composite tags are built after extraction, so appear last

2. **With group prefix** (e.g., "GPS:GPSLatitude"):
   - `GroupMatches()` filters tags by group (ExifTool.pm:5125)
   - Group families checked: 0 (general), 1 (specific), 2 (category)
   - Returns first match from filtered list

### Composite Tag Groups

Composite tags always belong to:

- **Family 0**: 'Composite'
- **Family 1**: 'Composite'
- **Family 2**: Inherited from module (e.g., 'Location' for GPS composites)

The `-G` flag shows family 1 groups, so composite tags appear as "Composite:TagName".

### Priority and Avoid Flags

Some composite tags use special flags:

- `Avoid => 1`: Tag not returned unless specifically requested by group
- `Priority => 1`: Overrides Avoid's default priority of 0
- Example: GPS composite tags set both to provide formatted output by default

## GPS Coordinate Conversions

### Raw GPS Format

EXIF GPS coordinates are stored as three rational64u values:

```
[[degrees, 1], [minutes, 1], [seconds, 100]]
```

### Composite Conversion Implementation

From GPS.pm:353, the GPS composite tags convert to decimal degrees:

```perl
GPSLatitude => {
    Require => {
        0 => 'GPS:GPSLatitude',      # [[d,1],[m,1],[s,100]]
        1 => 'GPS:GPSLatitudeRef',   # 'N' or 'S'
    },
    ValueConv => '$val[1] =~ /^S/i ? -$val[0] : $val[0]',
    PrintConv => 'Image::ExifTool::GPS::ToDMS($self, $val, 1, "N")',
}
```

The `ToDegrees()` function (GPS.pm:582) performs the conversion:

```perl
my $deg = $d + (($m || 0) + ($s || 0)/60) / 60;
```

### Hemisphere Handling

- Latitude: 'S' (South) makes value negative
- Longitude: 'W' (West) makes value negative
- PrintConv formats back to DMS with appropriate suffix

### Other GPS Composites

- **GPSDateTime**: Combines GPSDateStamp + GPSTimeStamp
- **GPSAltitude**: Merges altitude value with Above/Below sea level ref
- **GPSPosition**: Not standard, but could combine lat/lon
- **GPSDestLatitude/Longitude**: For destination coordinates

## Composite Tag Group Hierarchy

### Module Registration

When a module adds composites via `AddCompositeTags()`:

1. **Default groups assigned**:

   ```perl
   GROUPS => { 0 => 'Composite', 1 => 'Composite', 2 => 'Other' }
   ```

2. **Module groups inherited**:

   ```perl
   %Image::ExifTool::GPS::Composite = (
       GROUPS => { 2 => 'Location' },  # Overrides default group 2
   ```

3. **Tag-specific groups**:

   ```perl
   GPSDateTime => {
       Groups => { 2 => 'Time' },  # Override for this tag only
   ```

### Group Hierarchy

Groups are resolved in this order:

1. Tag-specific Groups hash
2. Module Composite table GROUPS
3. Main Composite table defaults

### Dynamic Group Assignment

The `GetGroup()` function (ExifTool.pm:3738) handles group resolution:

- Groups 0-2 are guaranteed to be defined after processing
- Dynamic groups (G0, G1, etc.) can override at extraction time
- Composite tags cannot "masquerade" as other groups - they're always in family 0/1 'Composite'

### Accessing Composite Tags by Group

Examples:

- `Composite:GPSLatitude` - Explicitly request composite version
- `GPS:GPSLatitude` - Gets raw GPS IFD value (3 rationals)
- `GPSLatitude` - Gets composite (decimal) due to Priority flag

## Performance Considerations

- Tags are built on-demand when accessed
- Dependency checking adds overhead for complex hierarchies
- Caching optimizes repeated access in SubDoc scenarios
- Circular dependency detection prevents infinite loops

## See Also

- `lib/Image/ExifTool/README` - Tag table structure documentation
- `html/TagNames/Composite.html` - Complete list of standard composite tags
- `config.html` - User-defined composite tag examples
