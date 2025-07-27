# Tag Filtering Architecture

This document describes ExifTool's tag filtering architecture, based on examination of the ExifTool source code.

## Summary

**ExifTool uses a two-phase filtering architecture**: 
1. **Pre-extraction filtering** via `REQ_TAG_LOOKUP` and `IgnoreTags=all` option
2. **Post-extraction filtering** for specific tag lists via GetInfo()

The critical insight: **ExifTool does NOT filter during tag extraction itself**. Instead, it uses the `IgnoreTags={all => 1}` option combined with `REQ_TAG_LOOKUP` to create an allowlist-based filtering system that operates at the FoundTag level.

## Architecture Details

### 1. Command Line Parsing (exiftool script)

**Location**: `/exiftool` lines 1432-1436

```perl
} else {
    my $lst = s/^-// ? \@exclude : \@tags;
    Warn(qq(Invalid TAG name: "$_"\n)) unless /^([-_0-9A-Z*]+:)*([-_0-9A-Z*?]+)#?$/i;
    push @$lst, $_; # (push everything for backward compatibility)
}
```

Tags specified on command line (e.g., `-Orientation`, `-EXIF:all`, `-GPS*`) are parsed and added to either:
- `@tags` array for inclusion requests
- `@exclude` array for exclusion requests (prefixed with `-`)

### 2. Tag List Processing

**Location**: `/exiftool` lines 2201, 2326, 2344

Three different scenarios for setting `@foundTags`:

```perl
# For -if conditions: extract everything but mention specific tags
@foundTags = ('*', @tags) if @tags;  # line 2201

# Standard extraction with tag list
@foundTags = @tags;  # lines 2326, 2344
```

### 3. REQ_TAG_LOOKUP Setup

**Location**: `/lib/Image/ExifTool.pm` lines 5003, 5069, 5076-5080

```perl
# Initialize lookup hash for requested tags
$$self{REQ_TAG_LOOKUP} = { } unless $$self{ReqTagAlreadySet};

# Add RequestTags option to lookup
if ($$options{RequestTags}) {
    $$self{REQ_TAG_LOOKUP}{$_} = 1 foreach @{$$options{RequestTags}};
}

# Parse command-line tags into lookup
foreach (@{$$self{REQUESTED_TAGS}}) {
    /^(.*:)?([-\w?*]*)#?$/ or next;
    $$self{REQ_TAG_LOOKUP}{lc($2)} = 1 if $2;      # tag name
    next unless $1;
    $$self{REQ_TAG_LOOKUP}{lc($_).':'} = 1 foreach split /:/, $1;  # group names
}
```

**Key Pattern**: Tag names are stored in lowercase, group names are stored with trailing colon.

### 4. The Filtering Mechanism

**Location**: `/lib/Image/ExifTool.pm` lines 9390-9397 (FoundTag function)

```perl
# ignore specified tags (AFTER doing RawConv if necessary!)
if ($$options{IgnoreTags}) {
    if ($$options{IgnoreTags}{all}) {
        return undef unless $$self{REQ_TAG_LOOKUP}{lc $tag};
    } else {
        return undef if $$options{IgnoreTags}{lc $tag};
    }
}
```

**Critical Architecture Decision**: When `IgnoreTags={all => 1}` is set, **every single tag** is rejected unless it exists in `REQ_TAG_LOOKUP`. This creates an allowlist-based filtering system.

### 5. Performance Optimization Pattern

**Location**: Documentation suggests using:

```bash
exiftool -API IgnoreTags=all -MIMEType image.jpg
```

This pattern:
1. Sets `IgnoreTags={all => 1}` to reject all tags by default
2. The `-MIMEType` argument adds `mimetype` to `REQ_TAG_LOOKUP`
3. FoundTag only allows the MIMEType tag through
4. All other metadata parsing still occurs, but tags are discarded at FoundTag level

## Implementation Strategy for exif-oxide

### Option 1: Inline Filtering (Current)
- Add filtering checks at every tag creation point
- **Problems**: Fragmented, unmaintainable, doesn't follow ExifTool patterns

### Option 2: FoundTag-level Filtering (Recommended)
- Create a central `FoundTag` equivalent function that all tag creation goes through
- Implement allowlist-based filtering at this chokepoint
- **Benefits**: Single filtering location, matches ExifTool architecture exactly

### Recommended Architecture

```rust
pub struct TagRegistry {
    req_tag_lookup: HashMap<String, bool>,
    ignore_all_tags: bool,
}

impl TagRegistry {
    fn found_tag(&self, tag_name: &str, value: TagValue) -> Option<TagEntry> {
        // Apply filtering logic here - single point of control
        if self.ignore_all_tags && !self.req_tag_lookup.contains_key(&tag_name.to_lowercase()) {
            return None;
        }
        
        Some(TagEntry::new(tag_name, value))
    }
}
```

## Key Files Referenced

- `/exiftool` lines 1432-1436: CLI tag parsing
- `/exiftool` lines 2201, 2326, 2344: @foundTags setup  
- `/lib/Image/ExifTool.pm` lines 5003-5080: REQ_TAG_LOOKUP initialization
- `/lib/Image/ExifTool.pm` lines 9390-9397: FoundTag filtering logic

## Performance Notes

ExifTool's approach means:
- All metadata parsing still occurs (no performance gain from skipping parsing)
- Performance improvement comes from reduced memory usage and output processing
- The filtering happens AFTER RawConv but BEFORE ValueConv/PrintConv conversions

This explains why simple requests like `-MIMEType` are fast - not because less parsing occurs, but because fewer tags are kept in memory and processed through value conversions.